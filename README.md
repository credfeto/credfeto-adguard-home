# credfeto-dns

Technitium DNS Server running in Docker with host networking, managed block lists, and version-controlled zone files.

## Prerequisites

- Docker with Compose plugin
- `firewall-cmd` (firewalld)
- `systemd`
- Port 53 available on the host — disable the systemd-resolved stub listener if present:

  ```sh
  sudo systemctl disable --now systemd-resolved
  ```

## First-time setup

### 1. Configure the admin password

```sh
cp .env.example .env
# Edit .env and set a strong DNS_ADMIN_PASSWORD
```

### 2. Install the zone sync timer

```sh
cp scripts/.env.example scripts/.env
# Leave DNS_API_TOKEN blank for now — fill it in after step 4
./install
```

This installs a systemd timer that runs `scripts/sync-zones` every 15 minutes.

### 3. Start the server and apply firewall rules

```sh
./update
```

This creates `/data/technitium/config` and `/data/technitium/logs`, starts the container, and ensures all firewall rules are in place.

### 4. Generate an API token

Open the admin UI at `https://<host-ip>:53443`, log in with the password from `.env`, then go to **Settings → API Keys** and create a token. Paste it into `scripts/.env`:

```sh
DNS_API_TOKEN=your-token-here
```

### 5. Import zones

Paste the API token into `scripts/.env`, then run `./update` again. At the end of `update`, `sync-zones --all` is called automatically and imports every `.db` file in `zones/` into Technitium.

The imported zones are persisted by Technitium in the same directory (as `.zone` files, which are `.gitignore`d).

---

## Day-to-day operations

### Updating

```sh
./update
```

Pulls the latest repo changes, ensures the container is running, applies firewall rules, then imports all zones via `sync-zones --all`.

### Zone changes

Edit zone files in `zones/`, commit, and push. The systemd timer will pick up the changes within 15 minutes and import the updated zones via the Technitium API. To apply immediately:

```sh
./scripts/sync-zones
```

To import all zones regardless of git state (e.g. after a fresh deploy):

```sh
./scripts/sync-zones --all
```

### Restarting the container

```sh
docker compose restart dns-server
```

---

## Scripts

| Script | Purpose |
| --- | --- |
| `install` | Installs and enables the systemd timer for zone sync |
| `update` | Full update: pull, start container, apply firewall rules |
| `scripts/sync-zones` | Pull latest; import changed zones via API. `--all` imports every zone. |
| `reset` | Stop and remove containers, then run `update` |

## Files

| Path | Purpose |
| --- | --- |
| `.env` | Admin password (not committed) |
| `.env.example` | Template for `.env` |
| `scripts/.env` | API token for zone sync (not committed) |
| `scripts/.env.example` | Template for `scripts/.env` |
| `zones/` | Version-controlled zone files, mounted into the container |
| `docker-compose.yml` | Container definition |

## Data directories

| Host path | Container path | Contents |
| --- | --- | --- |
| `/data/technitium/config` | `/etc/dns` | Technitium configuration (persists across restarts) |
| `/data/technitium/logs` | `/var/log/technitium/dns` | DNS query logs |
| `./zones` | `/etc/dns/config/zones` | Zone files (version-controlled) |

## Zone file layout

Zones are maintained as BIND-format `.db` files in `zones/`. Technitium writes its internal `.zone` files alongside them (`.gitignore`d).

| File | Zone type | Purpose |
| --- | --- | --- |
| `lan.db` | Primary | `.lan` hostnames (Proxmox nodes, network devices) |
| `dns.lan.db` | Primary | DNS server hostnames |
| `proxy.markridgwell.com.db` | Primary | Canonical A record → `192.168.150.250` |
| `<service>.markridgwell.com.db` | Override | Per-service CNAME → `proxy.markridgwell.com` |
| `<resolver>.db` | Override | DoH resolver IP overrides (Cloudflare, Google, Quad9) |

**Split-brain DNS:** Each `<service>.markridgwell.com` is its own override zone (not a full `markridgwell.com` zone), so Cloudflare remains authoritative for all other names under `markridgwell.com`.

**Adding a new internal service:**

```sh
# Point a new subdomain at the proxy:
cp zones/home.markridgwell.com.db zones/newservice.markridgwell.com.db
# Edit the $ORIGIN line, commit, push — timer will import within 15 min.
```

**Changing the proxy IP:** Edit `proxy.markridgwell.com.db` — all CNAME-based services follow automatically.

## Security notes

- The container uses `network_mode: host` — it binds directly to the host's network stack. Firewall rules are the primary access control.
- DHCP ports 67/68 are explicitly rejected at the firewall even though Technitium's DHCP server is inactive by default.
- `DNS_SERVER_RECURSION=AllowOnlyForPrivateNetworks` prevents this server from acting as an open resolver.
- The admin UI uses a self-signed certificate on port 53443. Accept the browser warning or add the cert to your trust store.
