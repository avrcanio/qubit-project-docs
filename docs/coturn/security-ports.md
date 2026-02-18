# coturn security and ports

## Otvoreni portovi

Minimalno za trenutni deployment:

- `3479/udp`
- `3479/tcp`
- `49160-49200/udp`

Opcionalno za TURN-TLS:

- `5349/tcp`

## Hardening preporuke

- Koristiti `use-auth-secret` + kratki TTL kredencijala (npr. 600s)
- Ne drzati static secret u git-u; koristiti root-only fajl na hostu
- Ograniciti relay range na potreban minimum
- Ukljuciti detaljne logove samo tokom testiranja (`TURN_VERBOSE=true`), zatim ugasiti
- Periodicno rotirati shared secret i redeploy backend + coturn

## Primijenjene firewall/rate-limit mjere

- `DOCKER-USER` chain jump na coturn portove:
  - TCP: `3479,5349`
  - UDP: `3479`, `49160-49200`
- per-source TCP new-connection limit: `300/min` (burst `400`)
- per-source UDP packet cap: `2000/s` (burst `4000`)
- systemd persistencija:
  - `/etc/systemd/system/coturn-firewall.service`
  - script `/opt/stacks/coturn/scripts/apply-firewall.sh`

## Kontekst trenutnog servera

Na hostu vec postoji DERP STUN na `3478/udp` preko Traefik-a. Zbog toga je coturn trenutno mapiran na `3479`.

Ako se zeli standardni TURN na `3478`, potrebno je planirati migraciju DERP STUN porta i uskladiti derpmap konfiguraciju.
