# coturn docker compose

## Lokacija stack-a

- compose: `/opt/stacks/coturn/docker-compose.yml`
- env: `/opt/stacks/coturn/.env`
- start script: `/opt/stacks/coturn/scripts/start-coturn.sh`

## Pokretanje

```bash
docker compose -f /opt/stacks/coturn/docker-compose.yml --env-file /opt/stacks/coturn/.env up -d
```

## Stop/Restart

```bash
docker compose -f /opt/stacks/coturn/docker-compose.yml --env-file /opt/stacks/coturn/.env down
docker compose -f /opt/stacks/coturn/docker-compose.yml --env-file /opt/stacks/coturn/.env up -d --force-recreate
```

## Verifikacija

```bash
docker ps --filter name=coturn
docker logs --tail 100 coturn
ss -lunp | rg ':3479|:4916|:4920'
ss -ltnp | rg ':3479|:5349'
echo | openssl s_client -connect turn.qubitsecured.online:5349 -servername turn.qubitsecured.online -brief
```

## Firewall / rate limiting

Coturn firewall hardening se primjenjuje kroz `DOCKER-USER` chain:

- script: `/opt/stacks/coturn/scripts/apply-firewall.sh`
- systemd unit: `coturn-firewall.service` (enabled)

## Secret rotacija

1. Upisi novi random secret u `/srv/secrets/turn-static-auth-secret/secret`
2. Restartuj `coturn`
3. Restartuj `qubit` servis (backend mora ucitati isti secret)

```bash
umask 077
openssl rand -base64 48 > /srv/secrets/turn-static-auth-secret/secret
chmod 600 /srv/secrets/turn-static-auth-secret/secret

docker compose -f /opt/stacks/coturn/docker-compose.yml --env-file /opt/stacks/coturn/.env up -d --force-recreate
docker compose -f /opt/stacks/qubit/docker-compose.yml up -d qubit
```
