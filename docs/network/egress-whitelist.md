# DNS server outbound (egress) lockdown

Working notes for restricting outbound traffic from the `credfeto-dns` hosts
(`dns-01`..`dns-06`, `192.168.42.101`-`.106`) at the OPNsense firewall
(`opnsense.lan`, `192.168.10.1` / gateway seen from the DNS VLAN as
`192.168.42.1`) to an explicit allow-list, deny-by-default.

Status: draft, built from the docker-compose config, live `pfctl -s state`
on OPNsense, and live `ss` on `dns-01`. Not yet implemented as firewall
rules. Needs review before anything is enforced.

## Hosts

| Host | LAN IPv4 | LAN IPv6 | Role |
| --- | --- | --- | --- |
| dns-01..dns-06 | 192.168.42.101-106 | 2a02:8010:61d5:42::101-106 | Technitium DNS server (this repo), six nodes |
| OPNsense | 192.168.10.1 (mgmt), 192.168.42.1 (DNS VLAN gateway) | — | Firewall/router |

IPv6 addresses inferred from dns-01's `/etc/resolv.conf` nameserver list by
matching the trailing octet (`::101` <-> `.101` etc.) — worth confirming
against each node directly (`ip -6 addr`) rather than trusting this table
blindly, particularly since the file's own comment says the list is
deliberately rotated so adjacent lines are *not* the same host.

## What the DNS server is configured to reach externally

From `docker-compose.yml`:

- `DNS_SERVER_FORWARDERS=https://security.cloudflare-dns.com/dns-query`
  (DoH, HTTPS/443) — **stale/misleading**: `DNS_*` env vars only take
  effect on first-run bootstrap, not on every restart. Verified live via
  the Technitium API (`/api/settings/get`) on all six nodes: the real,
  current forwarder on every node is `https://dns.nextdns.io/be6f94/DNS-0X`
  (per-node device path), `forwarderProtocol: Https`, consistently — this
  compose line no longer reflects reality. Worth fixing in
  `docker-compose.yml` anyway so a fresh volume/redeploy doesn't silently
  revert to Cloudflare, but it's not an active problem today.
- `DNS_SERVER_BLOCK_LIST_URLS` — hourly-ish fetch of blocklists over
  HTTPS/443 from:
  - `abp.markridgwell.com` — CNAME to `credfeto.github.io` (GitHub Pages).
    Resolves within GitHub Pages' Fastly range (confirmed:
    `185.199.108-111.153` / `2606:50c0:8000-8003::153`), same as
    `raw.githubusercontent.com` below — already covered by that rule, no
    separate pin needed.
  - `raw.githubusercontent.com` (GitHub/Fastly, hagezi blocklists)
- `watchtower` — polls for image updates every hour
  (`WATCHTOWER_POLL_INTERVAL=3600`). **Confirmed not direct-to-internet:**
  Docker's daemon config (`/etc/docker/daemon.json`) sets
  `registry-mirrors: ["https://docker-cache.markridgwell.com"]`, which
  resolves to `192.168.150.250` — an internal cache/proxy host. Watchtower
  pulls through the Docker daemon/API, so it inherits this automatically;
  it only needs to reach that internal host, not Docker Hub directly.
  (Decided: keeping Watchtower — this makes that easy, no public Docker
  Hub range needed.)

From systemd units on the host (`ansible-pull.service` /
`.timer`, runs hourly):

- `ansible-pull -U https://github.com/credfeto/credfeto-setup-arch-vm.git`
  — HTTPS/443 to `github.com` (clone/pull over HTTPS, per your note).

**Audited `credfeto-setup-arch-vm` directly** (cloned at
`~/work/personal/credfeto-setup-arch-vm`). Findings:

- Package/registry mirrors (`pacman.markridgwell.com`,
  `docker-cache.markridgwell.com`, etc.) all confirmed internal-only —
  see "Other likely requirements" below.
- `roles/users/tasks/main.yml` fetches `https://github.com/credfeto.keys`
  every `ansible-pull` run, to provision `markr`'s SSH key. Same
  `github.com` destination as the git clone above, not the API.
- `roles/packages/tasks/main.yml`: installs the Chaotic AUR keyring and
  mirrorlist packages directly from **`cdn-mirror.chaotic.cx`** (real
  external CDN, not the internal proxy) — this task's `when:` condition
  stays true forever once the signing key is trusted, so it runs on
  *every* hourly pull, not just once at bootstrap. A genuinely new
  external destination this doc hadn't accounted for.
- Same role receives the Chaotic AUR signing key via
  **`hkps://keyserver.ubuntu.com`** — but only when the key isn't already
  locally trusted (guarded, with retries + a rescue block so a keyserver
  outage doesn't break the whole play) — a rare/recovery-path dependency,
  not a steady-state one.
- `roles/ssh/tasks/main.yml` configures sshd's `AuthorizedKeysCommand` to
  `curl https://keys.markridgwell.com/keys/<hostname>/%u` — confirmed
  internal (`192.168.150.250`), but note this runs on **every SSH login**
  to any of these boxes, not just during ansible-pull — a live runtime
  dependency, not a periodic one.
- `roles/telegraf/templates/telegraf.conf.j2` pushes metrics to
  `https://metrics.markridgwell.com` every 10s — this hostname had no DNS
  record at all, likely a naming mismatch against `monitoring.markridgwell.com`
  (the one that actually resolves). **Fixed:** tracked and being worked
  on in credfeto/credfeto-monitoring#28 and credfeto/credfeto-setup-arch-vm#220.
- **Now fully explains the live `20.26.156.215` traffic**: a direct TLS
  cert probe on that IP shows `CN=github.com` (plain GitHub, not the API
  as first guessed from IP-range proximity alone). That's exactly the
  `github.com` this repo calls — the git clone and/or the `credfeto.keys`
  fetch, both here. Technitium's own update-check is separately confirmed
  to never touch GitHub (source shows it redirects through
  `download.technitium.com` instead).

## DNS-to-DNS traffic (per your note)

Traffic between the six DNS nodes themselves needs to stay allowed.
Confirmed via the API on dns-03 that `clusterInitialized: false`, so
Technitium's own clustering feature is **not** in use — this must be
plain zone secondary/primary sync (AXFR/IXFR, TCP+UDP/53) instead. Zone
type per node not yet checked. Treat `192.168.42.101-106` <->
`192.168.42.101-106` on 53/tcp+udp as required between nodes until
confirmed otherwise.

For the actual rules, use a range rather than hardcoding the six
current addresses individually — `101`-`106` is just what exists today
(from `credfeto-setup-arch-vm`'s `network_nameserver_suffixes`), not a
fixed set. `192.168.42.96/28` (`.96`-`.111`) and
`2a02:8010:61d5:42::100/124` (`::100`-`::10f`) both cleanly cover the
current six with headroom for growth, without needing a firewall
rule change for every new node.

## Confirmed live external egress (OPNsense `pfctl -s state`, dns-01 `ss`)

Snapshot only — a longer log review is needed before finalising the list.

| Source | Destination | Port | Owner (from IP lookup) | Likely purpose |
| --- | --- | --- | --- | --- |
| 192.168.42.101 (dns-01) | 20.26.156.215 | 443 | AS8075 Microsoft (London) | **Fully resolved.** Direct TLS cert probe shows `CN=github.com` — plain `github.com`, not the API (the earlier "same pool as `api.github.com`" reasoning was wrong; IP adjacency in Azure's shared ranges doesn't mean same service). Also ruled out Technitium's own update-check by reading its source: `dnsServerEnableCheckForUpdate` hits `https://go.technitium.com/?id=42`, which redirects to `download.technitium.com` — never touches GitHub at all. So this is `ansible-pull`'s git clone and/or its `credfeto.keys` fetch, both confirmed real `github.com` calls in the playbook audit. |
| 2a02:8010:61d5:42::/64 (a DNS-VLAN host, privacy address) | 2a00:11c0:8:4::9 | 443 | AS42473 Anexia (London) | TLS certificate probe shows `O=NextDNS Inc.; CN=blockpage.nextdns.io`. **Confirmed the "block page" setting is disabled on the `be6f94` profile itself**, so this isn't live/current double-blocking behaviour — the connection in the state-table snapshot is most likely stale/historic, or an unrelated one-off. Not treated as an ongoing egress requirement; drop the corresponding firewalld rule below unless it recurs. |
| 192.168.42.105, .103 | 45.90.30.1 / 45.90.28.1 | 53 (udp) | NextDNS anycast | Verified via the Technitium API on all six nodes: every node's forwarder is correctly `https://dns.nextdns.io/be6f94/DNS-0X` with `forwarderProtocol: Https` — nothing is misconfigured. This plain UDP/53 traffic is almost certainly Technitium's own bootstrap resolution of the forwarder's hostname (`dns.nextdns.io` itself resolves within NextDNS's own anycast range), not the forwarder channel, which goes out separately over 443. `fcf757.dns.nextdns.io.db` is grouped with the anti-DoH-bypass block zones (same size/mtime as `dns.google.db`/`dns.opendns.com.db`/`dns.quad9.net.db`), not a second forwarder. No fix needed here. |

Not egress, but worth recording: OPNsense's state table also showed
`192.168.42.102:53 (8.8.8.8:53) <- 192.168.80.218:...` — that's a client
on a *different* VLAN (`192.168.80.218`) hardcoded to use `8.8.8.8`
(Google DNS), which OPNsense transparently redirects to dns-02 instead of
letting it actually reach Google. No traffic leaves the network here;
it's an existing inbound NAT redirect on the firewall, unrelated to what
the DNS servers themselves need outbound.

Also observed: every DNS node has an established TCP session to
`192.168.90.254:1433` (MSSQL) — that's cross-VLAN, not internet egress, but
still needs an explicit firewall allow if OPNsense enforces inter-VLAN
rules too. **Confirmed via the API**: Technitium's "Query Logs (SQL
Server)" app is installed (`/api/apps/list` on dns-01) — this is it, not
an unrelated service. Still to confirm: whether this should keep running
on every node vs. consolidating, and whether the SQL Server itself is
locked down to only accept from the DNS VLAN.

## Other likely requirements not yet seen live

- NTP (`systemd-timesync1.service` is enabled on dns-01) — currently
  unconfigured, so it falls back to the `arch.pool.ntp.org` DNS pool
  (dynamic, not egress-lockdown-friendly). **Decision: repoint at
  OPNsense's own NTP service** (`192.168.42.1`, the DNS VLAN gateway —
  confirm OPNsense is actually serving NTP there, it usually does by
  default) instead of the public pool. Set `NTP=192.168.42.1` in
  `/etc/systemd/timesyncd.conf` (or a `timesyncd.conf.d/` drop-in) on each
  node — turns this into a fixed private-IP rule instead of a dynamic
  pool, so it moves out of the TBD list below.
- `firewalld` is also running locally on dns-01 (host-level firewall, in
  front of OPNsense) — its own egress rules should be checked/aligned too,
  otherwise the two could disagree.
- **Confirmed:** `pacman.markridgwell.com`, `aur.markridgwell.com`,
  `dotnet.markridgwell.com`, `npm.markridgwell.com`,
  `docker-cache.markridgwell.com`, `docker-registry.markridgwell.com` and
  `api-nuget.markridgwell.com` all resolve to the same internal host,
  `192.168.150.250` — a single reverse-proxy/cache fronting every package
  source. Egress for all of these is internal-only from the DNS boxes'
  point of view; no external whitelisting needed for package installs.
  Still need to confirm `credfeto-setup-arch-vm`'s playbook actually uses
  these internal names rather than the real upstreams directly (see open
  question below), but the DNS-server-side answer looks settled.

## Open questions before drafting actual firewall rules

1. ~~Should `abp.markridgwell.com` be locked to a specific IP?~~
   **Resolved:** it's a CNAME to `credfeto.github.io` (GitHub Pages),
   resolving in the same Fastly range as `raw.githubusercontent.com` —
   already covered by that rule, no separate pin needed.
2. ~~Is NextDNS forwarding consistent/correct across nodes?~~ **Resolved:**
   confirmed via the API on all six nodes — every node correctly forwards
   to `https://dns.nextdns.io/be6f94/DNS-0X` over DoH. No action needed.
3. ~~What is the 8.8.8.8 NAT redirect on dns-02?~~ **Resolved:** it's the
   reverse of what it first looked like — an existing OPNsense inbound
   redirect that catches a client on another VLAN hardcoded to `8.8.8.8`
   and sends it to dns-02 instead, so it never reaches Google. Not egress,
   no action needed.
4. ~~What does `credfeto-setup-arch-vm` reach?~~ **Resolved — audited
   directly.** Package mirrors are internal-only (`192.168.150.250`), as
   expected. Three real external destinations found that weren't
   previously known: `cdn-mirror.chaotic.cx` (Chaotic AUR, runs every
   hourly pull), `hkps://keyserver.ubuntu.com` (rare, only on an
   untrusted key), and `github.com` itself for `credfeto.keys` (in
   addition to the git clone already listed). Also surfaced: `keys.markridgwell.com`
   is a live per-SSH-login dependency, not just an ansible-pull one; and
   `metrics.markridgwell.com`'s missing DNS record — being fixed in
   credfeto/credfeto-monitoring#28 and credfeto/credfeto-setup-arch-vm#220.
5. ~~Is Watchtower still wanted?~~ **Decided: yes, keep it — and resolved:**
   it pulls through the Docker daemon's `registry-mirrors` setting
   (`docker-cache.markridgwell.com` -> `192.168.150.250`, internal), so it
   never needs direct Docker Hub egress. No public range needed after all.
6. ~~What is `192.168.90.254:1433` (MSSQL) for?~~ **Resolved:** Technitium's
   own "Query Logs (SQL Server)" app, confirmed installed via the API.
   Still open: does it need to stay reachable from every node, or should
   logging be consolidated?
7. ~~What exactly calls `api.github.com`?~~ **Resolved, and the earlier
   "GitHub API" attribution was wrong.** Direct TLS cert probe on
   `20.26.156.215` shows `CN=github.com`, not the API. Also confirmed
   Technitium's update-check never touches GitHub at all (its source
   shows `https://go.technitium.com/?id=42`, which redirects to
   `download.technitium.com`). It's `ansible-pull`'s git clone and/or its
   `credfeto.keys` fetch — both real `github.com` calls already known
   from the playbook audit.
8. ~~Is NextDNS's own blocking active alongside Technitium's?~~
   **Resolved:** confirmed the "block page" setting is disabled on the
   `be6f94` profile. The block-page IP seen in the state-table snapshot
   wasn't current/ongoing behaviour.

## Proxmox firewall: tried, ruled out

Before the `firewalld` approach below, tried enforcing egress at the
hypervisor level instead, via Proxmox's own per-VM firewall on DNS-06
(VMID 112) — enforced outside the guest, so a compromised VM can't
disable it. **Ruled out**: Proxmox's firewall framework has a hardcoded,
always-on rule (`-A PVEFW-Drop -p udp --sport 53 -j DROP`, confirmed via
`pve-firewall compile` and the official docs at
`pve.proxmox.com/wiki/Firewall`) that drops any UDP packet with *source*
port 53 — exactly what Technitium's own outbound recursive/bootstrap
queries use. It's independent of the VM's own `policy_in`/`policy_out`
and has no exposed exemption. Applying it broke DNS-06's resolution
entirely (real outage, since reverted — see `dnyw4l3n13/proxmox`'s
`NOTES.md`, 2026-09-01, for the full incident writeup). Proxmox's
firewall may be fine for other, non-DNS VMs on that cluster; it's a dead
end specifically for restricting these DNS servers' own egress.

## Suggested `firewalld` rules (host-level, draft — not applied)

dns-01 runs `firewalld 2.5.1` locally, active zone `public` (default),
plus a `docker` zone for container bridges. It already ships firewalld's
stock `gateway-*` policy objects (`gateway-lan-to-world`,
`gateway-world-to-HOST`, `gateway-lan-to-HOST`, ...) but **all of them are
disabled** — they're firewalld's built-in templates, not something this
host's ansible role turned on, so nothing currently restricts egress at
the host level. This is separate from — and in addition to — whatever
gets enforced at OPNsense; the two should agree, not be the only line of
defence on their own.

Egress filtering in firewalld needs a **policy object** (`--policy`,
ingress zone -> egress zone), not the ingress-only zone/rich-rule
mechanism used for opening inbound ports. Suggested shape: one policy per
host, `ingress-zone=HOST`, `egress-zone=ANY`, `target=DROP`, with explicit
`accept` rich rules layered on top for every confirmed destination —
mirroring the confirmed/TBD split above.

```bash
#!/usr/bin/env bash
set -euo pipefail

POLICY=dns-egress

# Deny-by-default outbound from this host, everything else explicit below.
firewall-cmd --permanent --new-policy "${POLICY}" 2>/dev/null || true
firewall-cmd --permanent --policy "${POLICY}" --set-target DROP
firewall-cmd --permanent --policy "${POLICY}" --add-ingress-zone HOST
firewall-cmd --permanent --policy "${POLICY}" --add-egress-zone ANY

allow_egress_ipv4() {
    local dest="$1" port="$2" protocol="$3"
    firewall-cmd --permanent --policy "${POLICY}" \
        --add-rich-rule="rule family='ipv4' destination address='${dest}' port port='${port}' protocol='${protocol}' accept"
}

allow_egress_ipv6() {
    local dest="$1" port="$2" protocol="$3"
    firewall-cmd --permanent --policy "${POLICY}" \
        --add-rich-rule="rule family='ipv6' destination address='${dest}' port port='${port}' protocol='${protocol}' accept"
}

# --- Confirmed: DNS-to-DNS sync between the six nodes (53/tcp+udp) ---
# Range, not individual /32s per node - .101-.106 is just what exists
# today, not a fixed set; this covers headroom for future nodes too.
allow_egress_ipv4 "192.168.42.96/28" 53 tcp
allow_egress_ipv4 "192.168.42.96/28" 53 udp
allow_egress_ipv6 "2a02:8010:61d5:42::100/124" 53 tcp
allow_egress_ipv6 "2a02:8010:61d5:42::100/124" 53 udp

# --- Confirmed: NextDNS (be6f94) is the sole forwarder, DoH/443 ---
# Verified live via the Technitium API on all six nodes. Anycast range
# 45.90.28.0/24 / 45.90.30.0/24 covers both the DoH endpoint (443) and the
# plain-UDP/53 bootstrap lookup of dns.nextdns.io itself - allow both.
allow_egress_ipv4 "45.90.28.0/24" 443 tcp
allow_egress_ipv4 "45.90.30.0/24" 443 tcp
allow_egress_ipv4 "45.90.28.0/24" 53 udp
allow_egress_ipv4 "45.90.30.0/24" 53 udp

# NextDNS's block-page server (blockpage.nextdns.io) deliberately NOT
# included: "block page" is confirmed disabled on the be6f94 profile, so
# this isn't an ongoing requirement - the one connection seen live was
# stale/unrelated, not current behaviour.

# Cloudflare's security DoH forwarder is NOT wanted (superseded by
# NextDNS, and the compose env var that named it is stale anyway) - left
# out entirely.

# --- Confirmed: raw.githubusercontent.com AND abp.markridgwell.com ---
# GitHub Pages/raw-content's Fastly range - covers both blocklist URLs,
# abp.markridgwell.com being a CNAME to credfeto.github.io.
allow_egress_ipv4 "185.199.108.0/22" 443 tcp
allow_egress_ipv6 "2606:50c0::/32" 443 tcp

# --- Confirmed (decided): NTP via OPNsense itself, not the public pool ---
# Requires each node's timesyncd.conf to set NTP=192.168.42.1 first -
# confirm OPNsense is actually serving NTP on the DNS VLAN before relying on this.
allow_egress_ipv4 "192.168.42.1/32" 123 udp

# --- Confirmed: internal package/registry cache proxy (192.168.150.250) ---
# Fronts docker-cache, docker-registry, pacman, aur, dotnet, npm and
# api-nuget (all confirmed to resolve here) - covers Watchtower's pulls
# (via Docker's registry-mirrors setting) and, once confirmed, whatever
# credfeto-setup-arch-vm restores. Cross-VLAN, not internet egress, but
# still needs an explicit allow if OPNsense enforces inter-VLAN rules.
allow_egress_ipv4 "192.168.150.250/32" 443 tcp

# --- Confirmed: Technitium's own MSSQL query-logging app ---
allow_egress_ipv4 "192.168.90.254/32" 1433 tcp

# --- TBD, needs confirming before enabling (see "Open questions") ---
# github.com (ansible-pull's git clone and credfeto.keys fetch -
#                                confirmed via TLS cert probe, not
#                                Technitium, which never touches GitHub) -
#                                GitHub's IP ranges are published but
#                                change; needs an ipset +
#                                refresh job, not a static rule
# cdn-mirror.chaotic.cx         - Chaotic AUR CDN, runs every hourly pull
#                                  once the signing key is trusted; same
#                                  ipset-or-published-range problem
# hkps://keyserver.ubuntu.com   - GPG keyserver, only hit when the
#                                  Chaotic AUR key isn't already trusted -
#                                  rare/recovery-path, lower priority

firewall-cmd --reload
```

Notes on this draft:

- Domain-backed destinations still without a stable range (`github.com`
  for `ansible-pull`, Docker Hub) can't be pinned to a static IP/CIDR
  safely — CDNs rotate. Options once confirmed: a periodic job that
  resolves the name and rewrites an `ipset`/policy rich-rules, or accept
  the provider's documented full CIDR
  range if one exists and is stable (GitHub publishes one via
  `api.github.com/meta`).
- This would need to be applied per-node (`dns-01`..`dns-06`), and ideally
  added to the `credfeto-setup-arch-vm` ansible role rather than run ad hoc
  over SSH — otherwise the next `ansible-pull` run (hourly) has no reason
  to know about it, but also can't be relied on to *not* clobber it either
  without checking that role first.
- Test on one node with a fast way to revert (console/OPNsense access, not
  just SSH — an egress mistake can cut the node off from the DoH
  forwarder it needs to resolve anything with) before rolling to all six.

## Next steps

- Confirm OPNsense is serving NTP on `192.168.42.1`, then set
  `NTP=192.168.42.1` in `timesyncd.conf` on each node (via the ansible
  role, not by hand) — removes NTP from the TBD list.
- Confirm/resolve the remaining open questions above.
- Pull a longer OPNsense firewall log sample (not just the current state
  table) to catch anything that only happens occasionally (e.g. daily/
  weekly jobs) rather than what happened to be live during this snapshot.
- Read `credfeto-setup-arch-vm`'s playbook to get its full egress
  footprint.
- Once the list is believed complete, draft explicit OPNsense alias +
  rule set (deny-by-default outbound from the DNS VLAN, allow only the
  confirmed destinations/ports) — as a separate, reviewed change, not
  folded into this notes pass.
