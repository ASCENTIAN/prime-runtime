# Glossary

Terms are listed in the order a reader meets them.

**PrimeRuntime.** The product, the agnostic runtime mechanism: a local server that lets an AI client call a fixed list of operations, with journaled writes and receipts. Version 0.1.0 is the first public release of PrimeRuntime. It wraps the PrimeShell kernel 0.5.2.

**MCP (Model Context Protocol).** The protocol that AI clients use to call tools on a server. Messages are JSON-RPC objects.

**Client.** The program that calls PrimeRuntime's tools, such as Claude Code.

**Launcher (`prt`).** The shell script you run. It checks the default catalog and starts the kernel with the right flags. It also provides `version`, `health`, `serve` and `connect`.

**Kernel.** The program that runs the operations and keeps the journal. Its file is `bin/primeshell`.

**primeshell.** The internal name of the kernel executable and of its server name (`PrimeShell MCP`) in this release. It is kept so that the binary and its reports match the released build. It is not a separate product.

**Guarded mode.** The mode the launcher always uses. Only three tools are offered, the catalog is pinned, and writes need explicit flags.

**Catalog.** A JSON file that lists the operations (capabilities), their executors, their permissions and their limits.

**Capability.** One operation in the catalog, for example `project.read`.

**Executor.** The implementation that runs one capability. Each capability names one or more executors.

**Catalog pin.** The SHA-256 digest of the catalog file. The kernel checks it at start and on each call. The launcher checks the default catalog against `catalog/default.json.sha256`.

**Permission.** A grant named in the catalog, such as `read_project` or `write_project`. Permissions are granted only by startup flags.

**Startup flags.** Command-line options that enable features. The flags that start with `--fabric-` control guarded mode. `--fabric-allow-write` enables writes.

**Path prefix.** The only directory a write capability may change, relative to the project root.

**Project root.** The directory the server serves. The default is the package directory.

**Managed directory.** The directory the client sandbox makes read-only for the client. Usually the project root.

**Sandbox.** The `prt-sandbox` wrapper. It runs a client in a bubblewrap container with a limited view of the filesystem and environment. Its limits are in [SECURITY_BOUNDARIES.md](SECURITY_BOUNDARIES.md).

**bubblewrap.** A Linux tool that starts a process in a restricted view of the system. The sandbox needs it.

**Unix socket.** A local connection point for processes on one host. The kernel can serve clients over one.

**stdio.** Standard input and output. A client that starts the server as a child process talks to it over stdio.

**request_id.** The idempotency key of a write: 1 to 64 letters, digits or hyphens. The same request ID with the same request returns the stored result.

**Idempotency key.** A value that makes a repeated request return the first result instead of acting again.

**expected_content.** For a write to an existing file, the exact content the file must have now. `null` means the file must not exist yet.

**Journal.** The record of each write, kept before and after the change, in the state directory.

**Reservation.** The journal entry written before a write changes a file. It has no result until the write finishes.

**Replay.** Returning the stored result of an earlier request with the same `request_id`, without writing again. The response status is `REPLAYED`.

**Recovery.** What the kernel does with a reservation that has no result, when the same request is repeated. See [TROUBLESHOOTING.md](TROUBLESHOOTING.md#pending-recovery).

**Pending recovery.** A state in which at least one reservation has no result. `prt health` exits with `1`.

**State directory.** The directory that holds the journal and receipts. It must have the form `<directory>/data/primeshell/<profile>`, with owner-only access.

**Estate.** The directory you own that contains the state directory path. The kernel requires the name `data/primeshell/<profile>` under it. The word appears in refusal messages for that reason.

**Profile.** The last part of the state directory path, for example `demo`. Letters, digits, dots, underscores and hyphens only.

**Receipt.** The stored record of one write: its outcome, the digests of its path and content, and its seal.

**Seal.** A SHA-256 digest over the receipt payload (`receipt_sha256`). It shows that a receipt has not changed. It is not a signature.

**Receipt version.** The format number of a receipt. This release writes version 2.

**Atlas (in names only).** An internal name for the capability data a snapshot carries. It appears only in the refusal codes `ATLAS_SNAPSHOT_*` and the `--atlas-snapshot` flag, because the released binary uses those names. This release ships no such data.

**Snapshot.** A directory of capability data supplied from outside the release. The kernel binds it only if every check passes. None is shipped.

**Admitted snapshot.** A snapshot whose admission status is `ADMITTED`. Admission is decided by the snapshot's curator, not by this release.

**Pin label.** A name such as `SEAT_PIN`, printed by `health` and `ps_status`. It names one registry table that an external snapshot must supply; its value is that table's SHA-256 digest. The release ships none of those tables. See [CAPABILITY_PROVIDER.md](CAPABILITY_PROVIDER.md#registry-pin-labels).

**Capability provider.** The part of the server that answers capabilities that need a snapshot. Without an admitted snapshot it refuses. The shipped catalog has no such capability.

**Routing.** Choosing which capability answers a task. This release has no routing data, so routing needs a snapshot.

**Refusal.** A request the server declines, with a code and a reason. Nothing is changed by a refusal, except that a refused write is recorded when a state directory is set.

**Health report.** The JSON output of `prt health`: whether the server is healthy, its version, the kernel digest, the mode and the journal counts.

**Executable digest.** The SHA-256 of `bin/primeshell`, reported by `health` and `ps_status`. Compare it with `PROVENANCE.json`.

**Provenance record.** `PROVENANCE.json`. It records the version, the toolchain, the build flags and the digests of the release.

**SBOM (software bill of materials).** A list of software components, in CycloneDX format, here `SBOM.cdx.json`.

**Managed prefix.** Another name for the managed directory, used in the sandbox options.
