# Group Chat - Backend Tasks (Django + Postgres + Redis)

Datum: 2026-02-08
Scope: User service (Django) + Postgres (durable) + Redis (ephemeral/signaling)
Princip:
- Backend ne vidi plaintext poruka (payload ide P2P, E2EE).
- Postgres je durable source-of-truth za *group management* (rijetke promjene).
- Redis je za *runtime/signaling* (ceste, TTL-based, evictable).
- Nema centralne povijesti poruka; novi clan ne dobiva stare poruke (no-backfill).

## Task List

- [x] Postgres modeli + migracije: `Group` i `GroupMember` (indexi/constrainti/semantika aktivnog clana)
- [x] Authz matrica + normalizacija: `qid` uppercase, `groupId` validacija, centralni permission helper
- [x] Management API (JWT): create/add/accept/remove/leave/rename/delete + role endpointi
- [x] Snapshot API: `GET /api/group/snapshot` (sinceRevision/304 ili unchanged flag)
- [x] Realtime wake: MQTT wake (preko `chat_control.services.mqtt.publish_wake`) na aktivne clanove
- [~] Observability + rate limit: dodani JSON logovi + metrics counteri; rate-limit je uveden samo za dio endpointa
- [~] API docs (drf-spectacular): endpointi su u OpenAPI; `@extend_schema` je dodan za dio endpointa (treba dopuniti za role/delete/rename/leave)

## Milestones

### M2: Runtime send-intent koristi Postgres kanon (remove client-supplied members)

Cilj: runtime endpoints (posebno `POST /api/group/send-intent/`) ne smiju ovisiti o `memberQids` iz requesta. Canonical clanstvo/accepted/roles dolaze iz Postgresa, Redis se koristi samo za signaling (directjob/webrtc/ack).

Definition of done:
- `POST /api/group/send-intent/`:
  - ignorira `memberQids` iz requesta (ili ih potpuno ukloni iz schema)
  - canonical members cita iz Postgresa (`GroupMember`, aktivni + joined)
  - enforce max 24
  - enforce accepted gating (ako ostaje): samo accepted clanovi dobivaju wake/directjob
  - actor mora biti aktivni clan (inace 403)
- Testovi:
  - “memberQids spoof” ne moze prosiriti grupu
  - removed/left user dobije 403
  - soft-deleted group -> 410/404 (po odluci), bez side-effecta u Redis-u
- Observability:
  - log + metric kad se detektira mismatch (request memberQids != canonical) ako request polje jos postoji

## Implementirano (2026-02-08)

Durable modeli + migracije (Postgres):
- Modeli: `qubit/user/backend/chat_control/models.py` (`Group`, `GroupMember`)
- Migracije: `qubit/user/backend/chat_control/migrations/0006_groups_postgres.py` (+ auto `0007_*` za postojece promjene u app-u)

Kanonski management endpointi (JWT, Postgres-backed) u `qubit/user/backend/chat_control/views.py` + rute u `qubit/user/backend/chat_control/urls.py`:
- `POST /api/group/create/`
- `POST /api/group/add/`
- `POST /api/group/accept-invite/` (namjerno odvojeno od Redis runtime `POST /api/group/accept/`)
- `POST /api/group/remove/`
- `POST /api/group/leave/`
- `POST /api/group/rename/`
- `POST /api/group/delete/`
- Role:
  - `POST /api/group/role/promote-admin/`
  - `POST /api/group/role/demote-admin/`
  - `POST /api/group/role/promote-superadmin/`
  - `POST /api/group/role/demote-superadmin/`

Snapshot endpoint (kanonski read):
- `GET /api/group/snapshot` i `GET /api/group/snapshot/` (sinceRevision -> `unchanged:true` kada je revision isti)

Wake/signaling:
- Na svaki management write koji mijenja state radi wake: event `group.updated` ili `group.deleted` (preko MQTT topic-a `qubit/{qid}/chat/wake`).

OpenAPI docs:
- `/api/schema/` + `/api/docs/` vec postoje i novi endpointi se pojavljuju u schemi.

## Prijedlozi (što još treba / rizici)

- Prebaciti runtime flow da koristi Postgres kanon:
  - `POST /api/group/send-intent/` trenutno je Redis-kanon i ovisi o `memberQids` iz requesta (client-supplied). To treba promijeniti da clanove cita iz Postgresa (`GroupMember`) i koristi Redis samo za per-message authz window / signaling queue.
- Dodati testove minimalno za authz edge-caseove:
  - ADMIN remove pravilo (`added_by_qid`), OWNER leave transfer, demote-superadmin samo owner, soft delete behaviour, snapshot authz.
- Rate-limit dovrsiti konzistentno:
  - trenutno je uveden za create/add/remove; dodati i za rename/delete/leave/role.
- Snapshot caching/transport:
  - opcija: vratiti HTTP 304 kad je `sinceRevision == revision` (trenutno vraca 200 + `unchanged:true`).
- Validacija `groupId`:
  - trenutno je regex; ako zelis striktno UUID format, promijeniti validaciju i/ili koristiti UUIDField.

## 0) Terminologija

- `qid`: identitet uredaja (user == device).
- `groupId`: UUID/string identifikator grupe.
- `chatId`: kanonski thread id na klijentu: `g:<groupId>`.
- Role:
  - `OWNER` (tocno 1) je u `groups.owner_qid`.
  - `SUPER_ADMIN` (0..N) i `ADMIN` (0..N) i `MEMBER` su u `group_members.role`.
- Pravilo:
  - `SUPER_ADMIN` smije promote-ati SUPER_ADMIN (dodati), ali demote SUPER_ADMIN moze samo OWNER.
  - `ADMIN` moze remove samo clanove koje je on dodao (`added_by_qid`).
  - `SUPER_ADMIN` ima prava skoro kao OWNER (add/remove/rename/delete).
  - OWNER moze izaci iz grupe bez rucnog transfera: auto-transfer na SUPER_ADMIN (deterministic).

## 1) Postgres: durable modeli + migracije

### 1.1 Tablica: `groups`

Zadatak:
- Dodati Django model `Group` + migraciju.
- Dodati indexe i constraint-e (ispod).

Polja:
- `id` (PK): TEXT/UUID, vrijednost = `groupId`
- `name`: TEXT NOT NULL
- `created_at`: TIMESTAMPTZ NOT NULL (default now)
- `created_by_qid`: TEXT NOT NULL
- `owner_qid`: TEXT NOT NULL
- `revision`: BIGINT NOT NULL (default 1)
- `max_members`: INT NOT NULL (default 24)
- `name_updated_at`: TIMESTAMPTZ NULL
- `name_updated_by_qid`: TEXT NULL
- `deleted_at`: TIMESTAMPTZ NULL
- `deleted_by_qid`: TEXT NULL

Indexi/constrainti (preporuka):
- PK(`id`)
- INDEX(`owner_qid`)
- (opcionalno) partial index za “aktivne grupe”: `WHERE deleted_at IS NULL`

Semantika:
- `revision` se bumpa samo na management evente (rename, owner change, membership/roles, delete).
- Ne upisivati `last_activity` u Postgres.

### 1.2 Tablica: `group_members`

Zadatak:
- Dodati Django model `GroupMember` + migraciju.
- Dodati indexe i constraint-e (ispod).

Polja:
- `group_id`: FK -> `groups.id`
- `member_qid`: TEXT NOT NULL
- `role`: ENUM/TEXT NOT NULL (`SUPER_ADMIN|ADMIN|MEMBER`)
- `added_by_qid`: TEXT NOT NULL
- `invited_at`: TIMESTAMPTZ NOT NULL (default now)
- `accepted_at`: TIMESTAMPTZ NULL
- `joined_at`: TIMESTAMPTZ NULL
- `left_at`: TIMESTAMPTZ NULL
- `removed_at`: TIMESTAMPTZ NULL
- `removed_by_qid`: TEXT NULL

Indexi/constrainti (preporuka):
- UNIQUE(`group_id`, `member_qid`)
- INDEX(`group_id`)
- INDEX(`member_qid`)
- (opcionalno) partial index aktivnih: `WHERE joined_at IS NOT NULL AND left_at IS NULL AND removed_at IS NULL`
- CHECK: nije dozvoljeno da `left_at` i `removed_at` budu oba setana (ili definirati prioritet i enforce u kodu).

Semantika:
- Aktivni clan: `joined_at != null AND left_at is null AND removed_at is null`.
- Pozvan ali nije prihvatio: `invited_at != null AND accepted_at is null`.
- No-backfill: clan dobiva poruke samo nakon `joined_at` (klijent enforcement), backend snapshot to nosi.

## 2) Redis: runtime/signaling (ostaje kao danas)

Ovo ostaje “ephemeral”, TTL-based:
- `directjob:*` (authz window, TTL clamp 15..180s)
- `webrtc:*` (signaling queue, TTL 300s, max 500)
- `ack:*` (ACK fallback, TTL configurable)

Ako ostaje split Redis:
- `redis-ephemeral`: directjob/webrtc/ack, eviction OK
- `redis-directory`: (ako i dalje drzimo) group:* TTL state (ali nakon Postgresa nije kanon)

Napomena:
- Nakon sto Postgres postane kanon, Redis `group:*` moze ostati samo kao cache (opcionalno), ali ne kao source-of-truth.

## 3) Management API: endpoints (kanonski)

Napomena:
- Sve endpoints autorizirati JWT-om.
- Svi write endpointi bumpaju `groups.revision` (atomicno).
- Svi endpointi moraju provjeriti `groups.deleted_at IS NULL` (osim ako vracamo “gone”).

### 3.0 API Docs (OpenAPI)

Zadatak:
- Izgenerirati i hostati OpenAPI schema + UI docs za sve group chat endpoint-e.

Stack:
- Koristiti `drf-spectacular` (vec je u `requirements.txt` i `INSTALLED_APPS`).

Endpointi (preporuka):
- `GET /api/schema/` (OpenAPI JSON)
- `GET /api/docs/` (Swagger UI)

Napomena:
- Za svaki endpoint dodati request/response primjere (iz ovog dokumenta) preko `@extend_schema`.

### 3.1 Create group

`POST /api/group/create/`
- Body: `name`, `memberQids[]` (inicijalni invited), opcionalno `maxMembers`
- Efekt:
  - kreira `groups` (owner = caller)
  - kreira `group_members` redove za ownera (joined odmah) + invited (invited_at)
  - `revision = 1`
- Guard:
  - max 24 ukupno (owner + invited)
  - svi qid-evi uppercased i distinct

### 3.2 Add/invite member

`POST /api/group/add/`
- Body: `groupId`, `memberQid`
- Efekt:
  - upis/refresh `group_members` za `memberQid` kao invited (ako nije vec aktivan)
  - `added_by_qid = caller`
  - bump `revision`
- Guard:
  - authz: OWNER/SUPER_ADMIN/ADMIN (ADMIN samo add, uvijek OK)
  - ne preko `max_members`

### 3.3 Accept invite

`POST /api/group/accept-invite/` (Redis-only `POST /api/group/accept/` ostaje za runtime gating; ovaj endpoint je Postgres-kanon)
- Body: `groupId`
- Efekt:
  - provjeri da postoji `group_members` invite za caller
  - set `accepted_at` + `joined_at` (ako nije vec)
  - bump `revision` (opcionalno, ali preporuka da se vidi join event u snapshotu)
- Guard:
  - caller mora biti invited member

### 3.4 Remove member

`POST /api/group/remove/`
- Body: `groupId`, `memberQid`
- Efekt:
  - set `removed_at` + `removed_by_qid`
  - bump `revision`
- Guard:
  - OWNER/SUPER_ADMIN: mogu remove bilo koga osim OWNER-a
  - ADMIN: moze remove samo ako `group_members.added_by_qid == caller` i target je `MEMBER`

### 3.5 Leave group

`POST /api/group/leave/`
- Body: `groupId`
- Efekt:
  - ako caller nije OWNER: set `left_at`
  - ako caller je OWNER:
    - auto-transfer OWNER na SUPER_ADMIN (deterministic izbor)
    - stari owner: set `left_at`
    - bump `revision`
- Guard:
  - OWNER leave dozvoljen samo ako postoji barem 1 SUPER_ADMIN osim ownera
  - (alternativa) auto-promote nekog ADMIN->SUPER_ADMIN prije transfera, ali to treba eksplicitno odluciti

### 3.6 Rename group

`POST /api/group/rename/`
- Body: `groupId`, `name`
- Efekt:
  - update `groups.name`, set `name_updated_at/by`
  - bump `revision`
- Guard:
  - OWNER/SUPER_ADMIN (ADMIN opcionalno NE)

### 3.7 Delete group (soft delete)

`POST /api/group/delete/`
- Body: `groupId`
- Efekt:
  - set `groups.deleted_at/by`
  - bump `revision`
- Guard:
  - OWNER/SUPER_ADMIN

### 3.8 Roles

`POST /api/group/role/promote-admin/`, `.../demote-admin/`
- Guard:
  - OWNER/SUPER_ADMIN

`POST /api/group/role/promote-superadmin/`
- Guard:
  - OWNER ili SUPER_ADMIN

`POST /api/group/role/demote-superadmin/`
- Guard:
  - samo OWNER

Napomena:
- OWNER se ne demota kroz `group_members.role` nego mijenja `groups.owner_qid`.

## 4) Snapshot / sync contract

Zadatak:
- Implementirati kanonski read endpoint koji klijent koristi kao source-of-truth.

`GET /api/group/snapshot?groupId=...&sinceRevision=...`
- Ako `sinceRevision == groups.revision`: vratiti `{unchanged:true, revision:...}` ili HTTP 304.
- Inace vratiti:
  - `group`: id, name, owner_qid, revision, max_members, deleted_at/by, name_updated_*
  - `members[]`: member_qid, role, added_by_qid, invited_at, accepted_at, joined_at, left_at, removed_at/by

Client semantika:
- Ako `deleted_at != null`: client brise group iz Chat List + brise local prefs.
- “No-backfill”: client ne trazi poruke; samo krece primati od `joined_at`.

## 5) Event/wake (realtime)

Zadatak:
- Kad se `revision` promijeni, backend salje MQTT wake event:
  - topic: `group.updated` (payload: groupId, revision)
  - topic: `group.deleted` (payload: groupId, revision)
- Recipienti:
  - svi aktivni clanovi (joined + not removed/left)
  - (opcionalno) invited clanovi (da vide invite UI)

Napomena:
- Ovaj wake ne nosi plaintext, samo signal “idi povuci snapshot”.

## 6) Write policy (da Postgres nije “stalno pisanje”)

- Postgres writes:
  - samo management eventi (create/add/accept/remove/leave/rename/delete/role/owner transfer)
- Redis writes:
  - send-intent, webrtc signaling, ack, rate-limit kljucevi, TTL refresh
- Nikad:
  - per-message writes u Postgres (nema receipts, nema last_activity)

## 7) Observability + rate limiting

Zadatak:
- Strukturirani JSON logovi za svaki management endpoint:
  - `chat.group.mgmt.ok` / `chat.group.mgmt.fail`
  - fields: endpoint, groupId, actorQid, targetQid (ako postoji), role, err
- Metrike counteri (StatsD/Prom):
  - `chat.group.mgmt_create_total`, `..._add_total`, `..._remove_total`, `..._leave_total`, `..._rename_total`, `..._delete_total`
  - `chat.group.mgmt_403_total` (reason)
- Rate-limit:
  - per qid: npr. 10/min create, 30/min add/remove, 60/min accept

## 8) Security hardening (minimum)

- Normalizacija QID (uppercase) i `groupId` validation.
- Authz matrica role->akcija (jedno mjesto u kodu).
- Concurrency: optimistic lock preko `revision` (opcionalno):
  - write endpoint moze primati `ifRevision` i vratiti 409 ako je stale.

## 9) Deployment / migration plan (pragmatic)

Faza A (kompatibilnost):
- Postgres models + snapshot endpoint uvedeni, ali Redis-only flow radi paralelno.

Faza B (prebacivanje kanona):
- Client prvo pocinje koristiti snapshot kao source-of-truth.
- Management write endpointi postaju jedini nacin izmjene membership/roles.
- Redis `group:*` ostaje samo cache ili se gasi.
