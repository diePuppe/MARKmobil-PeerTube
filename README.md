# Markmobil – Custom PeerTube Platform

> **Markmobil** ist eine datenschutzkonforme VOD-Plattform, aufgebaut auf [PeerTube](https://joinpeertube.org), bereitgestellt als Managed-Dienst für Content-Creator und Medienunternehmen.

Der erste Kunde ist ein **Reise & Politik Kanal** mit ca. 120.000 Nutzer:innen.

---

## Was ist Markmobil?

Markmobil ist ein vorkonfiguriertes, DSGVO-konformes PeerTube-Setup mit:

- 🎨 **Custom Theme** – Dark Navy/Grün/Gold, Serif-Headlines, Weltkarten-Hintergrund
- 🔒 **Keine Registrierung** – Videos öffentlich schaubar, kein Cookie-Banner nötig
- 📊 **Plausible Analytics** – datenschutzfreundliche Statistiken, self-hosted
- 🐳 **Docker Compose** – PeerTube + PostgreSQL + Redis + Nginx + Let's Encrypt
- 🗂️ **Kategorien** – Reise, Politik, Reportage mit Länder-Tags
- 📺 **Auto-Transcoding** – 360p, 720p, 1080p
- 🚀 **CDN-ready** – lokale Speicherung jetzt, Bunny.net CDN später einfach einhängbar

---

## Voraussetzungen

- Hetzner VPS (Ubuntu 22.04, empfohlen: CX21 oder größer)
- Domain mit DNS-Zugang (A-Record → Server-IP)
- Docker & Docker Compose installiert
- `gh` CLI (optional, für GitHub-Setup)

---

## Deployment auf Hetzner VPS (Ubuntu 22.04)

### 1. Server vorbereiten

```bash
# System aktualisieren
sudo apt update && sudo apt upgrade -y

# Docker installieren
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker

# Docker Compose Plugin prüfen
docker compose version
```

### 2. Repository klonen

```bash
git clone https://github.com/diePuppe/MARKmobil-PeerTube.git /opt/markmobil
cd /opt/markmobil
```

### 3. Umgebungsvariablen konfigurieren

```bash
cp .env.example .env
nano .env   # Alle CHANGE_ME Platzhalter ersetzen!
```

Wichtige Werte in `.env`:
| Variable | Beschreibung |
|---|---|
| `PEERTUBE_WEBSERVER_HOSTNAME` | Deine Domain (z.B. `videos.example.com`) |
| `PEERTUBE_SECRET` | Zufallsstring: `openssl rand -hex 32` |
| `PEERTUBE_DB_PASSWORD` | Starkes Datenbankpasswort |
| `VIDEO_STORAGE_PATH` | Absoluter Pfad für Videospeicher (z.B. `/opt/markmobil/videos`) |
| `PLAUSIBLE_DOMAIN` | Domain für Plausible (z.B. `stats.example.com`) |
| `PLAUSIBLE_SECRET_KEY_BASE` | `openssl rand -base64 64 \| tr -d '\n'` |

### 4. Videospeicher erstellen

```bash
sudo mkdir -p /opt/markmobil/videos
sudo chown -R 999:999 /opt/markmobil/videos  # PeerTube Container-UID
```

### 5. Nginx Konfiguration anpassen

```bash
# YOUR_DOMAIN in nginx/peertube.conf durch deine echte Domain ersetzen
sed -i 's/YOUR_DOMAIN/videos.example.com/g' nginx/peertube.conf
```

### 6. SSL-Zertifikat ausstellen (Let's Encrypt)

```bash
# Certbot initial ausführen (HTTP muss erreichbar sein)
docker compose up nginx certbot -d

docker compose run --rm certbot certonly \
  --webroot --webroot-path=/var/www/certbot \
  -d videos.example.com \
  -d stats.example.com \
  --email admin@example.com \
  --agree-tos --no-eff-email
```

### 7. Stack starten

```bash
docker compose up -d
docker compose logs -f peertube   # Logs beobachten
```

### 8. Admin-Account einrichten

```bash
# Initial-Passwort aus den Logs holen
docker compose logs peertube | grep -i password
```

Dann im Browser: `https://videos.example.com` → Login → Einstellungen → Passwort ändern.

---

## Nginx Log-Rotation (DSGVO: Logs nach 24h löschen)

```bash
sudo nano /etc/logrotate.d/markmobil-nginx
```

Inhalt:
```
/var/lib/docker/volumes/*/nginx/*.log {
    daily
    rotate 1
    compress
    missingok
    notifempty
    postrotate
        docker compose -f /opt/markmobil/docker-compose.yml exec nginx nginx -s reopen
    endscript
}
```

---

## Theme anpassen (CSS / Farben / Logo)

### CSS hochladen

Die Datei `theme/custom.css` enthält das komplette Markmobil-Design. Sie wird in PeerTube unter **Administration → Konfiguration → Erweitertes → Benutzerdefiniertes CSS** eingefügt.

```bash
cat theme/custom.css | xclip -selection clipboard   # Kopieren
```

Dann im PeerTube Admin-Panel einfügen.

### Farben ändern

Die CSS Custom Properties am Anfang der Datei anpassen:
```css
:root {
  --mm-navy:   #0d1b2a;   /* Haupthintergrund */
  --mm-green:  #1a4a3a;   /* Sekundärfarbe */
  --mm-gold:   #c9a84c;   /* Akzentfarbe */
}
```

### Logo einbinden

In PeerTube Admin → **Konfiguration → Instanz** → Logo hochladen.
Empfohlenes Format: SVG oder PNG, 200×60px.

---

## Plausible Analytics einbinden

### Script-Tag in PeerTube einfügen

In PeerTube Admin → **Konfiguration → Erweitertes → Benutzerdefinierter JavaScript-Header**:

```html
<script defer data-domain="videos.example.com"
  src="https://stats.example.com/js/script.js"></script>
```

Plausible ist vollständig DSGVO-konform: kein Cookie, keine IP-Speicherung.

---

## CDN-Erweiterung (Bunny.net) – später

Wenn CDN hinzugefügt wird:

1. In `peertube/production.yaml` den `object_storage`-Block auskommentieren und Bunny.net-Zugangsdaten eintragen
2. Videos in Bunny.net Storage Zone hochladen oder Pull-Zone auf den lokalen Server zeigen lassen
3. `PEERTUBE_WEBSERVER_HOSTNAME` bleibt gleich, nur `base_url` in `object_storage` ändert sich

---

## Footer & Impressum (AGPL-Pflicht)

Im PeerTube Admin-Panel unter **Konfiguration → Instanz → Beschreibung** folgenden Text hinzufügen:

```
Markmobil basiert auf PeerTube (AGPL-3.0).
Quellcode: https://github.com/diePuppe/MARKmobil-PeerTube

Impressum: [Dein Name], [Adresse], [E-Mail]
Datenschutz: Keine Cookies, keine Registrierung, anonyme Statistiken via Plausible.
```

Dieser Text wird auch automatisch im Footer via `theme/custom.css` (CSS `::after`) angezeigt.

---

## Kategorien & Tags

PeerTube unterstützt von Haus aus Kategorien und Tags. Standardkategorien aktivieren oder eigene über die Admin-API anlegen:

```bash
# Tag-Beispiele für Länder
# Im Admin-Panel: Video hochladen → Tags → "Deutschland", "Türkei", "USA" etc.
# Kategorien: Reise (17), Politik (14) – aus PeerTube-Standardkategorien
```

---

## AGPL-Lizenz

Markmobil basiert auf [PeerTube](https://github.com/Chocobozzz/PeerTube), Copyright © Framasoft, lizenziert unter der [GNU Affero General Public License v3.0](https://www.gnu.org/licenses/agpl-3.0).

Alle in diesem Repository enthaltenen Konfigurationen und Anpassungen sind ebenfalls unter AGPL-3.0 veröffentlicht.

**Quellcode:** [github.com/diePuppe/MARKmobil-PeerTube](https://github.com/diePuppe/MARKmobil-PeerTube)

---

## Repository-Struktur

```
MARKmobil-PeerTube/
├── docker-compose.yml          # Hauptstack
├── .env.example                # Platzhalter-Konfiguration (KEIN echtes .env!)
├── .gitignore                  # Schützt .env vor versehentlichem Commit
├── nginx/
│   └── peertube.conf           # Reverse Proxy + SSL
├── peertube/
│   └── production.yaml         # PeerTube-Konfiguration
├── theme/
│   └── custom.css              # Markmobil Dark Theme
├── plausible/
│   ├── docker-compose.yml      # Plausible Standalone-Referenz
│   ├── clickhouse-config.xml   # ClickHouse Logging (24h TTL)
│   └── clickhouse-user-config.xml
├── LICENSE                     # AGPL-3.0
└── README.md                   # Diese Datei
```

---

*Bereitgestellt von Markmobil – Datenschutzfreundliche VOD-Plattform.*
