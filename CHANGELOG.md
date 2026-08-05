# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

<!--
Please ADD ALL Changes to the UNRELEASED SECTION and not a specific release
-->

## [Unreleased]
### Security
### Added
- Added .ai-instructions and ai/local/index.md from cs-template standard
- Added `dev-upgrader-01.lan` (`192.168.150.20`) to `zones/lan.db`.
- Added `dev-upgrader-02.lan` (`192.168.150.21`) to `zones/lan.db`.
- Added `linux-cache-01.lan` (`192.168.150.200`), `linux-cache-02.lan` (`192.168.150.201`), `defi.lan` (`192.168.150.12`), and `arch-vm-test.lan` (`192.168.150.247`) to `zones/lan.db`.
- DNS record for keys.markridgwell.com pointing to proxy on 192.168.150.250
- DNS record for cctv.markridgwell.com pointing to proxy on 192.168.150.250
- Support IPv6 (private ranges and 2a02:8010:61d5::/64) in firewall rules
- Added `homepage.lan` (`192.168.150.150`) to `zones/lan.db`.
- Added `gickup.lan` (`192.168.150.13`) to `zones/lan.db`.
- Added `dns-06.lan` / `dns-06.dns.lan` (`192.168.42.106`, `2a02:8010:61d5:42::106`) to `zones/lan.db` and `zones/dns.lan.db`.
- Added `audio-bookshelf.lan` (`192.168.150.16`) to `zones/lan.db`.
- Added `dev-cache-01.lan` (`192.168.150.100`) and `dev-cache-02.lan` (`192.168.150.101`) to `zones/lan.db`.
- Added `development.lan` (`192.168.150.176`) to `zones/lan.db`.
- Added `immich.lan` (`192.168.150.154`) to `zones/lan.db`.
- Added `proxmox-05.lan` (`192.168.86.15`) and `proxmox-06.lan` (`192.168.86.16`) to `zones/lan.db`.
- Added `firewall.lan` (`192.168.10.1`) to `zones/lan.db`.
- Added new `network.lan` zone (`zones/network.lan.db`) with `stairs-01` (`192.168.10.2`) and `bedroom` (`192.168.10.5`); also added flat `stairs-01.lan` and `bedroom.lan` aliases to `zones/lan.db`.
- Added IPv6 AAAA records for public DNS resolvers: 1dot1dot1dot1.cloudflare-dns.com, mozilla.cloudflare-dns.com, security.cloudflare-dns.com, dns.google, and dns.quad9.net.
- Added public DNS resolver zones for Cloudflare Family (family.cloudflare-dns.com), AdGuard DNS (dns.adguard-dns.com), and OpenDNS (dns.opendns.com), each with IPv4 A and IPv6 AAAA records.
- Added dns-06.markridgwell.com zone (192.168.150.250).
### Fixed
- Fixed typo in `zones/lan.db`: `docker-tegistry` renamed to `docker-registry` (`192.168.150.202`).
- Corrected `monitoring.lan` IP in `zones/lan.db` from `192.168.150.135` to `192.168.150.134`.
### Changed
- Removed the legacy repository `hosts` file workflow; zone files in `zones/` are now the source of truth.
- Widened the allowed IPv6 firewall network from 2a02:8010:61d5::/64 to 2a02:8010:61d5::/48 to cover all subnets, not just subnet 0
- Switched Technitium recursion ACL from AllowOnlyForPrivateNetworks to an explicit network ACL including 2a02:8010:61d5::/48, since public IPv6 addresses are never classified as private
- Renumbered dns-01 through dns-04 from 192.168.42.251-254 to 192.168.42.101-104, added dns-05 (192.168.42.105), and added AAAA records (2a02:8010:61d5:42::101-105) for all five in lan.db and dns.lan.db
### Deprecated
### Removed
### Deployment Changes
<!--
Releases that have at least been deployed to staging, BUT NOT necessarily released to live.  Changes should be moved from [Unreleased] into here as they are merged into the appropriate release branch
-->
## [0.0.0] - Project created