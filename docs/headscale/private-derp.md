# Private DERP (HR VPS) - how-to i operativni checklist

Ovaj dokument pokriva M1 pripremu i uvodjenje privatnog DERP servera (npr. HR VPS) za Headscale deployment.

## Cilj

1. Ukloniti ovisnost o Tailscale public DERP mapi.
2. Zadrzati fallback/relay put za klijente iza CGNAT/simetricnog NAT-a.
3. Uvesti operativni postupak za dodavanje novih DERP regija.

## Lokacije fajlova

- Repo primjer mape: `examples/derpmap/derpmap.example.json`
- Aktivna mapa u Headscale stacku: `config/derpmap.json`
- Mount path u kontejneru: `/etc/headscale/derpmap.json`

## Headscale konfiguracija (bez public DERP URL-a)

U `config/config.yaml` koristi samo lokalnu mapu:

```yaml
derp:
  server:
    enabled: false
  urls: []
  paths:
    - /etc/headscale/derpmap.json
  auto_update_enabled: false
  update_frequency: 3h
```

Napomena: Headscale treba imati barem jednu DERP regiju u mapi, inace se servis nece podici.

## Primjer za HR VPS

Pretpostavke:
- DNS: `derp-hr.qubitsecured.online`
- HTTPS/TLS endpoint: `443/tcp`
- STUN endpoint: `3478/udp`

DERP cvor u mapi:

```json
{
  "Name": "hr",
  "RegionID": 2001,
  "RegionCode": "HR",
  "RegionName": "Croatia",
  "Nodes": [
    {
      "Name": "hr-1",
      "RegionID": 2001,
      "HostName": "derp-hr.qubitsecured.online",
      "DERPPort": 443,
      "STUNPort": 3478,
      "IPv4": "none",
      "IPv6": "none",
      "InsecureForTests": false,
      "CanPort80": false
    }
  ]
}
```

## Checklist: dodavanje novog DERP-a bez downtime

1. Podigni novi DERP endpoint (docker/systemd) i potvrdi da slusa na `443/tcp` i `3478/udp`.
2. Potvrdi DNS i TLS certifikat (`curl -fsS https://<hostname>/derp/probe`).
3. Dodaj novi region/node u `config/derpmap.json`.
4. Validiraj JSON (`jq . config/derpmap.json`).
5. Deploy-aj mapu atomicki (temp fajl pa rename/move na final path).
6. Restartaj Headscale (`docker compose restart headscale`).
7. Verificiraj logove i health (`docker compose logs --tail=200 headscale`).
8. Testiraj klijent konekciju iz barem dvije razlicite mreze (normalni NAT + restriktivna mreza).

## Operativne napomene

- `RegionID` mora biti jedinstven unutar mape.
- Po regiji mozes dodati vise `Nodes` radi HA.
- `Latitude` i `Longitude` su opcionalni metadata atributi; preporuceni su za bolji prikaz/odabir klijenta, ali nisu obavezni da DERP radi.
