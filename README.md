# GeoMetrikks Unraid Templates

Unraid Community Apps templates for [GeoMetrikks](https://github.com/GilbN/geometrikks) - a real-time geolocation analytics tool that tails your reverse proxy's access logs, does GeoIP lookups, and visualizes traffic on a live interactive map. It reads nginx, Traefik JSON and Caddy JSON logs, and works out which is which per file.

This repository ships two Docker templates:

| Template | What it is |
|---|---|
| [`geometrikks-timescaledb`](templates/geometrikks-timescaledb.xml) | TimescaleDB + PostGIS database - install this first |
| [`geometrikks`](templates/geometrikks.xml) | The GeoMetrikks app and web UI |

## Setup

### 1. Install the database

Add `geometrikks-timescaledb` from Community Apps. Set a **POSTGRES_PASSWORD** - you'll reuse this exact value in step 2. Leave POSTGRES_USER (`geouser`) and POSTGRES_DB (`geometrikks`) at their defaults unless you have a reason to change them.

The DB Port config (default `5432`) is exposed to your LAN so the app container can reach it.

The template starts Postgres with `timescaledb.max_background_workers=40` (and the matching `max_parallel_workers` / `max_worker_processes`). GeoMetrikks registers around 32 TimescaleDB jobs whose refresh policies all fire on the same tick, and the image's default of 16 workers can't cover that: the database log fills with `failed to launch job ... out of background workers` and continuous aggregates stop refreshing. If you installed this template before September 2026, remove and re-add the container to pick the setting up, or add the three `-c` flags yourself under Post Arguments.

### 2. Install the app

Add `geometrikks` from Community Apps and fill in:

- **DB_HOST** - your Unraid server's IP address (e.g. `192.168.1.50`)
- **DB_PASSWORD** - the *same* password you set for `geometrikks-timescaledb` in step 1
- **APP_ADMIN_USER** / **APP_ADMIN_PASSWORD** - your web UI login or **APP_AUTH_DISABLED=true**
- **Access Logs** path - point this at wherever your reverse proxy writes its access logs (defaults to a SWAG-style path; change it for Nginx Proxy Manager, Traefik, Caddy, or whatever you actually run)

Leave DB_PORT/DB_USER/DB_DATABASE at their defaults unless you changed the matching values in step 1.

### PUID / PGID and file ownership

The app container starts as root only long enough to remap its internal user to **PUID**:**PGID**, fix ownership of the *GeoIP Data* and *App Logs* folders, and then drop privileges - the app itself never runs as root. The template defaults to Unraid's usual `99`:`100` (`nobody`:`users`), which matches the rest of your appdata and the ownership SWAG/Nginx Proxy Manager give their log files, so no manual `chown` step is needed.

Your **Access Logs** mount is read-only and is *not* touched by that fix-up - those files just need to be readable by PUID:PGID on the host.

### App logs

The *App Logs* path mount holds GeoMetrikks' own logs:

- `geometrikks.log` - structured JSONL application log
- `login.log` - plain-text login/logout/failed-login events, in a format fail2ban and CrowdSec can parse

Both rotate by size and gzip their archives (`LOG_MAIN_*` / `LOG_LOGIN_*`). You can also browse and download them in the web UI under **Settings -> Logs**. Mounting the path is optional but recommended: without it the logs are lost whenever the container is recreated, and host-side tools can't read `login.log`.

### Which log format?

The mount lands at `/var/log/nginx` inside the container whatever proxy you run, and **LOGPARSER_LOG_PATHS** names the file there. The format is detected per file, so Traefik and Caddy JSON logs need nothing beyond the path. Pin it with **LOGPARSER_LOG_FORMATS** (`geometrikks-json`, `nginx`, `traefik-json`, `caddy-json`) if detection gets it wrong.

Nginx is the one that needs work on your side: GeoMetrikks reads a keyed JSON `log_format` you have to add to your `nginx.conf`. The older positional format still parses, so an existing setup keeps working. Both are in the [upstream README](https://github.com/GilbN/geometrikks#nginx-setup).

Tailing several files? **LOGPARSER_LOG_PATHS** takes a JSON list, and giving **LOGPARSER_HOST_NAME** a list of the same length labels each file separately, so the UI can filter them as separate sources.

### 3. Get a free MaxMind GeoLite2 key

GeoMetrikks needs MaxMind's free GeoLite2 databases for GeoIP lookups:

1. Sign up at <https://www.maxmind.com/en/geolite2/signup>
2. Create a license key under your account
3. Set **MAXMINDDB_USER_ID** and **MAXMINDDB_LICENSE_KEY** on the `geometrikks` template

Without these, GeoMetrikks runs in degraded mode (no GeoIP lookups) until you add them.

Two databases are downloaded into the *GeoIP Data* mount: City, which drives the map and geo analytics, and ASN, which records the network behind each request and fills the Top ASNs view. Set **GEOIP_ASN_ENABLED=false** to skip the second one.

### 4. Open the web UI

`http://<your-unraid-ip>:8000/` - log in with the admin credentials you set in step 2.

## Notes

- Full environment variable reference: <https://github.com/GilbN/geometrikks/blob/main/docs/configuration.md>. Most of it is exposed as "advanced" settings on the `geometrikks` template - expand "Show more settings" when adding the container to see database pool tuning, log parser tuning, log rotation, analytics retention, and map settings.
- **LOGPARSER_IGNORE_IPS** is worth setting: give it your own public IP (or any CIDR) to drop that traffic entirely - no geo event, access log or debug row. LAN traffic is never ingested in the first place, so this is for your own WAN-side hits.
- Reaching the UI through a TLS reverse proxy? Set **APP_SESSION_SECURE=true**, and put your proxy's IP/CIDR in **APP_TRUSTED_PROXIES** (e.g. `172.17.0.0/16`) so client IPs are read from `X-Forwarded-For`. Leave `APP_SESSION_SECURE` at `false` if you browse to `http://<unraid-ip>:8000` directly, or logins won't stick.
- **API_LOG_LEVEL** is deprecated - use **LOG_LEVEL**. If you're updating an existing container, clear the old `API_LOG_LEVEL` value.
- **APP_PROXY_ADVISORY** puts a warning on *Settings > Status* when the traffic in a tailed file comes from CDN or private addresses, which means your proxy is logging the hop in front of it instead of the visitor. Set it to `false` if that's deliberate (Tailscale-only access, a CDN you front on purpose).
- **APP_MODE** is `full` here and should stay that way. `agent` turns the container headless: no UI, no API, just tail, geolocate and write into a shared database. It's for a second GeoMetrikks next to another proxy, feeding this one.
- The template sets `--stop-timeout=20` so the app can finish flushing its ingestion batch on shutdown. Unraid's default of 10 seconds kills it mid-write.
- GeoMetrikks itself: <https://github.com/GilbN/geometrikks>
