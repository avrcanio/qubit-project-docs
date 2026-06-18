# Exit Node (Headscale) - Docker Compose

Ovaj dokument opisuje kako podici Exit Node (Tailscale client) koji se spaja na Headscale control server, i kako ga odobriti na Headscale strani.

Referentna implementacija zivi u Headscale stacku:

- `/opt/stacks/headscale/exitnode/` (`docker-compose.yml`, `.env.example`, `README.md`)

## Preduvjeti

- Linux server s Dockerom + Docker Compose
- `/dev/net/tun` dostupan (Tailscale tun device)
- pristup Headscale serveru (CLI ili admin pristup) za odobravanje exit-node ruta
- Headscale preauth key (preporuka: reusable + ogranicen, ili tag + auto-approvers kroz ACL)

## Quick start (na exit node serveru)

```bash
cd /opt/stacks/headscale/exitnode
cp .env.example .env
# upisi TS_AUTHKEY u .env

docker compose up -d
docker logs -f tailscale-exitnode
```

Napomena:

- `.env` sadrzi tajnu i ne ide u git; koristi `.env.example` kao template.
- Compose ukljucuje `--advertise-exit-node` i `--accept-dns=false`.

## Odobravanje exit node ruta (na Headscale serveru)

Exit node setup je "double opt-in": node mora oglasiti exit-node rute, a control server mora odobriti/enable te rute.

```bash
docker exec headscale headscale nodes list-routes
docker exec headscale headscale nodes approve-routes --identifier <NODE_ID> --routes 0.0.0.0/0,::/0
```

## IP forwarding (na exit node serveru)

Compose postavlja `net.ipv4.ip_forward=1` unutar kontejnera. Na hostu preporuceno persistirati:

```bash
printf '%s\n' 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-qubit-exitnode.conf
sudo sysctl --system
```

## Klijent: koristenje exit node-a

Na klijentu (npr. Windows) odaberi exit node u aplikaciji ili:

```bash
tailscale set --exit-node qubit-fi
```

Za provjeru, provjeri javni IP na klijentu: treba postati IP exit node servera.

## Windows: spajanje na Headscale

1. Instaliraj Tailscale for Windows
2. U PowerShell:

```powershell
tailscale up --login-server=https://hs-control.qubitsecured.online --authkey=<PREAUTH_KEY>
```

3. Za exit node: odaberi `qubit-fi` u Tailscale aplikaciji
