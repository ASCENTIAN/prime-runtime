# Architecture

PrimeRuntime is a local server. It lets an AI client call a fixed list of operations on your machine. It keeps a journal and receipts for writes. It contains no language model.

## Components

| Component | File | Role |
| --- | --- | --- |
| Launcher | `bin/prt` | Shell script. Checks the shipped catalog digest, then starts the kernel with guarded flags. Also provides `health`, `serve`, `connect` and `version`. |
| Kernel | `bin/primeshell` | The execution core. Speaks MCP over stdio or a Unix socket. Enforces the catalog, the permissions, the journal and the receipts. |
| Service wrapper | `bin/prtd` | Runs the kernel as a Unix-socket service. |
| Client sandbox | `bin/prt-sandbox` | Starts an MCP client in a bubblewrap sandbox. The client reaches the kernel only through the socket. |
| Default catalog | `catalog/default.json` | The operations offered by default: one, `project.read`, which reads one file under the project root. |
| Catalog pin | `catalog/default.json.sha256` | SHA-256 of the default catalog. The launcher refuses to start if the file differs. |

`primeshell` is the internal name of the kernel executable in this release. See [GLOSSARY.md](GLOSSARY.md). Started directly, without the launcher's flags, the kernel serves a wider tool set than the launcher. The guarantees in this document apply to the launcher. See [SECURITY_BOUNDARIES.md](SECURITY_BOUNDARIES.md#the-kernel-binary-started-directly).

## Diagram: one client, one kernel

```text
  MCP client (Claude Code)
        |  MCP, JSON-RPC over stdio (or a Unix socket)
        v
  bin/prt  ---- checks catalog digest against the pin
        |
        v
  bin/primeshell (kernel, guarded mode)
        |-- catalog: operations, executors, path prefixes, byte limits
        |-- startup flags: the only source of permissions
        |-- executors ---> project root (files you pinned)
        |-- journal and receipts ---> state directory (owner-only)
        '-- capability provider (optional, external snapshot; refuses if absent)
```

## Diagram: sandboxed client

```text
  +------------------- sandbox (bubblewrap) -------------------+
  |  MCP client                                                |
  |  managed directory: read-only                              |
  |  kernel state directory: hidden                            |
  |  environment: allowlist only                               |
  |        | stdio bridge: prt connect <socket>                |
  +--------|---------------------------------------------------+
           | Unix socket, mode 0600, same user ID only
           v
  prt serve <socket>  (kernel outside the sandbox, as a user service)
```

## Request lifecycle for a tool call

1. The client sends `tools/call`. In guarded mode (the launcher), the kernel accepts only the three tools it offers: `ps_status`, `ps_capability_plan`, `ps_capability_invoke`.
2. For `ps_capability_invoke`, the kernel checks the request keys against the executor. Unknown keys are refused.
3. The kernel checks the catalog pin again and resolves the capability to one executor.
4. The executor checks its path prefix and byte budgets.
5. Authorization: the catalog entry and the startup flags must allow the operation. Grants come from startup flags only.
6. For a write with a state directory: a reservation record is written, then the file is checked under a lock, the new content is written by atomic rename, and a terminal receipt is stored under the `request_id`.
7. The response is returned with `status`. A repeat of the same `request_id` returns the stored result with `REPLAYED`.

Detail and field names: [EXAMPLE_GUARDED_WRITE.md](EXAMPLE_GUARDED_WRITE.md).

## Invariants

These are the properties the design is built to keep.

| ID | Invariant | Failure behaviour |
| --- | --- | --- |
| I1 | Operations come only from the pinned catalog. | Unknown capability: `CAPABILITY_NOT_FOUND`. Pin mismatch at start: the server does not start. |
| I2 | The catalog digest is checked again on each call. | `FABRIC_CATALOG_CHANGED`. |
| I3 | Permissions come only from startup flags. Writes need `--fabric-allow-write`. | Without the flag, a write capability is refused with `CAPABILITY_UNAVAILABLE`. A path outside the prefix is refused with `FABRIC_WRITE_SCOPE`. |
| I4 | A write stays under its catalog `path_prefix`. | `FABRIC_WRITE_SCOPE`. |
| I5 | A write with a state directory is journaled: reservation, then effect, then terminal receipt. | An interrupted write is observed on retry, not rewritten blindly. See [TROUBLESHOOTING.md](TROUBLESHOOTING.md#pending-recovery). |
| I6 | A write to an existing file needs its exact current content (`expected_content`). | Refused with a conflict message; a refused receipt is stored. |
| I7 | The same `request_id` with a different binding is refused. | `DEVELOPMENT_REPLAY_CONFLICT`. |
| I8 | No capability content is used without an admitted snapshot. | A capability that needs a snapshot is refused with `CAPABILITY_UNAVAILABLE`; the shipped catalog has none. `ATLAS_SNAPSHOT_*` for a refused snapshot. See [CAPABILITY_PROVIDER.md](CAPABILITY_PROVIDER.md). |
| I9 | Receipts are sealed with SHA-256. | Any change to a stored receipt is detected by the replay check (`DEVELOPMENT_REPLAY_INVALID`). |

## What the design does not claim

- It does not isolate the kernel from your user account. The kernel runs as you.
- It does not restrict what a client does outside the sandbox. See [SECURITY_BOUNDARIES.md](SECURITY_BOUNDARIES.md).
- It does not provide routing, a model or a knowledge base in this release.

## Determinism

For the same request, the same pinned catalog, the same snapshot (if bound) and the same binary, the normalized receipt fields are the same. The fields excluded from that comparison are `operation_id`, `path`, `receipt_sha256` (which covers them), `result_bytes` and `run_id`.

Side effects are not idempotent by default. The journal returns a stored result only for the same `request_id` and the same binding. A changed binding is refused.

No model is involved, so no sampling changes the output.

## Where state lives

| Data | Location | Who writes it |
| --- | --- | --- |
| Installed package | `~/.local/prime/runtime/versions/0.1.0/` | Installer (you) |
| Files you write through guarded writes | Your project directory, under the catalog path prefix | The kernel, on an authorized call |
| Reservations, receipts | State directory `<directory>/data/primeshell/<profile>` | The kernel |
| Logs | stderr of the process (only with `PRIMESHELL_LOG=json`) | The kernel |

The launcher sends no data off the host. The default build links no HTTP, TLS or socket-client crate (checked with `cargo tree`). The kernel binary can run an HTTP GET through an external `curl` only when started directly with `--allow-http`; see [SECURITY_BOUNDARIES.md](SECURITY_BOUNDARIES.md#the-kernel-binary-started-directly).
