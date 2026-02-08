# Group Chat (P2P-first) - Spec + Task Plan (qubit.chat)

Datum: 2026-02-07
Updated: 2026-02-08
Scope: Android app + User Service (Django)
Princip: backend je metadata/signalizacija only; payload poruka nikad ne ide kroz backend.

## 1) Ciljevi (sto isporucujemo)

- Poruke idu direktno uredaj <-> uredaj (Tailnet/TCP).
- Backend nikad ne vidi sadrzaj poruka (E2EE envelope).
- Podrzani su 1:1 i grupni chatovi (max 24 clana).
- Grupe imaju role: Owner, Admin, Member (P1).
- UI ima:
  - Create group (wizard)
  - Convert 1:1 -> group
  - Group info (members + add member)
  - Prikaz "Delivered X/Y" za outgoing group poruke

## 2) Non-goals (za sada)

- Nema server-hosted history (server ne sprema poruke).
- Nema central group key-a (nema "jednog kljuca grupe" kojeg dijeli server).
- Nema multi-device per user (trenutno qid == user == device).

## 3) High-level arhitektura

  Android/iOS --HTTPS/JWT--> Backend (Django)
     |                         |
     +---- P2P TCP (Tailnet) --+
            (E2EE payload)

Backend (Django)
- auth (JWT)
- direct/group signalizacija (send-intent, ready/status, ack fallback)
- group management (clanstvo, role, naziv grupe)
- key directory (public keys clanova)
- redis cache (opcionalno)

Klijenti (Android/iOS)
- generiraju i cuvaju identity keypair
- seal/open E2EE envelope
- salju payload P2P
- backend koriste za meta-podatke i discovery

## 4) Kriptografski model (E2EE) - kratko

- Identitet: qid == user == device
- Algoritmi:
  - X25519 (key agreement)
  - HKDF-SHA256 (derivacija)
  - ChaCha20-Poly1305 (AEAD)
- Envelope:
  - msgKey je per-message (random)
  - msgKey se wrapa za svakog primatelja (ephemeral X25519)

## 5) Grupni chat - pravila

- groupId: ID grupe
- max: 24 clana
- role:
  - OWNER (tocno 1)
  - ADMIN (0..N)
  - MEMBER
- Owner:
  - add/remove member
  - promote/demote admin
  - mora transfer-owner prije leave

Kontakti nisu preduvjet za grupu: grupa ima svoj "key directory" (pubkeys clanova).

### 5.1) P0 routing model (Hub) - razlog i tradeoff

Problem u praksi: clanovi grupe cesto nisu medusobno autorizirani kontakti (A autoriziran s Ownerom, ali ne i s B).
Ako forsiramo "mesh" (svaki clan salje svakome), slanje cesto padne.

P0 odluka (radi bez backend managementa):
- Owner je "hub".
- Non-owner salje poruku samo Owneru.
- Owner po prijemu poruke radi fan-out ostalim clanovima.

Ogranicenja:
- Owner mora biti online da bi se poruke clanova isporucile ostalima (P1 rjesava sa server-snapshot + pouzdana signalizacija).

### 5.2) Role i permisije (OWNER/SUPER_ADMIN/ADMIN/MEMBER)

Napomena: u P0 imamo owner-only kontrole (lokalno + P2P invite). `ADMIN`/`SUPER_ADMIN` su planirani (P1.5+) i uvode se tek kad imamo kanonsko clanstvo/role (idealno preko backend management API; P2P-control ostaje best-effort fallback).

Definicije
- OWNER: tocno 1
- SUPER_ADMIN: 0..N (vise super-admina po grupi)
- ADMIN: 0..N
- MEMBER: ostali

Pravila (predlozena, pragmatic)
- Create:
  - creator -> OWNER
  - opcionalno: creator automatski i SUPER_ADMIN (radi jednostavnije logike "co-owner")
  - svi ostali -> MEMBER
- Add member:
  - P0: samo OWNER (lokalno)
  - P1.5+: OWNER i SUPER_ADMIN i ADMIN
- Remove member:
  - OWNER/SUPER_ADMIN moze remove bilo koga osim OWNER-a (owner leave rjesavamo automatskim transferom; vidi dolje)
  - ADMIN moze remove samo MEMBER koje je ADMIN dodao (enforcement preko `added_by_qid`)
- Promote/Demote admin:
  - OWNER/SUPER_ADMIN moze promote MEMBER->ADMIN i demote ADMIN->MEMBER
  - ADMIN moze promote MEMBER->ADMIN (ali ne moze demote druge ADMINE, niti moze mijenjati SUPER_ADMIN)
- Promote/Demote super-admin:
  - SUPER_ADMIN smije dodijeliti SUPER_ADMIN rolu (promote MEMBER/ADMIN -> SUPER_ADMIN)
  - samo OWNER smije ukloniti SUPER_ADMIN rolu (demote SUPER_ADMIN -> ADMIN/MEMBER)
  - nitko ne moze demote OWNER-a (OWNER je uvijek tocno 1)
- Transfer owner:
  - P1.5 cilj: user ne mora manualno birati transfer kad OWNER zeli izaci
  - Pravilo: OWNER ne moze leave ako nema nijednog SUPER_ADMIN-a osim sebe
  - Kad OWNER stisne "Leave group":
    - backend automatski bira novog OWNER-a iz SUPER_ADMIN seta (deterministic: npr. lexicographically najmanji QID u super_admins bez starog ownera)
    - stari owner napusta grupu (postaje non-member)
    - novi owner ostaje SUPER_ADMIN (implicitno)
  - Manual transfer-owner UI ostaje opcionalan (npr. "Make owner") ali nije obavezan
- Leave:
  - MEMBER/ADMIN mogu leave bilo kad
  - SUPER_ADMIN moze leave bilo kad (dok god ostaje barem 1 SUPER_ADMIN ili OWNER, ovisno o politici)
  - OWNER leave dozvoljen samo uz auto-transfer na SUPER_ADMIN (gore)
- Rename group:
  - OWNER/SUPER_ADMIN moze rename
  - ADMIN moze rename (opcionalno; predlazem NE u startu)
- Delete group:
  - OWNER/SUPER_ADMIN moze delete group
  - ADMIN nema delete

Napomena o "ADMIN remove only added members"
- Ovo uvodi dodatni metadata po clanu: `added_by_qid` (tko ga je ubacio u grupu).
- To pravilo smanjuje mogucnost zloupotrebe admina, ali komplicira UX i backend.
- Alternativa (jednostavnije): ADMIN moze remove bilo kojeg MEMBER-a (ali ne ADMIN/SUPER_ADMIN/OWNER).

## 6) Kriticna odluka: identitet threada (chatId)

Da bi svi clanovi vidjeli poruke u istom threadu, moramo imati stabilan chatId koji je isti na svim uredajima.

Definicije
- peerQid:
  - 1:1: stvarni qid peer-a
  - group chat u UI: pseudo peer "g:<groupId>"
- chatId:
  - 1:1: canonical DRID (postojeci model)
  - grupa: "g:<groupId>"

Kriticno za MVP
- Envelope meta za group poruke mora nositi chatId = "g:<groupId>".
- Receiver mora spremati inbound poruke pod meta.chatId (ne samo pod "fromQid"), da se group thread ne raspadne u 1:1 threadove.

## 6.1) P0: Invite (signalizacija) i lokalni metadata

Za P0 ne radimo backend "create group". Umjesto toga:
- creator/owner generira `groupId`
- lokalno sprema:
  - `groupName`
  - `members` (puni popis, ukljucuje ownera)
  - `owner_qid`
- svakom clanu salje CONTROL frame `group_invite` (P2P), koji receiver koristi da:
  - lokalno persist-a group metadata
  - doda group dialog u Chat List
  - prikaze notifikaciju ("Added to group ...") koja otvara `app://qubit/chat/g:<groupId>`

P0 owner operacije (P2P control, best-effort)
- `group_members_update`: owner broadcast-a novi `members_csv` (+ opcionalno `owner_qid`) prema preostalim clanovima
- `group_removed`: owner salje samo uklonjenom clanu (client brise group iz Chat List + brise prefs)
- `group_owner_changed`: owner broadcast-a `new_owner_qid` (client update-a owner u prefs; relay routing se odmah prebaci)
- `group_member_left`: member salje owneru; owner update-a members i broadcast-a `group_members_update`
- `group_deleted`: owner broadcast-a svima; client brise group iz Chat List + brise prefs

Napomena: ovo je UI/klijent-level "istina" dok ne uvedemo backend management API (P1). Ako je uredaj offline, promjene su best-effort (stizu kad P2P signaling uspije).

Payload `group_invite` (CONTROL body):
- `group_id`
- `group_name`
- `members_csv`
- `owner_qid` (fallback: ako nije poslan, owner = inviter/peerQid)

## 6.2) P0: Relay (owner fan-out)

Kad owner relay-a poruku clanovima, koristi TEXT body flagove:
- `author_qid`: originalni sender (member)
- `relayed_by_owner`: `"1"` (da izbjegnemo petlju)

## 7) UI flowovi (tocan UX)

Flow A: Create group (iz Chat Lista)
1. Chat List: akcija "New group".
2. Wizard Step 1: odabir clanova iz kontakt liste (multi-select, search, CTA disabled dok <2).
3. Wizard Step 2: unos naziva grupe (required) + "Create".
4. Nakon create: otvori Chat Detail za peerQid = "g:<groupId>".

Flow B: Convert 1:1 -> group (iz Chat Detail 1:1)
1. Chat Detail (1:1): top bar "Info" -> "Create group".
2. Wizard Step 1: trenutni peer je preselected + locked; korisnik dodaje jos clanova.
3. Wizard Step 2: naziv + "Create".
4. Navigacija u novi group chat ("g:<groupId>").
5. (Optional P2) U stari 1:1 thread dodati system link "Group created: <name>".

Flow C: Add member (iz Group Info)
1. Chat Detail (group): top bar "Info" -> "Group info".
2. Group info: members list + CTA "Add member".
3. P0: Add/Remove su lokalne promjene i owner-only (UI enforce).
4. P1.5+: Add/Remove/roles idu preko backend managementa + system poruke.

## 8) Backend API (Redis-only group directory + signaling)

Napomena: u app-u vec postoji group-send signalizacija endpoint set:
- POST /api/group/send-intent/
- GET  /api/group/status
- GET  /api/group/members
- POST /api/group/accept/

Status 2026-02-08:
- Group endpoints su Redis-only (nema vise upisa u chat_control OutboxMessage/OutboxRecipient iz /api/group/*).
- Legacy TCP direct flow i dalje koristi Postgres tablice (DirectMessageJob):
  - POST /api/direct/ready/
  - GET  /api/direct/status/

### 8.1) Redis kanon (Android contract)

Group directory (canonical membership)
- group:{chatId}:members (SET), vrijednosti: QID-evi (UPPERCASE), TTL 7d inactivity
- group:{chatId}:meta (HASH), TTL 7d inactivity
  - owner_qid
  - version (trenutno 1)
  - updated_at_ms (postavi se samo na kreiranju ili membership promjeni)
  - last_activity_ms (refresha se na svaki group/send-intent)
- group:{chatId}:owner (STRING), TTL 30d inactivity (anti-squatting "tombstone")
Role status:
- P0/P1: owner_qid postoji; admin/super-admin set nije implementiran (P1.5).

P1.5 role keys (ako ostanemo na Redis-u za roles prije Postgresa)
- group:{chatId}:superadmins (SET), TTL 7d inactivity
- group:{chatId}:admins (SET), TTL 7d inactivity
- group:{chatId}:members_added_by (HASH), TTL 7d inactivity
  - field: memberQid -> value: added_by_qid
  - koristi se za enforce "admin moze remove samo svoje"

Anti-squatting / re-init (kad uvodimo SUPER_ADMIN)
- Trenutno tombstone je `group:{chatId}:owner` (STRING, TTL 30d).
- Kad imamo SUPER_ADMIN, preporuka je prosiriti "controller tombstone" na vise qid-eva:
  - group:{chatId}:controllers (SET, TTL 30d inactivity): OWNER + svi SUPER_ADMIN
  - re-init dozvoljen ako caller je u controllers
  - (alternativa: zadrzati samo owner i u praksi uvijek imati 1 owner; ali onda owner leaving je opet problem)

Init + max 24
- Init se radi samo ako group:{chatId}:meta ne postoji.
- Init members dolaze iz requesta, ali server filtrira na aktivne user-e i uvijek doda caller-a.
- Enforce max 24 (ako init lista > 24 -> 400).

Expiry + re-init nakon 7d
- members/meta imaju TTL 7d inactivity.
- Ako grupa istekne (meta/members nestanu), re-init je dozvoljen samo ako:
  - group:{chatId}:owner ne postoji (nova grupa), ili
  - caller == group:{chatId}:owner (owner moze obnoviti)
- Inace 403.

Atomicnost
- Init-or-get je Lua (jedan atomic poziv).
- directjob create-or-reuse je Lua.
- WebRTC poll consume (LRANGE+LTRIM) je Lua.

Accepted gate (anti-spam/consent)
- group:{chatId}:accepted (SET), TTL 7d inactivity
  - refresh na group/send-intent i group/accept (ne na group/members)
  - init: owner (caller) se automatski doda u accepted
- Fanout (directjob + chat.incoming wake) ide samo prema accepted clanovima.
- POST /api/group/accept/:
  - guard: caller mora biti u group:{chatId}:members, inace 403
  - efekt: SADD accepted + TTL refresh

### 8.2) Redis kljucevi (signaling)

Per-recipient authz window (directjob), TTL clamp 15..180s
- directjob:{minQid}:{maxQid}:{messageId} (HASH)

WebRTC signaling queue (per-peer LIST), TTL 300s, max 500
- webrtc:{toQid}:{messageId}:{fromQid} (LIST)

ACK fallback, TTL 7 dana
- ack:{toQid}:{fromQid}:{messageId} (HASH)

Pravila
- enforce max 24 clana
- enforce tocno 1 owner po grupi

## 9) Task backlog

### P0 (MVP: UI + routing + local metadata, bez role managementa)

- [x] UI entry points
- [x] Chat List: top bar/overflow = "New group" (wizard), FAB ide na Contacts (New chat)
- [x] Chat Detail: Info button + menu
- [x] 1:1: "Create group" (convert flow)
- [x] group: "Group info"

- [x] GroupCreateWizard (pravi screen)
- [x] Step 1: multi-select iz Contacts (authorized), search, "Selected (n)"
- [x] Step 2: group name input + Create

- [x] Local group metadata (P0 local-only)
- [x] Lokalno spremi groupName + members
- [x] Persist owner (`owner_qid`) za P0 hub routing
- [x] Chat List row za g:* koristi groupName
- [x] Chat Detail header za g:* prikazuje groupName + "n members"

- [x] Thread identity fix (kriticno)
- [x] Envelope meta za group send: chatId = "g:<groupId>"
- [x] Receiver persistence: inbound poruke idu u thread chatId (meta.chatId)

- [x] Invite: clanovi dobiju notifikaciju + group se pojavi u Chat List
- [x] Hub routing: member -> owner, owner fanout -> ostali
- [x] UI: prikaz "Delivered X/Y" po group poruci (per-message label uz status ikonu)
- [x] UI: prikaz autora u group bubble (ime iz kontakata; fallback na author_name iz relaya/invite)
- [x] Router: nakon create, back iz wizard-a vraca u Chat List (ne u seed 1:1 thread)

### P1 (backend: Redis-only directory + signaling)

- [x] Backend: Redis-only group directory (members/meta) + init-or-get Lua + max 24
- [x] Backend: anti-squatting tombstone (group:{chatId}:owner) + re-init pravila
- [x] Backend: WebRTC-only status contract (WAITING_READY|EXPIRED|INVITED) + membersCount/acceptedCount
- [x] Backend: accepted gate (group:{chatId}:accepted) + POST /api/group/accept + fanout samo prema accepted
- [x] Backend: observability (metrics/logovi) za init/re-init/403, queue overflow, ttl expiry (+ requestId/remoteAddr u JSON logovima)
- [x] Backend: X-Request-Id middleware (generira requestId ako ga nema + echo u response header)
- [x] Backend: split Redis (ephemeral vs directory) da group:* ne bude pod allkeys-lru evictionom
- [x] Client: eksplicitni accept flow (na open group chat pozvati /api/group/accept)
- [x] Client: TOFU warning na promjenu pubkey-a za clanove grupe (pin u chat_key_pin + banner kad je status=suspect)

### P1 (backend: management API + key directory) (nije implementirano)

- [ ] Backend (Postgres): model `groups` (+ migracije + indexi)
- [ ] Backend (Postgres): `groups` polja: `id`, `name`, `created_at`, `created_by_qid`, `owner_qid`, `revision`, `max_members` (default 24), `name_updated_at`, `name_updated_by_qid`, `deleted_at`, `deleted_by_qid`
- [ ] Backend (Postgres): model `group_members` (+ migracije + indexi; UNIQUE(group_id, member_qid))
- [ ] Backend (Postgres): `group_members` polja: `group_id`, `member_qid`, `role`, `added_by_qid`, `invited_at`, `accepted_at`, `joined_at`, `left_at`, `removed_at`, `removed_by_qid`

Backend: management API (kanonski)
- [ ] Backend: create group
- [ ] Backend: invite/add member (postavlja `added_by_qid` + `invited_at`)
- [ ] Backend: accept invite (postavlja `accepted_at` + `joined_at`)
- [ ] Backend: remove member (postavlja `removed_at` + `removed_by_qid`)
- [ ] Backend: leave group (postavlja `left_at`)
- [ ] Backend: rename group (postavlja `name_updated_at/by`)
- [ ] Backend: delete group (soft delete: `deleted_at/by`)
- [ ] Backend: owner leave auto-transfer (odabir novog ownera iz SUPER_ADMIN seta)
- [ ] Backend: authz enforcement (OWNER/SUPER_ADMIN/ADMIN) + rate-limit za management endpointe
- [ ] Backend: snapshot endpoint (jedan kanonski odgovor): members + roles + `added_by_qid` + `accepted_at` + `revision` + `deleted_at/by`
- [ ] Backend: key directory / snapshot: vraca pubkeys clanova (da clanovi ne moraju biti kontakti)
- [ ] Backend: event/wake signal (MQTT) za `group.updated` i `group.deleted` (da svi uredaji osvjeze snapshot)
- [ ] Client: migrate sa P0 "local-only membership" na backend kanonsko clanstvo (persist `revision`, reconcile local prefs)
- [ ] Client: na open group: POST accept + GET snapshot (update local prefs + system poruke iz diff-a)
- [ ] Client: ako snapshot kaze `deleted_at!=null`: makni group iz Chat List + obrisi prefs
- [ ] Client: Group Info akcije (add/remove/transfer/delete/rename/roles) prebaciti sa P2P-control na backend management API
- [ ] Client: UI enforcement admin pravila (npr. "remove samo svoje" preko `added_by_qid`)

### P1.5 (roles)

- [ ] Backend: role storage (SUPER_ADMIN/ADMIN) + enforcement pravila
- [ ] Backend: `added_by_qid` tracking i enforce "ADMIN remove only added members"
- [ ] Backend: endpoint promote admin
- [ ] Backend: endpoint demote admin
- [ ] Backend: endpoint promote super-admin (OWNER + SUPER_ADMIN)
- [ ] Backend: endpoint demote super-admin (samo OWNER)
- [ ] Backend: endpoint owner leave auto-transfer (select new owner from superadmins)
- [ ] Client: Group Info UI: prikaz role per member (Owner/Super admin/Admin/Member)
- [ ] Client: owner UI: add/remove/promote/demote super-admin
- [ ] Client: super-admin UI: add/remove + promote/demote admin + promote super-admin
- [ ] Client: admin UI: add member + remove samo "svoje"

### P2 (polish)

- [x] System poruke: member added/removed (UI-only, generira se na promjenu membership snapshot-a)
- [x] Owner kontrole u Group Info (owner-only add/remove UI; admin roles P1.5)
- [x] Owner lifecycle (P0/P2, best-effort P2P control):
- [x] Delete group (broadcast `group_deleted` + lokalno brisanje + confirm dialog)
- [x] Remove member propagacija (owner -> removed `group_removed` + ostalima `group_members_update`)
- [x] Transfer ownership (broadcast `group_owner_changed` + `group_members_update`)
- [x] Leave group (member -> owner `group_member_left`, owner rebroadcast `group_members_update`)
- [x] Receiver: obrada control tipova `group_members_update|group_owner_changed|group_deleted|group_removed|group_member_left`
- [x] Convert 1:1 -> group: system link u 1:1 thread
- [x] Concurrency tuning (2) i retry UX za group sends (Delivered X/Y + retrying/failed suffix)
- [x] Compose stability: unique LazyColumn keys u group wizardu (case-normalizacija)

### Future (ideas / optional)

- [ ] Read receipts (seenAt): opt-in per user/group; u grupi owner-aggregated (hub) + UI "Seen by X" (opcionalno umjesto Delivered X/Y)
- [ ] Role-change system poruke: "X promoted to Admin/Super admin", "Owner changed"
- [ ] Rename group (UI + backend) s audit poljima `name_updated_at/by`
- [ ] Better offline reconciliation: periodic snapshot refresh + conflict UI ("membership changed while you were offline")
Napomena: attachments su globalni subsystem (shared 1:1 + group). Plan i taskovi su u `docs/attachments/attachments.md`.

## 10) Acceptance criteria (MVP)

- Create group: user odabere >=2 kontakta, unese naziv, dobije novi chat g:<id> u Chat List.
- Send u group: poruka ide P2P (P0 hub routing: non-owner -> owner, owner -> ostali).
- Receive u group: svi clanovi vide poruku u istom group threadu (ne raspadne se u 1:1 threadove sa senderom).
- Ako clanovi nisu medusobno autorizirani, i dalje rade grupne poruke (preko owner fanout).
- Backend i dalje ne vidi payload poruka.

## 11) Status (repo today)

- E2EE envelope postoji (X25519/HKDF/ChaCha20-Poly1305).
- Postoji P0 group routing (chatId threading + invite + hub fanout).
- Backend group signaling/directory je Redis-only i ima accepted gate + anti-squatting pravila (owner tombstone).
- Najveci UX rizik i dalje: owner kao SPOF za member->others dok god koristimo P0 hub routing.
