# Attachments (Global Subsystem) - Plan + Task List

Datum: 2026-02-08
Scope: Android app + User Service (Django)
Princip: backend ne vidi plaintext; ako postoji centralni storage, sprema se samo ciphertext.

## 1) Ciljevi

- Jedan attachment subsystem za 1:1 i group chat (nema duplog posla).
- Pouzdan upload/download s jasnim retry UX-om i progresom.
- Opciono (Future): store-and-forward preko centralnog object storea za ciphertext blobove.

## 2) Non-goals (za sada)

- Nema server-side transkodiranja ili pregleda sadrzaja.
- Nema “history/backfill” semantike (nema centralne povijesti poruka).

## 3) Modeli (konceptualno)

Attachment (u poruci / DB-u) treba imati stabilni identitet i verificabilnost:
- `attachment_id` (UUID)
- `mime`
- `size_bytes`
- `sha256_ciphertext` (integritet ciphertexta)
- `crypto` metadata (verzija sheme, nonce/iv, eventualno wrapped file key)
- `transport_ref` (P2P ili object-store reference)

Napomena: za E2EE je bitno da se file key nikad ne salje u plaintext; mora biti wrapan unutar envelope-a ili uz attachment meta.

## 4) Transporti

### 4.1) P2P transport (default)

- Sender salje attachment direktno peer-u (1:1) ili radi fanout (group) koristeci isti attachment pipeline.
- Ako receiver nije online ili je veza losa, retry je best-effort i bez “resume” (dok ne uvedemo range/resume).

### 4.2) Store-and-forward (Future)

Ideja: klijent enkriptira file lokalno i upload-a ciphertext u object store (S3/MinIO ili slicno) preko signed URL-a.
Poruka sadrzi samo referencu na blob + metadata.

Dobit:
- receiver moze skinuti ciphertext kad bude online
- retry/resume (range requests) i bolji UX
- backend i dalje ne vidi plaintext

Cijena:
- storage cost + retention/cleanup
- authz (tko smije downloadati blob) i revocation (kad member izadje iz grupe)
- vise komponenti (signed URLs, metrics, rate-limit)

## 5) Task backlog

### P0 (shared UX + stabilnost)

- [ ] Android: unify attachment send flow (isti kodpath 1:1 i group; samo se razlikuje recipients set)
- [ ] Android: stabilan progress + retry UI (upload/download states, error surface)
- [ ] Android: resumable download (ako je moguce u P2P) ili bar “retry from start” bez duplih zapisa
- [ ] Android: dedupe po `attachment_id` i/ili `sha256_ciphertext` (da retry ne napravi duple stavke)

### P1 (store-and-forward ciphertext, optional)

- [ ] Backend: endpointi za signed upload/download URL (JWT authz)
- [ ] Backend: storage key schema + metadata (ciphertext only)
- [ ] Backend: retention/cleanup job (TTL, max size per user, quotas)
- [ ] Backend: audit/metrics (upload/download bytes, error rates)
- [ ] Android: upload ciphertext blob -> store, send message sa `transport_ref`
- [ ] Android: download ciphertext blob -> decrypt lokalno -> render
- [ ] Android: revoke/expire handling (ako blob vise nije dostupan)

## 6) Interakcija s grupama

- Grupa ne smije imati “svoj” attachment kod.
- Group send samo definira *kome* ide poruka; attachment subsystem definira *kako* ide file payload.

