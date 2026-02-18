# Exit Node (Headscale) - Docker Compose

Ovaj dokument opisuje kako podici Exit Node (Tailscale client) koji se spaja na Headscale control server, i kako ga odobriti na Headscale strani.

Referentna implementacija za server-side compose zivi u repou:

- `avrcanio/dedicated-server-exitnode` (`docker-compose.yml`, `.env.example`)

## Preduvjeti

- Linux server s Dockerom + Docker Compose
- `/dev/net/tun` dostupan (Tailscale tun device)
- pristup Headscale serveru (CLI ili admin pristup) za odobravanje exit-node ruta
- Headscale preauth key (preporuka: reusable + ogranicen, ili tag + auto-approvers kroz ACL)

## Quick start (na exit node serveru)

U `dedicated-server-exitnode` repou:

```bash
cp .env.example .env
# upisi TS_AUTHKEY u .env

docker compose up -d
docker logs -f tailscale-exitnode
```

Napomena:

- `.env` sadrzi tajnu i ne ide u git; koristi `.env.example` kao template.
- `TS_EXTRA_ARGS` vec ukljucuje `--advertise-exit-node` i `--accept-dns=false`.

## Odobravanje exit node ruta (na Headscale serveru)

Exit node setup je "double opt-in": node mora oglasiti exit-node rute, a control server mora odobriti/enable te rute.

Ako koristis noviji Headscale CLI (routes commands):

```bash
headscale routes list
# pronadji rute 0.0.0.0/0 i/ili ::/0 za exit node
headscale routes enable -r <ROUTE_ID>
```

Ako koristis Headscale varijantu s "nodes list-routes":

```bash
headscale nodes list-routes
headscale nodes approve-routes --identifier <NODE_ID> --routes 0.0.0.0/0
```

## IP forwarding + NAT (na exit node serveru)

Da bi exit node stvarno routao internet promet, moras ukljuciti IP forwarding i NAT.

IPv4 forwarding (privremeno, do restarta):

```bash
sysctl -w net.ipv4.ip_forward=1
```

Persist (preko `/etc/sysctl.d/`):

```bash
printf '%s\n' 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-qubit-exitnode.conf
sudo sysctl --system
```

NAT (iptables primjer, prilagodi `eth0` ako ti je WAN interface drugaciji):

```bash
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

## Klijent: koristenje exit node-a

Na klijentu (npr. Android ili Linux) odaberi exit node u aplikaciji ili:

```bash
tailscale set --exit-node <EXITNODE_HOSTNAME>
```

Za provjeru, provjeri javni IP na klijentu: treba postati IP exit node servera.

