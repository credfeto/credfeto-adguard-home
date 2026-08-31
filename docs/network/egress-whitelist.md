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
  (DoH, HTTPS/443) — recursive resolution forwarder. **Conflicts with your
  decision below that the only sanctioned forwarder is NextDNS
  `dns.nextdns.io/be6f94`** — this line in `docker-compose.yml` needs
  changing to match once you're ready for that (not done as part of this
  notes pass).
- `DNS_SERVER_BLOCK_LIST_URLS` — hourly-ish fetch of blocklists over
  HTTPS/443 from:
  - `abp.markridgwell.com` (own domain — confirm hosting IP before locking
    down to a single address, it may move)
  - `raw.githubusercontent.com` (GitHub/Fastly, hagezi blocklists)
- `watchtower` — polls for image updates every hour
  (`WATCHTOWER_POLL_INTERVAL=3600`) against the container registry the
  image was pulled from: `docker.io` / `registry-1.docker.io` /
  `index.docker.io` (HTTPS/443).

From systemd units on the host (`ansible-pull.service` /
`.timer`, runs hourly):

- `ansible-pull -U https://github.com/credfeto/credfeto-setup-arch-vm.git`
  — HTTPS/443 to `github.com` (clone/pull over HTTPS, per your note).
- Whatever that playbook itself pulls (package mirrors, etc.) — **not yet
  audited**, needs a read of `credfeto-setup-arch-vm` to know its full
  egress footprint (pacman/AUR mirrors, dotnet/nuget, npm — see zone list
  below, several look like they're meant to go through internal caching
  proxies rather than out to the internet directly).

## DNS-to-DNS traffic (per your note)

Traffic between the six DNS nodes themselves needs to stay allowed —
likely Technitium's zone secondary/primary sync (AXFR/IXFR, TCP+UDP/53)
and/or its clustering feature. **Not yet confirmed which** — need to check
each zone's type (primary/secondary/stub) and Technitium's cluster/sync
settings via its web UI or API to know exact ports/direction. Treat
`192.168.42.101-106` <-> `192.168.42.101-106` on 53/tcp+udp as required
between nodes until confirmed otherwise.

## Confirmed live external egress (OPNsense `pfctl -s state`, dns-01 `ss`)

Snapshot only — a longer log review is needed before finalising the list.

| Source | Destination | Port | Owner (from IP lookup) | Likely purpose |
| --- | --- | --- | --- | --- |
| 192.168.42.101 (dns-01) | 20.26.156.215 | 443 | AS8075 Microsoft (London) | Guessing NuGet.org (`api.nuget.org` is Azure-hosted) or another MS-hosted endpoint pulled in by `ansible-pull`'s playbook — **needs confirming**, not obviously required by this repo alone |
| 2a02:8010:61d5:42::/64 (a DNS-VLAN host, privacy address) | 2a00:11c0:8:4::9 | 443 | AS42473 Anexia (London) | Likely `abp.markridgwell.com` (your own blocklist host) — **confirm** |
| 192.168.42.105, .103 | 45.90.30.1 / 45.90.28.1 | 53 (udp) | NextDNS anycast | This is your one sanctioned forwarder (`dns.nextdns.io/be6f94`) — but it's going out as **plain UDP/53, not DoH/443**. `fcf757.dns.nextdns.io.db` is grouped with the anti-DoH-bypass block zones (same size/mtime as `dns.google.db`/`dns.opendns.com.db`/`dns.quad9.net.db`), not a second forwarder. **Two things to fix, not just confirm: (a) get all six nodes forwarding to `be6f94` instead of only .103/.105 with .101 on Cloudflare, (b) get the protocol actually set to DoH so it's HTTPS/443, matching the `https://dns.nextdns.io/be6f94/...` config rather than falling back to plain DNS/53.** |
| 192.168.42.102 | (NAT'd to 8.8.8.8) | 53 | Google | Saw a NAT translation annotation on OPNsense implying outbound port-53 on this host gets redirected — **check OPNsense NAT rules directly**, don't assume this is intentional egress to Google |

Also observed: every DNS node has an established TCP session to
`192.168.90.254:1433` (MSSQL) — that's cross-VLAN, not internet egress, but
still needs an explicit firewall allow if OPNsense enforces inter-VLAN
rules too. Purpose not yet confirmed (Technitium doesn't use SQL Server
natively — could be logging/reporting or an unrelated service on the same
host).

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
- `pacman`/AUR/dotnet/npm/docker-registry zone entries in `zones/` (e.g.
  `pacman.markridgwell.com`, `aur.markridgwell.com`, `dotnet.markridgwell.com`,
  `npm.markridgwell.com`, `docker-cache.markridgwell.com`) suggest package
  installs are meant to go through internal caching proxies rather than
  external mirrors directly — if so, egress for those should be internal
  only and NOT need whitelisting externally. Needs confirming against
  what `ansible-pull`'s playbook actually configures on these hosts.

## Open questions before drafting actual firewall rules

1. Should the block-list URL (`abp.markridgwell.com`) be locked to a
   specific IP, or does it move (e.g. behind a CDN/dynamic DNS)?
2. **Decided:** the only sanctioned forwarder is NextDNS
   `https://dns.nextdns.io/be6f94/...` (DoH) — no other forwarder,
   including the Cloudflare one currently in `docker-compose.yml`. Two
   follow-ups this creates: get all six nodes onto it consistently
   (currently only .103/.105 use it, .101 uses Cloudflare, .102/.104/.106
   not yet checked), and get it actually running as DoH/443 rather than
   the plain UDP/53 seen live for .103/.105.
3. What is the 8.8.8.8 NAT redirect on dns-02 (`.102`) — an existing
   OPNsense rule forcing plain DNS out to Google? Intentional?
4. What does `credfeto-setup-arch-vm`'s `site.yml` actually reach
   (mirrors, package sources)? Needs its own audit.
5. Is Watchtower still wanted at all, given it needs open egress to a
   container registry on every node, every hour? (Not asking you to
   decide now — just flagging it drives one of the wider allow rules.)
6. What is `192.168.90.254:1433` (MSSQL) for, and does it need to keep
   being reachable from every DNS node?

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
for ip4 in 192.168.42.101 192.168.42.102 192.168.42.103 192.168.42.104 192.168.42.105 192.168.42.106; do
    allow_egress_ipv4 "${ip4}/32" 53 tcp
    allow_egress_ipv4 "${ip4}/32" 53 udp
done
for ip6 in 2a02:8010:61d5:42::101 2a02:8010:61d5:42::102 2a02:8010:61d5:42::103 \
           2a02:8010:61d5:42::104 2a02:8010:61d5:42::105 2a02:8010:61d5:42::106; do
    allow_egress_ipv6 "${ip6}/128" 53 tcp
    allow_egress_ipv6 "${ip6}/128" 53 udp
done

# --- Decided: NextDNS (be6f94) is the only sanctioned forwarder, DoH/443 ---
# Anycast addresses seen live are 45.90.28.1 / 45.90.30.1, but those were
# observed on plain UDP/53, not confirmed as NextDNS's actual DoH endpoint
# IPs for dns.nextdns.io - re-verify before relying on this once the
# forwarder is switched to DoH on every node.
allow_egress_ipv4 "45.90.28.0/24" 443 tcp
allow_egress_ipv4 "45.90.30.0/24" 443 tcp

# Cloudflare's security DoH forwarder is NOT wanted (see "Open questions")
# - left out entirely, do not add back once docker-compose.yml is fixed.

# --- Confirmed: raw.githubusercontent.com blocklists (443/tcp) ---
# GitHub's documented Fastly range for raw content.
allow_egress_ipv4 "185.199.108.0/22" 443 tcp

# --- Confirmed (decided): NTP via OPNsense itself, not the public pool ---
# Requires each node's timesyncd.conf to set NTP=192.168.42.1 first -
# confirm OPNsense is actually serving NTP on the DNS VLAN before relying on this.
allow_egress_ipv4 "192.168.42.1/32" 123 udp

# --- TBD, needs confirming before enabling (see "Open questions") ---
# abp.markridgwell.com          - own domain, pin its real IP first
# github.com (ansible-pull)     - GitHub's IP ranges are published but
#                                  change; needs an ipset + refresh job,
#                                  not a static rule
# Docker Hub (watchtower)       - wide/CDN-fronted; consider dropping
#                                  watchtower instead of whitelisting it
# 20.26.156.215 (Microsoft)     - purpose not yet identified
# 2a00:11c0:8:4::9 (Anexia)     - purpose not yet identified
# 192.168.90.254:1433 (MSSQL)   - purpose not yet identified

firewall-cmd --reload
```

Notes on this draft:

- Domain-backed destinations (GitHub, NTP pool, Docker Hub, `abp.markridgwell.com`)
  can't be pinned to a static IP/CIDR safely — CDNs and pools rotate. Options
  once confirmed: a periodic job that resolves the name and rewrites an
  `ipset`/policy rich-rules, or accept the provider's documented full CIDR
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
