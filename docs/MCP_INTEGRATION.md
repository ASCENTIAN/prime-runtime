# MCP integration

MCP (Model Context Protocol) is the protocol that clients such as Claude Code use to call tools. PrimeRuntime is an MCP server. This page shows how to register it, which tools it offers, and how a call looks on the wire.

## Transports

| Transport | Start command | Use |
| --- | --- | --- |
| stdio | `prt` | A client starts the server as a child process. Default for Claude Code. |
| Unix socket | `prt serve <socket>` (or `prtd <socket>`) | A long-running service. Clients connect with `prt connect <socket>`. Used by the sandbox. |

The HTTP transport is not built in this release.

## Register with Claude Code

Register the launcher once per user:

```sh
claude mcp add --scope user prt -- "$HOME/.local/bin/prt"
```

Scopes: `user` (all projects), `local` (this project, private), `project` (shared through the project's `.mcp.json`). On Claude Code 2.1.311, `claude mcp add --help` lists these three scopes. Writing to the `project` scope was not run.

To list or remove the registration:

```sh
claude mcp list
claude mcp remove prt --scope user
```

### Serve a project instead of the package

By default the server serves the package directory, read-only. To serve a project directory:

1. Copy the catalog into the project. The launcher reads `catalog/default.json` under the served root and checks it against the shipped digest:

   ```sh
   cp -R "$HOME/.local/prime/runtime/current/catalog" /path/to/project/
   ```

2. Register the launcher with `PRT_ROOT` set to the project:

   ```sh
   claude mcp add --scope user -e PRT_ROOT=/path/to/project prt -- "$HOME/.local/bin/prt"
   ```

`PRT_CATALOG` names the catalog file as a path relative to `PRT_ROOT` (default `catalog/default.json`), as an alternative to copying the directory. The file must still match the shipped digest. If the catalog is missing under the root, the launcher exits with code 2 and a message on stderr, and the server does not start. In the 0.1.0 test, a catalog at an alternative path with the same bytes as the shipped one was accepted (health exit 0). A missing catalog produced the message shown in [TROUBLESHOOTING.md](TROUBLESHOOTING.md#health-exit-codes).

## Tools offered in guarded mode

The launcher always runs the kernel in guarded mode. These tools are visible:

| Tool | Arguments | Purpose |
| --- | --- | --- |
| `ps_status` | none | Report the kernel version, root, mode (`read-only` or `read-write`), guarded state and the catalog digest. |
| `ps_capability_plan` | `catalog`, `capability`, `input_bytes`, `output_bytes` (optional) | Show the plan for one catalog capability under the startup permissions. Does not execute. |
| `ps_capability_invoke` | `capability`, `input`, `output_bytes`, `request_id` | Execute one catalog capability. Writes need the startup flags, a path prefix and `request_id`. |

`tools/list` returns exactly these three tools. Any other tool name is refused with `FABRIC_TOOL_DENIED`.

## Protocol example

The kernel accepts an `initialize` request first. The protocol version used in tests is `2025-06-18`.

Client to server (one JSON object per line):

```json
{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18"}}
{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ps_capability_invoke","arguments":{"capability":"project.read","output_bytes":4096,"input":{"path":"README.md"}}}}
```

Server to client: the tool result is returned in `result.structuredContent`, and as JSON text in `result.content[0].text`. Refusals come in two shapes, observed in the 0.1.0 test. Some are bare text with `isError` set to `true`, for example `FABRIC_TOOL_DENIED`, `FABRIC_OUTPUT_LIMIT`, `FABRIC_WRITE_SCOPE` and `DEVELOPMENT_REPLAY_CONFLICT`. `CAPABILITY_NOT_FOUND` returns `isError` set to `false`, and its refusal is in `structuredContent.status`. Read the status field, not only `isError`.

The read above uses the default catalog's `project.read` (reads only). Its input accepts `path`, `offset_lines` and `limit_lines`.

## Writes through a client

A write needs three things at startup: a catalog that offers a write executor, the flag `--fabric-allow-write`, and a state directory. The default launcher provides none of these. A write-enabled setup is described in [EXAMPLE_GUARDED_WRITE.md](EXAMPLE_GUARDED_WRITE.md). It uses the kernel directly, not the launcher.

## Unix socket service

Run the kernel as a user service, then connect clients to the socket:

```sh
prt serve "$XDG_RUNTIME_DIR/prime/runtime/kernel.sock"     # foreground, for a test
prt connect "$XDG_RUNTIME_DIR/prime/runtime/kernel.sock"   # stdio bridge for a client
```

Socket rules: mode `0600` (observed). One frame is at most 1 MiB (observed: a larger frame returned `Frame exceeds 1 MiB limit`). The peer's user ID is checked when a client connects (`SO_PEERCRED`); a connection from a second account was not tested. When a 17th concurrent client connects, the oldest idle client is disconnected (observed).

For a systemd user service, see [OPERATIONS.md](OPERATIONS.md).

## Errors a client will see

| Code | Meaning |
| --- | --- |
| `FABRIC_TOOL_DENIED` | The tool is not offered in guarded mode |
| `FABRIC_GUARDED_MODE_REQUIRED` | `ps_capability_invoke` was sent to a server that is not guarded |
| `CAPABILITY_NOT_FOUND` | The capability is not in the catalog, for example `faculty.builtin`, which the shipped catalog does not offer |
| `CAPABILITY_UNAVAILABLE` | The capability needs data that is not bound (for example, a snapshot), or a write was called without `--fabric-allow-write`. The shipped catalog has no such capability. |
| `FABRIC_FIELDS_MISMATCH` | The `input` keys do not match the executor |
| `OUTPUT_BUDGET_EXCEEDED` | The result is larger than `output_bytes` |
| `FABRIC_OUTPUT_LIMIT` | `output_bytes` was omitted (observed for `project.read`). Always pass `output_bytes`. |

The full list is in [TROUBLESHOOTING.md](TROUBLESHOOTING.md#refusal-codes).
