# Headscale direct-only (bez DERP) - status i runbook

Ovaj dokument pokriva M0 direct-only cilj i rezultat testiranja na trenutnoj verziji:

- Headscale image: `headscale/headscale:0.28.0-beta.1`
- Config file: `config/config.yaml`

## Cilj faze

1. Ne koristiti Tailscale public DERP map.
2. Ne koristiti embedded DERP server.
3. Ne koristiti custom DERP map (private DERP) u ovoj fazi.

## Pokusana konfiguracija (strict direct-only)

```yaml
derp:
  server:
    enabled: false
  urls: []
  paths: []
  auto_update_enabled: false
  update_frequency: 3h
```

## Rezultat testa

Na `headscale v0.28.0-beta.1` servis ne starta sa potpuno praznim DERP map inputom.

Relevantan log:

```text
initial DERPMap is empty, Headscale requires at least one entry
```

Zbog toga strict direct-only (bez ijedne DERP mape) trenutno nije operativan u ovoj verziji.

## Trenutno stabilno stanje (rollback)

Za stabilan rad, aktivna konfiguracija je vracena na:

```yaml
derp:
  server:
    enabled: false
  urls:
    - https://controlplane.tailscale.com/derpmap/default
  paths:
    - /etc/headscale/derpmap.json
  auto_update_enabled: true
  update_frequency: 3h
```

## Primjena i provjera

Primjena:

```bash
docker compose restart headscale
```

Provjera:

```bash
docker ps --filter name=headscale --format 'table {{.Names}}\t{{.Status}}'
docker compose logs --tail=200 headscale
```

## Known limitations

I dalje vazi:

- bez DERP fallback-a dio konekcija iza CGNAT/simetricnog NAT-a moze failati,
- direct-only cilj ostaje validan, ali treba rjesenje kompatibilno s `v0.28.0-beta.1` (ili upgrade i ponovni test).

## Napomena za M1

`config/derpmap.json` ostaje priprema za private DERP (HR VPS) i M1 fazu.
