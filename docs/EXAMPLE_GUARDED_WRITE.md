# Worked example: a guarded, journaled write

This example writes one file through PrimeRuntime, shows the receipt, and then repeats the same request from a second process.
The second process does not write again. It returns the stored result (`REPLAYED`).

Time: about ten minutes. Prerequisite: [INSTALL.md](INSTALL.md) is complete and `prt health` exits 0.

The outputs shown were observed in the 0.1.0 test on the host listed in [COMPATIBILITY.md](COMPATIBILITY.md). Your values will differ where they depend on your system, such as paths and your own file contents.

## Why this example bypasses the launcher

The default catalog of this release offers one read operation (`project.read`). It has no write operation.
The launcher `prt` starts the server only with the shipped catalog, so it cannot start a write-enabled catalog.

This example therefore starts the kernel executable `bin/primeshell` directly, with:

- your own catalog file, pinned by its SHA-256 digest;
- the startup flag `--fabric-allow-write`, which is the only way to enable writes;
- a state directory (`--result-state-root`) that holds the journal and receipts.

In the 0.1.0 test, the launcher refused a write-enabled catalog with `prt: catalog digest mismatch; refusing to start` and exit 2.

## Step 1. Create a project and a state directory

```sh
PKG="$HOME/.local/prime/runtime/current"          # the installed package
DEMO="$HOME/prt-demo"                              # a project directory you own
ESTATE="$HOME/.local/state/prime/runtime"            # holds state directories you own
PROFILE=demo

mkdir -p "$DEMO/docs" "$DEMO/catalog"
mkdir -p "$ESTATE/data/primeshell/$PROFILE"
chmod 700 "$ESTATE" "$ESTATE/data" "$ESTATE/data/primeshell" "$ESTATE/data/primeshell/$PROFILE"
```

The state directory must have the form `<directory>/data/primeshell/<profile>`, an absolute path, with no symbolic links and with owner-only access. The kernel refuses any other shape.

## Step 2. Write the catalog

Save this as `$DEMO/catalog/docs-write.json`:

```json
{
  "schema_version": 1,
  "capabilities": [
    {
      "id": "project.read",
      "version": "1",
      "kind": "tool",
      "executors": [{
        "id": "rc.local.read",
        "adapter": "native",
        "state": "ready",
        "priority": 1,
        "permissions": ["read_project"],
        "max_input_bytes": 1024,
        "max_output_bytes": 32768
      }]
    },
    {
      "id": "docs.write",
      "version": "1",
      "kind": "tool",
      "executors": [{
        "id": "rc.local.write",
        "adapter": "native",
        "state": "ready",
        "priority": 1,
        "permissions": ["write_project"],
        "path_prefix": "docs",
        "max_input_bytes": 16384,
        "max_output_bytes": 4096
      }]
    }
  ]
}
```

Write it with a heredoc or an editor. Then pin it:

```sh
CAT_SHA=$(sha256sum "$DEMO/catalog/docs-write.json" | cut -d' ' -f1)
echo "$CAT_SHA"
```

The `path_prefix` `docs` limits the write to files under `docs/`. The kernel rechecks the pin on each call. If the file changes after startup, calls are refused with `FABRIC_CATALOG_CHANGED`.

## Step 3. Make the first call (process 1)

The request has two JSON-RPC lines: the handshake and one tool call to `ps_capability_invoke`.

```sh
REQ='{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18"}}'
WRITE='{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ps_capability_invoke","arguments":{"capability":"docs.write","output_bytes":4096,"request_id":"hello-0001","input":{"path":"docs/hello.md","content":"Hello from PrimeRuntime\n","expected_content":null}}}}'

printf '%s\n%s\n' "$REQ" "$WRITE" | "$PKG/bin/primeshell" --root "$DEMO" \
  --fabric-server-catalog catalog/docs-write.json --fabric-server-sha256 "$CAT_SHA" \
  --fabric-allow-write --result-state-root "$ESTATE/data/primeshell/$PROFILE" \
  | sed -n 2p
```

Input meaning:

| Field | Meaning |
| --- | --- |
| `capability` | `docs.write`, an entry in the pinned catalog |
| `output_bytes` | Upper limit for the whole response, 512 to 32768 |
| `request_id` | Idempotency key, 1 to 64 characters, letters, digits and hyphens. Required when a state directory is set. |
| `input.path` | Path relative to the project root. It must start with `docs/`. |
| `input.content` | The new file content |
| `input.expected_content` | `null` means "create only". To change a file, pass its exact current content. |

Observed: `status` is `EXECUTED`, `executed` is `true`, and the receipt reports `outcome` `committed` and `receipt_version` `2`.

Shape of the tool result (the JSON is in `result.structuredContent` or in `result.content[0].text`, depending on the client):

| Field | Expected value for this call |
| --- | --- |
| `status` | `EXECUTED` |
| `executed` | `true` |
| `capability` | `docs.write` |
| `executor` | `rc.local.write` |
| `catalog_sha256` | The value of `$CAT_SHA` |
| `result.receipt_version` | `2` |
| `result.receipt_sha256` | 64 hexadecimal characters, the SHA-256 over the receipt payload |
| `result.result.outcome` | `committed` |
| `result.result.committed` | `true` |
| `result.result.development_binding.path` | `docs/hello.md` |
| `result.result.development_binding.content_sha256` | SHA-256 of `Hello from PrimeRuntime` followed by a newline |

Check the file:

```sh
cat "$DEMO/docs/hello.md"
```

Expected: `Hello from PrimeRuntime`

## Step 4. Repeat the call from a second process (process 2)

Run the same command again. Each run starts a new kernel process.

```sh
printf '%s\n%s\n' "$REQ" "$WRITE" | "$PKG/bin/primeshell" --root "$DEMO" \
  --fabric-server-catalog catalog/docs-write.json --fabric-server-sha256 "$CAT_SHA" \
  --fabric-allow-write --result-state-root "$ESTATE/data/primeshell/$PROFILE" \
  | sed -n 2p
```

Expected:

| Field | Expected value |
| --- | --- |
| `status` | `REPLAYED` |
| `executed` | `false` |
| `result.receipt_sha256` | Identical to the value from step 3 |
| `result.result` | Identical to the value from step 3 |

The file is not written again. `cat "$DEMO/docs/hello.md"` still prints the same single line.

The journal keeps the record of this request in the state directory:

```sh
ls "$ESTATE/data/primeshell/$PROFILE"
```

Observed: the directory holds the reservation record `development-start-hello-0001` and the terminal receipt `development-result-hello-0001`, and also a `root-identity` file (mode 0600) and a `work/` directory. Do not edit or delete these files while you still need the receipt.

## Step 5. See the refusals (optional)

Each case below uses the same pinned catalog and flags. Only the request changes.

| Request change | Expected result | Why |
| --- | --- | --- |
| Same `request_id` (`hello-0001`) with different content | Refused with `DEVELOPMENT_REPLAY_CONFLICT` | The idempotency key is bound to the first request's path, content and catalog digest |
| `path` set to `src/app.py` | Refused with `FABRIC_WRITE_SCOPE` | Outside the catalog `path_prefix` |
| `expected_content` that does not match the file | Refused. The message is `Conflict: file content changed; read it again before writing`. A refused receipt is stored under that `request_id`. | Stops silent overwrites |
| No `request_id` | Refused with `WRITE_REQUEST_ID_REQUIRED` | A state directory is set, so every write must be journaled |

Each row was observed in the 0.1.0 test.

## How the write was recorded

1. The request was checked against the catalog and the startup flags.
2. A reservation record was written to the journal before any change.
3. The file was checked under a lock. The new content was written with an atomic rename.
4. A receipt was stored under `request_id`. The second process found it and returned it.

The receipt is sealed with SHA-256. It is tamper-evident, not signed. See [SECURITY_BOUNDARIES.md](SECURITY_BOUNDARIES.md).

## Clean up the demo

```sh
rm -rf "$DEMO"
rm -rf "$ESTATE"
```

The second command deletes the receipt. Run it only when you no longer need the receipt.
