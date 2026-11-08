# Troubleshooting

Start with two commands. Their output identifies most problems:

```sh
prt version
prt health; echo "exit=$?"
```

Never paste file content or receipts with content into a public issue. Paste the version output and the health report only.

## Health exit codes

| Exit code | Meaning | What to do |
| --- | --- | --- |
| `0` | Healthy | Nothing |
| `1` | Pending recovery: a write reservation has no result yet | See [Pending recovery](#pending-recovery) |
| `2` | Configuration refused: catalog digest, state directory, snapshot or flags | Read the `error` field or the message on stderr, then see the matching section below |

The launcher prints this message on stderr and exits with `2` when the default catalog does not match its pin:

```text
prt: catalog digest mismatch; refusing to start
```

If `catalog/default.json` is missing under `PRT_ROOT`, the launcher also exits with `2` and a message on stderr. Copy the catalog into the served project (see [MCP_INTEGRATION.md](MCP_INTEGRATION.md#serve-a-project-instead-of-the-package)). The observed message is:

```text
prt: <root>/catalog/default.json not found: copy the package catalog/ into PRT_ROOT or set PRT_CATALOG, then restart
```

## Catalog pin mismatch

Symptoms: the launcher prints the message above, or the kernel reports `FABRIC_CATALOG_CHANGED` or `FABRIC_INVALID_PIN`.

Causes:

- `catalog/default.json` was edited after install.
- A project copy of the catalog was changed.
- The pin passed to the kernel was not a 64-character lowercase hexadecimal SHA-256.

Fix:

1. Do not edit `catalog/default.json` in the installed package.
2. Compare the file with the digest:
   ```sh
   sha256sum "$HOME/.local/prime/runtime/current/catalog/default.json"
   cat "$HOME/.local/prime/runtime/current/catalog/default.json.sha256"
   ```
3. If they differ, reinstall the package from the verified archive (see [INSTALL.md](INSTALL.md)).
4. For a write-enabled catalog you wrote yourself, pass its own digest. See [EXAMPLE_GUARDED_WRITE.md](EXAMPLE_GUARDED_WRITE.md).

## Pending recovery

Symptoms: `prt health` exits with `1`, and the journal shows pending reservations.

Cause: a write reserved its record, and the process stopped before it stored a result. The kernel does not rewrite the file in this state.

Recovery:

1. Do not delete files from the state directory.
2. Repeat the same request: the same `capability`, `request_id`, `path`, `content` and `expected_content`. Pass the same state directory and the same startup flags.
3. The kernel checks the file under a lock, then does one of the following:
   - the new content is present: it stores a `RECOVERED` receipt and writes nothing;
   - the original content is present: it performs the write once and stores the receipt;
   - anything else: it refuses with `WRITE_REPLAY_UNCERTAIN` and touches nothing.
4. Check health with the kernel that holds the state directory, using the same root, catalog, pin and flags as the write (the flags in [EXAMPLE_GUARDED_WRITE.md](EXAMPLE_GUARDED_WRITE.md)) plus `--health`. In the 0.1.0 test, after a write, this reported `healthy: true`, `mode: read-write` and `journal.pending_reservations: 0`, and exited `0`. Do not use `prt health --result-state-root <dir>` for this check. The launcher serves the package root, so with a state directory created for another root it exits `2` with `{"error":"ROOT_IDENTITY_CHANGED","healthy":false}`.

If you get `WRITE_REPLAY_UNCERTAIN`: the file is in a state that neither the old nor the new content explains. Check the file yourself. Keep the state directory, and report the case with the health output (see [Reporting a problem](#reporting-a-problem)).

## Refusal codes

Codes appear in the tool error or in the receipt. Codes with `:<name>` add a detail after the colon.

| Code | Meaning | Typical fix |
| --- | --- | --- |
| `FABRIC_CATALOG_CHANGED` | The catalog file no longer matches its pin | See [Catalog pin mismatch](#catalog-pin-mismatch) |
| `FABRIC_INVALID_PIN` | The pin is not 64 lowercase hexadecimal characters | Pass the full SHA-256 value |
| `FABRIC_TOOL_DENIED` | The tool is not offered in guarded mode | Use one of the three guarded tools |
| `FABRIC_GUARDED_MODE_REQUIRED` | `ps_capability_invoke` was sent to a server that is not guarded | Start the server with a catalog |
| `FABRIC_FIELDS_MISMATCH` | The `input` keys do not match the executor | Send only the keys the executor takes |
| `FABRIC_WRITE_SCOPE` | The path is outside the `path_prefix` | Use a path under the prefix |
| `FABRIC_EXPECTED_CONTENT` | `expected_content` is neither a string nor `null` | Send a string, or `null` for create-only |
| `FABRIC_UNKNOWN_PERMISSION`, `FABRIC_DUPLICATE_PERMISSION` | A catalog permission is not valid, or is repeated | Fix the catalog |
| `CAPABILITY_NOT_FOUND` | The capability is not in the catalog | Check the `id` |
| `CAPABILITY_UNAVAILABLE` | The capability needs data that is not bound, or a write capability was called without `--fabric-allow-write` | See [Capability provider](CAPABILITY_PROVIDER.md), or pass the flag |
| `WRITE_REQUEST_ID_REQUIRED` | A write with a state directory has no `request_id` | Add a `request_id` |
| `DEVELOPMENT_REPLAY_CONFLICT` | The same `request_id` was used for a different request | Use a new `request_id` for a new request |
| `DEVELOPMENT_REPLAY_INVALID` | A stored receipt does not match its seal | Keep the state directory and report it |
| `WRITE_REPLAY_UNCERTAIN` | Recovery found an unexpected file state; nothing was touched | See [Pending recovery](#pending-recovery) |
| `ROOT_IDENTITY_CHANGED` | The state directory was created for a different project root than the one served | Restart the server with the root the state directory was created for |
| `HOST_INTEGRATION_UNAVAILABLE` | A protected process launch was requested. The public build has no host integration. | Not available in this release. See [Compatibility](COMPATIBILITY.md#protected-process-launches) |
| `OUTPUT_BUDGET_EXCEEDED` | The result is larger than `output_bytes` | Raise `output_bytes` (limit 32768) or read less |
| `FABRIC_OUTPUT_LIMIT` | `output_bytes` was omitted (observed for `project.read`) | Pass `output_bytes` |
| `Conflict: file content changed; read it again before writing` | `expected_content` does not match the file | Read the file again and send its current content |
| `Result state root must be <estate>/data/primeshell/<profile>` | The state directory has the wrong shape | Use the form in the example |
| `ATLAS_SNAPSHOT_*` | A capability snapshot was refused; the suffix names the check that failed | See [Capability provider](CAPABILITY_PROVIDER.md#snapshot-binding) |

## Sandbox problems

| Message or symptom | Meaning | Fix |
| --- | --- | --- |
| `refusing: --ro <managed prefix> and --hide <kernel state> are required` | The sandbox needs both options | Pass `--ro` and `--hide` |
| `--ro path missing` or `--pin dir missing` | A path given to the sandbox does not exist | Correct the path |
| `bwrap: Can't find source path` (exit `1`) for a path that contains a space | The sandbox's whitespace check runs after the path list is split, so it does not see the space; bubblewrap refuses the path. The sandbox fails closed. It does not print its documented whitespace message. | Use a path without spaces |
| `prt-sandbox: <path> is not a socket` | The kernel socket is not running | Start the service (see [OPERATIONS.md](OPERATIONS.md)) |
| `/usr/bin/bwrap: not found` or the sandbox does not start | bubblewrap is missing | Install bubblewrap, or set `BWRAP` to its path |
| The client reports `Read-only file system` when it writes to the managed directory | Working as intended | Write through the server |
| Socket `Permission denied` | The socket belongs to another user | Connect as the user who runs the service |

## Logs

Set `PRIMESHELL_LOG=json` to get one JSON line per event on stderr. The lines carry the event, tool, category, `duration_us`, `request_id` and transport. In the 0.1.0 test, the log of a write and a read carried no argument values, file content or paths.

## Reporting a problem

Open an issue in the public repository with:

- the output of `prt version`;
- the output of `prt health`, with any path you consider private removed;
- the refusal code and the step that produced it;
- your distribution and kernel version.

For a security problem, follow the SECURITY.md of the public repository instead.
