# Nextcloud-Stack

Meine selbst gehostete Nextcloud mit allem, was dazugehört: Datenbank, Redis, Cron,
Collabora Online fürs Bearbeiten von Office-Dateien und eine zweite Instanz zum Testen.

```
               ┌──────────────┐
  Tunnel ────▶ │ app  :8080   │ Nextcloud 33 (eigenes Image mit Node.js für „Recognize“)
               └──┬───────┬───┘
                  │       │
          ┌───────▼─┐  ┌──▼────┐   ┌──────────────────┐   ┌────────────────────┐
          │ MariaDB │  │ Redis │   │ cron (Hintergrund)│   │ Collabora  :9980   │
          └─────────┘  └───────┘   └──────────────────┘   └────────────────────┘

               ┌──────────────────┐
               │ development :8181│ Test-Instanz, eigenes Volume
               └──────────────────┘
```

## Dienste

| Dienst | Image | Zweck |
|---|---|---|
| `app` | `nextcloud-custom:33` (aus `dockerfile/`) | Nextcloud, Apache, Port nur auf `127.0.0.1:8080` |
| `db` | `mariadb:10.11.19` (Digest fest) | Datenbank |
| `redis` | `redis:8.10.2-alpine` (Digest fest) | Datei-Sperren und Cache, mit Passwort |
| `cron` | wie `app` | Hintergrundjobs (`/cron.sh`) |
| `collabora` | `collabora/code` (Digest fest) | Office im Browser, `127.0.0.1:9980` |
| `development` | `nextcloud:33` (Digest fest) | Spielwiese, `127.0.0.1:8181` |

Von außen ist nichts direkt offen: Alle Ports hängen an `127.0.0.1`, erreichbar ist der
Stack über einen Cloudflare-Tunnel.

## Einrichten

```bash
cp env/app.env.example env/app.env           # und die anderen *.env.example genauso
# Passwörter eintragen, z. B. mit: openssl rand -base64 24
chmod 600 env/*.env
docker compose up -d --build
```

| Datei | Inhalt |
|---|---|
| `env/app.env` | `MYSQL_PASSWORD`, `REDIS_HOST_PASSWORD`, `NEXTCLOUD_ADMIN_PASSWORD` |
| `env/db.env` | `MYSQL_ROOT_PASSWORD`, `MYSQL_PASSWORD` (gleich wie oben) |
| `env/redis.env` | `REDIS_PASSWORD` (gleich wie `REDIS_HOST_PASSWORD`) |
| `env/collabora.env` | `password` für die Collabora-Admin-Konsole |
| `env/development.env` | `NEXTCLOUD_ADMIN_PASSWORD` der Test-Instanz |

Die echten `env/*.env` stehen in `.gitignore` und landen nie im Repo.

Pfade auf dem Host: `/root/nextcloud/{html,config,custom_apps,themes,db}` und die Daten unter
`/srv/nextcloud-data`. Anpassen, wenn dein Server anders aussieht.

## Härtung

- Feste Versionen (Tag **und** Digest): Updates passieren nur bewusst.
- `no-new-privileges` und Healthchecks bei allen Diensten außer Collabora. Collabora braucht
  Datei-Capabilities für `coolforkit`, die `no-new-privileges` abschalten würde.
- Passwörter nur in `env/*.env` (Rechte 600), nicht in der Compose-Datei.

## Update

1. In `docker-compose.yml` Tag + Digest anpassen und im `dockerfile/Dockerfile` die Basis
   (`FROM nextcloud:33-apache`). Nextcloud nur **eine Hauptversion pro Schritt** anheben.
2. `docker compose build app && docker compose up -d`
3. `docker exec -u www-data nextcloud-app php occ status` und `occ upgrade`, falls nötig.
