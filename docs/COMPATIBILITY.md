# Compatibility and limitations

## Supported platform

| Item | Supported in 0.1.0 | Notes |
| --- | --- | --- |
| Operating system | Linux | Other systems are not supported |
| CPU architecture | x86_64 | No arm64 build |
| C library | glibc | musl is not tested |
| Privileges | Your user account | Root and multi-user installs are not tested |
| Sandbox | bubblewrap at `/usr/bin/bwrap` | Needed only for `prt-sandbox` |
| Test host | Aeon (VERSION_ID 20261022), kernel 7.3.0-1-default, x86_64 | Only this host is covered by the release tests |

Windows, macOS and WSL are not supported.

## MCP clients

| Client | Status in 0.1.0 |
| --- | --- |
| Claude Code (command-line) | Target client. Registration with `claude mcp add` is documented in [MCP_INTEGRATION.md](MCP_INTEGRATION.md). Release test version: 2.1.311. |
| Other MCP clients over stdio | Expected to work for the three guarded tools. Not tested. |
| Clients over HTTP | Not available. The HTTP transport is not built. |

Protocol: the server answers `initialize` with the protocol version the client sent. The release tests used `2025-06-18` and `2024-11-05`, and the server echoed each one. Other versions were not tested.

## Versions

- The package version is `0.1.0`. The kernel inside it reports version `0.5.2`. `prt version` prints both.
- The catalog format is `schema_version` 1.
- Receipts are `receipt_version` 2.
- The release is pre-1.0. Command names, field names and refusal codes may change in a later 0.x release, and the changelog will say so.

## Not in this release

| Item | Status |
| --- | --- |
| Language model, local or remote | Not included. The kernel has no model client. |
| Routing and capability data | Not included. Needs an external snapshot that is not shipped. |
| HTTP transport | Not built |
| Command execution in guarded mode | Not offered |
| Wider tool set in the bare kernel | The kernel binary, started directly without the launcher's flags, serves 33 tools, including command execution, HTTP GET and file writes. Each is enabled only by its startup flag (`--allow-command`, `--allow-http`, `--allow-write`). The launcher does not offer them. |
| Protected process launches | Refused with `HOST_INTEGRATION_UNAVAILABLE` |
| Network access by the kernel | Through the launcher: none. The kernel binary performs an HTTP GET only when started directly with `--allow-http`, which runs the named `curl` as a child process. Remote egress was not tested. |
| Source code | Not published with this release |
| Signed packages or receipts | Not included. Packages are checked with SHA-256, receipts are SHA-256 sealed. |
| Container image | Not provided |
| Telemetry | None found. The binary contains no telemetry endpoint strings (static check). Logs go to stderr only. |

## Performance

Resource use (time, memory, disk) for 0.1.0 was not measured as part of this release. No performance figure is claimed.

## Limits in the code

| Limit | Value |
| --- | --- |
| Frame size on the socket | 1 MiB |
| Clients on the socket | A 17th concurrent client evicts the oldest idle client |
| Catalog file size | 64 KiB (a 66079-byte catalog was refused with `FABRIC_CATALOG_LIMIT`; the exact boundary was not isolated) |
| Request ID length | 1 to 64 characters |
| Write size (kernel write path) | 256 KiB (262144 bytes committed; 262145 bytes refused) |

The limits above were observed in the 0.1.0 test. Other limits in the source were not observed and are not listed.

## Protected process launches

Protected process launches (not part of the shipped catalog) need host integration, which this public build does not contain. The kernel refuses them with the generic code `HOST_INTEGRATION_UNAVAILABLE`, and the `--protected-boundary` option is refused at startup with the same code. Any other unlisted startup option is refused as unknown. No host signal or service path is consulted.
