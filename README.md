# GeoMetrikks Unraid Templates

Unraid Community Apps templates for [GeoMetrikks](https://github.com/GilbN/geometrikks) - a real-time geolocation analytics tool that tails nginx access logs, does GeoIP lookups, and visualizes traffic on a live interactive map.

This repository ships two Docker templates:

| Template | What it is |
|---|---|
| [`geometrikks-timescaledb`](templates/geometrikks-timescaledb.xml) | TimescaleDB + PostGIS database - install this first |
| [`geometrikks`](templates/geometrikks.xml) | The GeoMetrikks app and web UI |

## Setup

### 1. Install the database

Add `geometrikks-timescaledb` from Community Apps. Set a **POSTGRES_PASSWORD** - you'll reuse this exact value in step 2. Leave POSTGRES_USER (`geouser`) and POSTGRES_DB (`geometrikks`) at their defaults unless you have a reason to change them.

The DB Port config (default `5432`) is exposed to your LAN so the app container can reach it.

### 2. Install the app

Add `geometrikks` from Community Apps and fill in:

- **DB_HOST** - your Unraid server's IP address (e.g. `192.168.1.50`)
- **DB_PASSWORD** - the *same* password you set for `geometrikks-timescaledb` in step 1
- **APP_ADMIN_USER** / **APP_ADMIN_PASSWORD** - your web UI login or **APP_AUTH_DISABLED=true**
- **Nginx Logs** path - point this at wherever your reverse proxy writes its access logs (defaults to a SWAG-style path; change it for Nginx Proxy Manager, Caddy, or whatever you actually run)

Leave DB_PORT/DB_USER/DB_DATABASE at their defaults unless you changed the matching values in step 1.

### 3. Get a free MaxMind GeoLite2 key

GeoMetrikks needs MaxMind's free GeoLite2 database for GeoIP lookups:

1. Sign up at <https://www.maxmind.com/en/geolite2/signup>
2. Create a license key under your account
3. Set **MAXMINDDB_USER_ID** and **MAXMINDDB_LICENSE_KEY** on the `geometrikks` template

Without these, GeoMetrikks runs in degraded mode (no GeoIP lookups) until you add them.

### 4. Open the web UI

`http://<your-unraid-ip>:8000/` - log in with the admin credentials you set in step 2.

## Notes

- Full environment variable reference: <https://github.com/GilbN/geometrikks/blob/main/docs/configuration.md>. Most of it is exposed as "advanced" settings on the `geometrikks` template - expand "Show more settings" when adding the container to see database pool tuning, log parser tuning, analytics retention, and map settings.
- GeoMetrikks itself: <https://github.com/GilbN/geometrikks>
