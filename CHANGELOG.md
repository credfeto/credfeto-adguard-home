# Changelog

All notable changes to this project are documented in this file.

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
### Fixed
### Changed
- Removed the legacy repository `hosts` file workflow; zone files in `zones/` are now the source of truth.
- Widened the allowed IPv6 firewall network from 2a02:8010:61d5::/64 to 2a02:8010:61d5::/48 to cover all subnets, not just subnet 0
- Switched Technitium recursion ACL from AllowOnlyForPrivateNetworks to an explicit network ACL including 2a02:8010:61d5::/48, since public IPv6 addresses are never classified as private
- Renumbered dns-01 through dns-04 from 192.168.42.251-254 to 192.168.42.101-104, added dns-05 (192.168.42.105), and added AAAA records (2a02:8010:61d5:42::101-105) for all five in lan.db and dns.lan.db
### Removed
### Deployment Changes
<!--
Releases that have at least been deployed to staging, BUT NOT necessarily released to live.  Changes should be moved from [Unreleased] into here as they are merged into the appropriate release branch
-->
## [0.0.0] - Project created