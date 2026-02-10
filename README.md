# cloudflareSetupNoorderpoort
Voor studenten van de software developement deepdive
# Cloudflare Tunnel Workshop

## Voordat je begint

Je hebt het volgende nodig:

- Een Ubuntu server (VM) waar je root-toegang op hebt
- Cloudflare-inloggegevens (krijg je van de docent) met toegang tot een domein

---

## Stap 1 — Cloudflared installeren

Open een terminal op je Ubuntu server en voer de volgende commando's uit als root:

```bash
sudo apt-get update
sudo apt-get install -y curl lsb-release
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt-get update
sudo apt-get install -y cloudflared
```

## Stap 2 — Inloggen bij Cloudflare

```bash
cloudflared tunnel login
```

Er verschijnt een URL in je terminal. Open deze in een browser en log in met de Cloudflare-account die je hebt gekregen. Kies het juiste domein wanneer daarom wordt gevraagd.

Na het inloggen wordt er een certificaat opgeslagen in `~/.cloudflared/cert.pem`.

## Stap 3 — Tunnel aanmaken

Maak een tunnel aan en geef deze een herkenbare naam:

```bash
cloudflared tunnel create mijn-tunnel
```

Vervang `mijn-tunnel` door een naam naar keuze. Er wordt nu een credentials-bestand aangemaakt in `~/.cloudflared/`.

Bekijk je tunnel met:

```bash
cloudflared tunnel list
```

Noteer de **tunnel-ID** (een lange reeks letters en cijfers), die heb je nodig in de volgende stap.

## Stap 4 — Config aanmaken

Maak de map `/etc/cloudflared/` aan en kopieer je credentials daarheen:

```bash
sudo mkdir -p /etc/cloudflared
sudo cp ~/.cloudflared/<TUNNEL-ID>.json /etc/cloudflared/
sudo chmod 600 /etc/cloudflared/<TUNNEL-ID>.json
```

Maak het configuratiebestand aan:

```bash
sudo nano /etc/cloudflared/config.yml
```

Plak de volgende inhoud en pas de waardes aan:

```yaml
tunnel: <TUNNEL-ID>
credentials-file: /etc/cloudflared/<TUNNEL-ID>.json

ingress:
  - hostname: app.jouwdomein.nl
    service: http://localhost:8080

  # Voeg meer hostnames toe als je meerdere services hebt:
  # - hostname: api.jouwdomein.nl
  #   service: http://localhost:3000

  # Voor services die zelf HTTPS draaien met een self-signed certificaat:
  # - hostname: portainer.jouwdomein.nl
  #   service: https://localhost:9443
  #   originRequest:
  #     noTLSVerify: true

  - service: http_status:404  # Catch-all voor onbekende hostnames, MOET erin staan
```

Vervang `<TUNNEL-ID>` door de ID uit stap 3 en pas de hostnames en poorten aan naar jouw situatie.

Valideer je config:

```bash
cloudflared tunnel --config /etc/cloudflared/config.yml ingress validate
```

## Stap 5 — DNS records aanmaken

Koppel een hostname aan je tunnel:

```bash
cloudflared tunnel route dns mijn-tunnel app.jouwdomein.nl
```

Dit maakt automatisch een CNAME-record aan in Cloudflare DNS. Herhaal dit voor elke hostname die je in je config hebt staan.

## Stap 6 — Tunnel als service installeren

Omdat de config en credentials al in `/etc/cloudflared/` staan, kun je de service direct installeren:

```bash
sudo cloudflared service install
```

Start en controleer de service:

```bash
sudo systemctl start cloudflared
sudo systemctl status cloudflared
```

Open je hostname (bijv. `app.jouwdomein.nl`) in een browser om te testen of je applicatie bereikbaar is.

## Stap 7 — Service beheren

Handige commando's voor het beheren van de tunnel-service:

```bash
sudo systemctl status cloudflared    # Status bekijken
sudo systemctl restart cloudflared   # Herstarten
sudo systemctl stop cloudflared      # Stoppen
journalctl -u cloudflared -f         # Logs volgen
```

Wanneer je later je config wilt wijzigen (bijv. een nieuwe hostname toevoegen):

1. Bewerk `/etc/cloudflared/config.yml`
2. Maak het DNS-record aan: `cloudflared tunnel route dns mijn-tunnel nieuwe-host.jouwdomein.nl`
3. Herstart de service: `sudo systemctl restart cloudflared`

---

## Veelvoorkomende problemen

**Tunnel start niet na `service install`**
Controleer of er geen dubbele config bestaat in zowel `~/.cloudflared/` als `/etc/cloudflared/`. De service leest alleen uit `/etc/cloudflared/`. Verwijder `~/.cloudflared/config.yml` om verwarring te voorkomen.

**"No ingress rules" fout**
De catch-all regel `- service: http_status:404` ontbreekt onderaan je ingress-blok. Deze is verplicht.

**Site niet bereikbaar**
- Controleer of je applicatie lokaal draait op de juiste poort
- Controleer of het DNS-record is aangemaakt via `cloudflared tunnel route dns` of via het Cloudflare dashboard
- Bekijk de logs: `journalctl -u cloudflared -f`
