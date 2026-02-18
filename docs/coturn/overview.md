# coturn overview

Ovaj setup pokrece `coturn` kao Docker servis za WebRTC relay (STUN/TURN) sa long-term auth secret modelom.

## Trenutna implementacija (dedicated server)

- domen: `turn.qubitsecured.online`
- coturn stack: `/opt/stacks/coturn`
- image: `coturn/coturn:4.7.0-r2`
- auth: `use-auth-secret` + `static-auth-secret`
- backend endpoint za creds: `GET /api/webrtc/turn-credentials`
- secret file: `/srv/secrets/turn-static-auth-secret/secret`

## Portovi

- `3479/udp` STUN/TURN UDP
- `3479/tcp` TURN TCP
- `49160-49200/udp` relay range
- `5349/tcp` TURN-TLS (`turns:`) aktivan

Napomena: standardni `3478` je trenutno zauzet DERP STUN servisom, zato je coturn postavljen na `3479`.

## Backend integracija

Qubit backend generise TURN kredencijale pomocu HMAC-SHA1 (`qid:expiry`) i shared secret-a.

Relevant env:

- `TURN_HOST=turn.qubitsecured.online`
- `TURN_PORT=3479`
- `TURN_TLS_PORT=5349`
- `TURN_STATIC_AUTH_SECRET_FILE=/run/secrets/turn_static_auth_secret/secret`
- `TURN_INCLUDE_TURNS_URLS=true`
- `TURN_ENABLE_TLS=true`
