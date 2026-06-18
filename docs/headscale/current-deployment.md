# Headscale Docker deployment (trenutno stanje)

Ovaj dokument opisuje trenutno aktivnu Docker postavku za Headscale iz projekta u `/opt/stacks/headscale`.

## Lokacija deployment fajlova

- Compose: `docker-compose.yml`
- Headscale config: `config/config.yaml`
- Exit node: `exitnode/docker-compose.yml` (zasebni stack, isti server)
- Okruzenje: `.env`

## Docker Compose pregled

### Servisi

1. `headscale`
2. `nginx-note`
3. `register-ui`

### Exit node (zasebni compose u `exitnode/`)

- Container: `tailscale-exitnode`
- Hostname u tailnetu: `qubit-fi` (`100.64.0.2`)
- Pokretanje: `cd /opt/stacks/headscale/exitnode && docker compose up -d`
- State: `./exitnode/tailscale-state/` (bind mount, nije u gitu)

### Headscale servis

- Image: `headscale/headscale:0.28.0-beta.1`
- Container name: `headscale`
- Command: `serve --config /etc/headscale/config.yaml`
- Restart policy: `unless-stopped`

### Mount-ovi (volumes)

`headscale`:
- `./config:/etc/headscale`
- `./data:/var/lib/headscale`

`nginx-note`:
- `./data:/usr/share/nginx/html:ro`

`register-ui`:
- `/var/run/docker.sock:/var/run/docker.sock`

### Mreze

- `backend` (external network)
- `proxy` (external network)

`headscale` je prikacen na obje mreze (`backend`, `proxy`), dok su `nginx-note` i `register-ui` samo na `proxy`.

## Portovi i izlaganje servisa

### Docker/Compose port model

- Nema `ports:` mapiranja na host za `headscale`.
- `headscale` koristi `expose: 8080` (dostupno unutar Docker mreza, ne direktno s hosta/Interneta).
- `nginx-note` koristi `expose: 80`.
- `register-ui` koristi `expose: 5000`.

### Headscale portovi u `config/config.yaml`

- `listen_addr: 0.0.0.0:8080` (HTTP API/control plane endpoint u kontejneru)
- `grpc_listen_addr: 0.0.0.0:50443`
- `metrics_listen_addr: 127.0.0.1:9090`

## Domena i TLS model

### Javni URL

- `server_url`: `https://hs-control.qubitsecured.online`

### TLS terminacija

TLS terminira Traefik (reverse proxy), kroz label-e na `headscale` servisu:

- Router rule: `Host(\`hs-control.qubitsecured.online\`)`
- Entrypoint: `websecure`
- Cert resolver: `le`
- Upstream service port (u kontejneru): `8080`

Prakticno: Headscale sam ne terminira javni TLS na hostu; Traefik prima HTTPS i prosljedjuje prema `headscale:8080` unutar Docker mreze `proxy`.

## Baza podataka

U trenutnoj konfiguraciji Headscale koristi Postgres:

- type: `postgres`
- host: `postgis`
- port: `5432`
- db: `headscale`
- user: `headscale`

Napomena:
- DB parametri su definirani i u `.env` i direktno u `config/config.yaml`.
- Lozinka je trenutno zapisana u `config/config.yaml` (plaintext), sto je funkcionalno ali sigurnosno slabije od secret/env pristupa.

## DERP stanje (trenutno)

Iz `config/config.yaml`:

- Embedded DERP server: `enabled: false`
- Public DERP URL je definiran: `https://controlplane.tailscale.com/derpmap/default`
- Dodatni lokalni derp map path postoji: `/etc/headscale/derpmap.json`
- `auto_update_enabled: true`

To znaci da je embedded DERP trenutno ugasen, ali konfiguracija i dalje ucitava DERP mapu (public + lokalna putanja).
