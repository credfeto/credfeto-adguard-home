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

### 5. Import existing hosts

Zone files committed to `zones/` are bind-mounted into the container at `/etc/dns/config/zones/` and loaded on startup. To migrate the legacy `hosts` file, create zones via the admin UI — the files will be written directly into `zones/` and can then be committed.

---

## Day-to-day operations

### Updating

```sh
./update
```

Pulls the latest repo changes (including any zone file updates), reloads zones if they changed, then ensures the container is running and firewall rules are current.

### Zone changes

Edit zone files in `zones/`, commit, and push. The systemd timer will pick up the changes within 15 minutes and reload zones automatically via the Technitium API. To apply immediately:

```sh
./scripts/sync-zones
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
| `scripts/sync-zones` | Pull latest; reload zones via API only if `zones/` changed |
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

## Security notes

- The container uses `network_mode: host` — it binds directly to the host's network stack. Firewall rules are the primary access control.
- DHCP ports 67/68 are explicitly rejected at the firewall even though Technitium's DHCP server is inactive by default.
- `DNS_SERVER_RECURSION=AllowOnlyForPrivateNetworks` prevents this server from acting as an open resolver.
- The admin UI uses a self-signed certificate on port 53443. Accept the browser warning or add the cert to your trust store.
