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
### Fixed
### Changed
- Removed the legacy repository `hosts` file workflow; zone files in `zones/` are now the source of truth.
### Removed
### Deployment Changes
<!--
Releases that have at least been deployed to staging, BUT NOT necessarily released to live.  Changes should be moved from [Unreleased] into here as they are merged into the appropriate release branch
-->
## [0.0.0] - Project created