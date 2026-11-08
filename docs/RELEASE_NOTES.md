# PrimeRuntime 0.1.0 release notes

Release date: 2026-11-09. First public release of PrimeRuntime. It wraps the PrimeShell kernel 0.5.2.

## What this release is

PrimeRuntime is a local server. It lets an AI coding client, such as Claude Code, call a fixed list of operations on your machine. Writes are off by default. A write that is enabled is journaled, and its result is kept as a sealed receipt. A repeated request returns the stored result instead of writing again. The server contains no language model.

This release contains the execution core only. It contains no capability data.

## Included

- `prt`: the launcher. Commands: `version`, `health`, `serve`, `connect`, and the MCP server over stdio.
- `bin/primeshell`: the kernel that runs the operations. It keeps the name `primeshell` inside this release.
- A default catalog with one entry: `project.read` (reads files under the project root). It offers no write operation.
- Guarded writes with `--fabric-allow-write`, a path prefix per write capability, exact-content checks, a write journal, and sealed receipts. The default catalog has no write capability; a write needs your own catalog (see [EXAMPLE_GUARDED_WRITE](EXAMPLE_GUARDED_WRITE.md)).
- Recovery of an interrupted write when the same request is repeated.
- `prt-sandbox`: starts an MCP client in a bubblewrap sandbox with the managed directory mounted read-only. A direct write there fails; changes to it go through the server, which records a receipt. The sandbox does not confine writes elsewhere (see below).
- A Unix-socket service and a systemd user unit template.
- Opt-in JSON logs with `PRIMESHELL_LOG=json`.
- SHA-256 checksums, a build provenance record, a software bill of materials, and license texts.

## Not included

- No capability data, snapshot or routing table.
- No language model, local or remote.
- No HTTP transport.
- No command execution in guarded mode.
- No source code.
- No Windows, macOS or non-x86_64 build.

## Known limits

- Writes outside the managed directory are not restricted by the sandbox. See [SECURITY_BOUNDARIES.md](SECURITY_BOUNDARIES.md).
- Receipts are tamper-evident, not signed.
- Tested on one Linux x86_64 glibc host. See [COMPATIBILITY.md](COMPATIBILITY.md).
- `bin/primeshell` started directly, without the launcher's flags, serves a wider tool set, including command execution, HTTP GET and file writes, each enabled by a startup flag. The guarantees in these notes apply to the launcher. See [SECURITY_BOUNDARIES.md](SECURITY_BOUNDARIES.md#the-kernel-binary-started-directly).


## Verify

```sh
sha256sum -c primeruntime-v0.1.0-x86_64-unknown-linux-gnu.tar.gz.sha256
prt version
prt health; echo "exit=$?"
```

See [INSTALL.md](INSTALL.md).
