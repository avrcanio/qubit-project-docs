# Exit Node - Provisioning Checklist

- [ ] OS: server je up-to-date (`apt update && apt upgrade` ili ekvivalent)
- [ ] Firewall: omogucen SSH pristup; ostalo po potrebi (Exit Node je outbound-heavy)
- [ ] Docker: instaliran Docker + Compose
- [ ] TUN: postoji `/dev/net/tun`
- [ ] Headscale: kreiran preauth key za exit node (pozeljno ogranicen/tagged)
- [ ] Stack: deployan `/opt/stacks/headscale/exitnode/` na serveru
- [ ] Secrets: `.env` popunjen (TS_AUTHKEY), `.env` nije commit-an
- [ ] Start: `docker compose up -d` i container radi (`docker ps`)
- [ ] Join: node se vidi u Headscale (`headscale nodes list`)
- [ ] Approve routes: exit-node rute su enabled (Headscale routes/nodes routes approve)
- [ ] Forwarding: `net.ipv4.ip_forward=1` (persistirano na hostu)
- [ ] Test: klijent postavi exit node i provjeri public IP
