# Changelog

All notable changes to PrimeRuntime are documented in this file.
The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and the project uses [Semantic Versioning](https://semver.org/spec/v2.0.0.html) for its 0.x releases, where interfaces may still change between minor versions.

## [0.1.0] - 2026-11-09

First public release of PrimeRuntime. It wraps the PrimeShell kernel 0.5.2.

### Added

- `prt` launcher with the commands `version`, `health`, `serve`, `connect`, and the MCP server over stdio.
- Kernel executable `bin/primeshell` (kernel version 0.5.2) in guarded mode: three MCP tools, a SHA-256 pinned catalog, and startup flags as the only source of permissions.
- Default catalog with one entry: `project.read` (reads files under the project root).
- Guarded writes behind `--fabric-allow-write`, limited by a catalog path prefix and by exact expected content.
- Write journal with reservation, effect and sealed receipts. A repeated request returns the stored result. Interrupted writes are observed on retry and are not rewritten blindly.
- `prt health` with exit codes 0 (healthy), 1 (pending recovery) and 2 (configuration refused).
- `prt-sandbox`, which starts an MCP client in a bubblewrap sandbox with a read-only managed directory, hidden kernel state, a cleared environment and separate namespaces.
- Unix-socket service (`prt serve`, `prtd`) with a systemd user unit template.
- Opt-in JSON logs with `PRIMESHELL_LOG=json`.
- Installation without root, with a versioned layout for upgrade and rollback.
- Documentation: quickstart, installation, a worked guarded write, architecture, MCP integration, security boundaries, capability provider, compatibility, troubleshooting, operations, licenses and glossary.
- Release assets: SHA-256 checksums, a provenance record and a CycloneDX software bill of materials.

### Security

- Receipts are sealed with SHA-256. They are tamper-evident and are not signed.
- The client sandbox does not restrict writes outside the managed directory. This limit is documented in [SECURITY_BOUNDARIES](docs/SECURITY_BOUNDARIES.md).

### Known limitations

- No capability data, snapshot, routing table or language model is included.
- No HTTP transport and no command execution in guarded mode.
- Tested on one Linux x86_64 glibc host. Other platforms are not supported.
