# PrimeRuntime

[![License: Apache-2.0](https://img.shields.io/badge/license-Apache--2.0-blue)](LICENSE)
![version 0.1.0](https://img.shields.io/badge/version-0.1.0-informational)
![platform linux-x86_64](https://img.shields.io/badge/platform-linux--x86__64-lightgrey)
![model-free](https://img.shields.io/badge/model--free-no%20LLM%20in%20the%20server-success)
![deterministic](https://img.shields.io/badge/deterministic-replayable%20receipts-success)

**Guarded, journaled, replayable operations for AI coding clients.**

PrimeRuntime lets an MCP client such as Claude Code call a pinned list of operations on your machine. Writes stay off until you enable them, each enabled write is journaled, and its result is kept as a SHA-256 sealed receipt. A repeated request returns the stored result instead of writing again. The server contains no language model and no capability data.

> **Version.** This is **PrimeRuntime 0.1.0**, the first public release of the PrimeRuntime product line. It wraps the **PrimeShell 0.5.2** execution kernel, which keeps its own version. `prt version` prints `primeruntime 0.1.0 (kernel: primeshell 0.5.2)`. The two numbers are expected: 0.1.0 is the release, 0.5.2 is the kernel inside it.

## What it is and what it solves

A coding client that runs its own tools has your user permissions. Prompts and hooks guide it, but they do not limit what it can reach. PrimeRuntime puts a narrower path between the client and your machine:

- a fixed operation list, pinned by SHA-256 digest, instead of the client's unrestricted tools for the calls that go through it;
- writes off by default, enabled only by startup flags, confined to a path prefix, and applied only when the file still has its expected content;
- a journal and a sealed receipt for every enabled write, with an identical repeat replayed from the stored result;
- an optional bubblewrap sandbox that confines one client process tree.

Scope: [SECURITY_BOUNDARIES](docs/SECURITY_BOUNDARIES.md) and [ARCHITECTURE](docs/ARCHITECTURE.md).

## Worked example: a guarded write, executed, then replayed

Two processes send the same request with the same `request_id`. The first writes a file. The second starts a new kernel process and returns the stored result without writing.

```text
process 1   status=EXECUTED  executed=true   outcome=committed  receipt_version=2
            receipt_sha256=<64 hex, SHA-256 over the receipt payload>

process 2   status=REPLAYED  executed=false  receipt_sha256=<same value as process 1>
            docs/hello.md holds "Hello from PrimeRuntime", written once
```

Angle brackets mark a value that differs per run. The commands, the catalog and the refusal cases (`DEVELOPMENT_REPLAY_CONFLICT`, `FABRIC_WRITE_SCOPE`, `WRITE_REQUEST_ID_REQUIRED`) are in [EXAMPLE_GUARDED_WRITE](docs/EXAMPLE_GUARDED_WRITE.md).

## Architecture

```text
MCP client (Claude Code)
   |  optional: run inside prt-sandbox (bubblewrap)
   |  managed directory read-only, kernel state hidden, environment allowlist
   |  MCP over stdio, or a Unix socket (mode 0600, same user only)
   v
prt   launcher: exactly 3 tools
   |        ps_status, ps_capability_plan, ps_capability_invoke
   |        checks the catalog digest against its pin
   v
primeshell  kernel, guarded mode: permissions come from startup flags only
   |-- catalog: operations, path prefixes, byte limits (SHA-256 pinned)
   |-- executors ----> project root (you choose it)
   '-- journal, receipts ----> state directory (owner-only, mode 0700)
```

Lifecycle and invariants: [ARCHITECTURE](docs/ARCHITECTURE.md).

## Quickstart

Five commands. The full walk-through, with the expected output of each step, is in [QUICKSTART](docs/QUICKSTART.md). The asset is `primeruntime-v0.1.0-x86_64-unknown-linux-gnu.tar.gz`, from the [v0.1.0 release](https://github.com/ASCENTIAN/prt/releases/tag/primeruntime-v0.1.0). Download it and its `.sha256` file into an empty directory.

```sh
# 1. Verify the archive
sha256sum -c primeruntime-v0.1.0-x86_64-unknown-linux-gnu.tar.gz.sha256
# expected: primeruntime-v0.1.0-x86_64-unknown-linux-gnu.tar.gz: OK

# 2. Install for your user (no root)
tar -xzf primeruntime-v0.1.0-x86_64-unknown-linux-gnu.tar.gz
mkdir -p "$HOME/.local/prime/runtime/versions" "$HOME/.local/bin"
mv primeruntime-v0.1.0-x86_64-unknown-linux-gnu "$HOME/.local/prime/runtime/versions/0.1.0"
ln -sfn "$HOME/.local/prime/runtime/versions/0.1.0" "$HOME/.local/prime/runtime/current"
ln -sfn "$HOME/.local/prime/runtime/current/bin/prt" "$HOME/.local/bin/prt"

# 3. Check the install
prt version                 # primeruntime 0.1.0 (kernel: primeshell 0.5.2)
prt health; echo "exit=$?"  # a JSON report with "healthy": true, then exit=0

# 4. Register with Claude Code
claude mcp add --scope user primeruntime -- "$HOME/.local/bin/prt"
```

Then ask Claude Code to call the `ps_status` tool of `primeruntime`. It reports version, mode (`read-only` by default), `fabric_guarded: true` and the digest of the catalog it runs.

Health exit codes: `0` healthy, `1` a write reservation is pending recovery, `2` configuration refused. See [TROUBLESHOOTING](docs/TROUBLESHOOTING.md#health-exit-codes).

## Guarantees and boundaries

| Covered | Not covered |
| --- | --- |
| Operations come only from the pinned catalog; the digest is checked on each call | Writes outside the managed directory: the sandbox binds the host filesystem read-write, except the paths it protects |
| Permissions come only from startup flags; writes are off by default | Your user account: the kernel runs as you |
| A write stays under its `path_prefix` and needs the exact current content | Processes started outside the launcher, even with the same user ID |
| Journaled writes; a repeated request replays the stored result | Root, other platforms, and clients not started through the sandbox |
| Receipts are sealed with SHA-256 (tamper-evident) | Network access of a sandboxed client, unless `--no-network` (which also blocks its model API) |
| Client sandbox: read-only managed directory, hidden kernel state, cleared environment, own namespaces | Receipts are not signed |

- **Launcher.** `tools/list` returns exactly three tools. Any other name is refused with `FABRIC_TOOL_DENIED`. Passing `--allow-write`, `--allow-command` or `--allow-http` to the launcher does not change the list.
- **Bare kernel (disclosed).** `bin/primeshell` started directly, without the launcher's flags, serves 33 tools, including command execution (`--allow-command`), HTTP GET (`--allow-http`) and file writes (`--allow-write`). Start it directly only for the worked example, with the flags you need. See [SECURITY_BOUNDARIES](docs/SECURITY_BOUNDARIES.md#the-kernel-binary-started-directly).
- **Receipts.** SHA-256 sealed and tamper-evident, not signed. A process running as your user outside the sandbox can rewrite the state directory. A sealed receipt shows that a change happened; it does not prevent one.
- **Sandbox.** Covers filesystem writes, environment and kernel state for one client process tree. It needs bubblewrap at `/usr/bin/bwrap`.

## External capability provider

This release ships no capability content and no snapshot. The default catalog offers one operation, `project.read`, which reads files under the project root. A capability that needs a snapshot is refused until an admitted snapshot is bound with `--atlas-snapshot <absolute directory>`. The kernel checks the snapshot's manifest and digests before use. A snapshot that fails any check is not used, and health exits `2`. Admission is decided by whoever curates the snapshot, outside this release. Details: [CAPABILITY_PROVIDER](docs/CAPABILITY_PROVIDER.md).

## When not to use it

- You need a language model, routing or capability data inside the server. None is included.
- You need containment of your own user account. The kernel and any process you start run as you; the sandbox covers only what runs inside it.
- You need writes through the launcher. The shipped catalog has no write operation; the worked example starts the kernel directly.
- You need tamper-proof or signed audit records against a local attacker. Receipts are tamper-evident only.
- You need Windows, macOS, arm64, musl, a root or multi-user install, or HTTP transport. None is tested or built in this release.

## Platform and status

Linux x86_64 with glibc, installed without root. Tested on one host (Aeon, kernel 7.3.0-1-default) with Claude Code 2.1.311 as the target client. Other MCP clients over stdio are expected to work but are not tested. Resource use is not measured for this release. Matrix: [COMPATIBILITY](docs/COMPATIBILITY.md).

## Documentation

| Document | For |
| --- | --- |
| [QUICKSTART](docs/QUICKSTART.md) | First install and first call |
| [INSTALL](docs/INSTALL.md) | Verified install, layout, uninstall |
| [EXAMPLE_GUARDED_WRITE](docs/EXAMPLE_GUARDED_WRITE.md) | Full guarded write, receipt and replay |
| [MCP_INTEGRATION](docs/MCP_INTEGRATION.md) | Registration, tools, transports, errors |
| [ARCHITECTURE](docs/ARCHITECTURE.md) | Components, request lifecycle, invariants |
| [SECURITY_BOUNDARIES](docs/SECURITY_BOUNDARIES.md) | What is enforced and what is not |
| [CAPABILITY_PROVIDER](docs/CAPABILITY_PROVIDER.md) | Catalog format, pins, snapshots |
| [COMPATIBILITY](docs/COMPATIBILITY.md) | Platforms, clients, limits |
| [TROUBLESHOOTING](docs/TROUBLESHOOTING.md) | Exit codes, refusal codes, recovery |
| [OPERATIONS](docs/OPERATIONS.md) | Service, logs, upgrade, rollback |
| [LICENSES](docs/LICENSES.md) | Licenses and third-party notices |
| [RELEASE_NOTES](docs/RELEASE_NOTES.md) | What 0.1.0 contains |
| [GLOSSARY](docs/GLOSSARY.md) | Terms used in these documents |
| [CHANGELOG](CHANGELOG.md) | Version history |

## License

PrimeRuntime runtime and documentation: Apache-2.0 ([LICENSE](LICENSE), [NOTICE](NOTICE)). Third-party crates keep their own licenses; their texts are in the package `licenses/` directory, and the SBOM is published with the release. The license does not cover capability content, which this release does not contain. Details: [LICENSES](docs/LICENSES.md).

## Security and contributing

To report a vulnerability, follow [SECURITY.md](SECURITY.md). Do not open a public issue for a security problem. Issues and documentation fixes are welcome; see [CONTRIBUTING.md](CONTRIBUTING.md).
