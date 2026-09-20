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
- **APP_ADMIN_USER** / **APP_ADMIN_PASSWORD** - your web UI login, or **APP_AUTH_DISABLED=true**, or an identity provider (see below)
- **Access Logs** path - point this at wherever your reverse proxy writes its access logs (defaults to a SWAG-style path; change it for Nginx Proxy Manager, Traefik, Caddy, or whatever you actually run)

Leave DB_PORT/DB_USER/DB_DATABASE at their defaults unless you changed the matching values in step 1.

### Signing in with an identity provider

GeoMetrikks 0.17.0 added OpenID Connect login, so the login page can hand you off to Authelia, Authentik, Keycloak, Pocket ID or Google. Four variables switch it on:

- **OIDC_ISSUER** - the provider's issuer URL, e.g. `https://auth.example.com`
- **OIDC_CLIENT_ID** / **OIDC_CLIENT_SECRET** - a confidential client you register for GeoMetrikks
- **OIDC_REDIRECT_URI** - `https://geo.example.com/api/v1/auth/oidc/callback`, registered at the provider exactly as written

Then say who gets in, with **OIDC_ALLOWED_USERS** (verified email addresses or subject identifiers), **OIDC_ALLOWED_GROUPS**, or both. One of the two is mandatory and the container refuses to start without it, which beats discovering that every Google account on earth can read your traffic.

Two things to watch for on Unraid:

- The redirect URI is the *public* https address of the app, so this needs a TLS reverse proxy in front of the container, and **APP_SESSION_SECURE=true** with it. `http://<unraid-ip>:8000` will not work as a redirect URI, and an https URI with a non-Secure cookie fails validation at startup.
- Keep **APP_ADMIN_PASSWORD** set unless you want the provider to be the only way in. With both, the login page shows the provider button and the password form, which is what you want the day Authelia stops answering. Clear it and the password form disappears.

Per provider: Authelia works with the default **OIDC_SCOPES** (`openid profile email groups`) and serves groups from its userinfo endpoint, which GeoMetrikks reads. Authentik wants the issuer URL that ends in the application slug, and `openid profile email` unless you add a groups mapping. Google rejects the `groups` scope, so use `openid profile email` there and allow people by email. **OIDC_LOGOUT_IDP** only does something when the provider advertises an `end_session_endpoint`; Authelia 4.39 and Google do not, so leave it `false` for those. *Settings > Status* tells you whether the discovery document could be fetched.

The [upstream README](https://github.com/GilbN/geometrikks#openid-connect) has the rest, including an Authelia client snippet to copy.

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
