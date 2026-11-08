# Quickstart (about five minutes)

This page takes you from download to a first call from Claude Code. Each step lists what you should see.
Full details: [INSTALL.md](INSTALL.md) and [MCP_INTEGRATION.md](MCP_INTEGRATION.md).

The outputs shown were observed in the 0.1.0 test on the host listed in [COMPATIBILITY.md](COMPATIBILITY.md).

## 1. Download and check (1 minute)

Download `primeruntime-v0.1.0-x86_64-unknown-linux-gnu.tar.gz` and its `.sha256` file into one empty directory. https://github.com/ASCENTIAN/prime-runtime/releases/tag/primeruntime-v0.1.0

```sh
sha256sum -c primeruntime-v0.1.0-x86_64-unknown-linux-gnu.tar.gz.sha256
```

Expected: `primeruntime-v0.1.0-x86_64-unknown-linux-gnu.tar.gz: OK`

## 2. Unpack and install (1 minute)

```sh
tar -xzf primeruntime-v0.1.0-x86_64-unknown-linux-gnu.tar.gz
mkdir -p "$HOME/.local/prime/runtime/versions" "$HOME/.local/bin"
mv primeruntime-v0.1.0-x86_64-unknown-linux-gnu "$HOME/.local/prime/runtime/versions/0.1.0"
ln -sfn "$HOME/.local/prime/runtime/versions/0.1.0" "$HOME/.local/prime/runtime/current"
ln -sfn "$HOME/.local/prime/runtime/current/bin/prt" "$HOME/.local/bin/prt"
```

Expected: `~/.local/bin/prt` exists. [INSTALL.md](INSTALL.md) explains each step and the file layout.

## 3. Check the install (30 seconds)

```sh
prt version
prt health; echo "exit=$?"
```

Expected:

```text
primeruntime 0.1.0 (kernel: primeshell 0.5.2)
{"healthy":true, ... "executable_sha256": "...", ...}
exit=0
```

Exit 0 means healthy. Other codes are explained in [TROUBLESHOOTING.md](TROUBLESHOOTING.md#health-exit-codes).

## 4. Register with Claude Code (1 minute)

```sh
claude mcp add --scope user prt -- "$HOME/.local/bin/prt"
```

Expected: Claude Code reports `Added stdio MCP server prt` to the user configuration. `claude mcp list` then shows `prt` as connected (observed with Claude Code 2.1.311).

By default the server serves the package directory, read-only. It offers `project.read` for reading files and no write operation. See [MCP_INTEGRATION.md](MCP_INTEGRATION.md) to serve a project.

## 5. First call (1 minute)

Start Claude Code and ask it to call the `ps_status` tool of `prt`. This step was not run in the 0.1.0 test, because it needs a model session. Step 5 runs the same tool call through the launcher.

Expected: a status report with `"version": "0.5.2"` (the kernel version), `"mode": "read-only"`, `"fabric_guarded": true`, and a `fabric_catalog_sha256` value equal to the digest in `catalog/default.json.sha256`.

To make the same call without a client, send two JSON-RPC lines to the launcher:

```sh
printf '%s\n%s\n' \
  '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18"}}' \
  '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ps_status","arguments":{}}}' \
  | "$HOME/.local/bin/prt" | sed -n 2p
```

Observed: one JSON line whose tool result carries the same status fields.

## 6. Next

- A guarded write with its receipt and replay: [EXAMPLE_GUARDED_WRITE.md](EXAMPLE_GUARDED_WRITE.md). This takes about ten minutes.
- How the write is recorded: [ARCHITECTURE.md](ARCHITECTURE.md).

## Remove

```sh
claude mcp remove prt --scope user
rm -f "$HOME/.local/bin/prt"
rm -rf "$HOME/.local/prime/runtime"
```
