# Private DERP - how-to

Ovaj dokument pokriva M1 pripremu i uvodjenje privatnog DERP servera za Headscale deployment.

## Cilj

1. Ukloniti ovisnost o Tailscale public DERP mapi.
2. Zadrzati fallback/relay put za klijente iza CGNAT/simetricnog NAT-a.
3. Definirati minimalne smjernice za private DERP mapu.

## Lokacije fajlova

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

## Operativne napomene

- `RegionID` mora biti jedinstven unutar mape.
- Po regiji mozes dodati vise `Nodes` radi HA.
- `Latitude` i `Longitude` su opcionalni metadata atributi; preporuceni su za bolji prikaz/odabir klijenta, ali nisu obavezni da DERP radi.
