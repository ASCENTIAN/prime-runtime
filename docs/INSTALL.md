# Install PrimeRuntime 0.1.0

This guide installs PrimeRuntime 0.1.0 for one Linux user account, without root.
Each step has an expected result. If a result differs, stop and see [TROUBLESHOOTING.md](TROUBLESHOOTING.md).

The outputs shown were observed in the 0.1.0 test on the host listed in [COMPATIBILITY.md](COMPATIBILITY.md).

## Requirements

| Item | Requirement |
| --- | --- |
| Operating system | Linux, x86_64, glibc. Other distributions are not tested. |
| Tools | `sh`, `tar`, `sha256sum`, `readlink`, `sed` (present on most Linux systems) |
| Sandbox (optional) | `bubblewrap` at `/usr/bin/bwrap` |
| MCP client (optional) | Claude Code, or any client that speaks MCP over stdio |
| Privileges | None. Do not install as root. Root installs are not tested. |

## Release assets

| Asset | Purpose |
| --- | --- |
| `primeruntime-v0.1.0-x86_64-unknown-linux-gnu.tar.gz` | The package: launcher, kernel, catalog, docs, licenses |
| `primeruntime-v0.1.0-x86_64-unknown-linux-gnu.tar.gz.sha256` | SHA-256 of the package |
| `primeruntime-v0.1.0.provenance.json` | Build record: versions, toolchain, flags, digests |
| `primeruntime-v0.1.0.sbom.cdx.json` | Software bill of materials (CycloneDX) |
| `LICENSE`, `NOTICE` | Project license and notice |

The package expands to one directory, `primeruntime-v0.1.0-x86_64-unknown-linux-gnu/`, with this layout:

```text
bin/primeshell          kernel executable (see GLOSSARY.md, "primeshell")
bin/prt                 launcher (shell script)
bin/prt-sandbox         client sandbox (shell script, needs bubblewrap)
bin/prtd                service wrapper (shell script)
catalog/default.json            default catalog (the operations the server offers)
catalog/default.json.sha256     SHA-256 pin of the default catalog
share/prtd.service              user service unit template
docs/                           these documents
licenses/                       third-party license texts
MANIFEST.json  PROVENANCE.json  SBOM.cdx.json  SHA256SUMS
```

## Step 1. Download the package

Download the release assets (the `.tar.gz`, its `.sha256` file, the provenance record and the SBOM) into an empty directory.
https://github.com/ASCENTIAN/prime-runtime/releases/tag/primeruntime-v0.1.0

Expected: the directory holds the `.tar.gz`, its `.sha256` file, and the provenance and SBOM files.

## Step 2. Check the package digest

```sh
sha256sum -c primeruntime-v0.1.0-x86_64-unknown-linux-gnu.tar.gz.sha256
```

Expected output:

```text
primeruntime-v0.1.0-x86_64-unknown-linux-gnu.tar.gz: OK
```

If the line says `FAILED`, delete the download and fetch it again. Do not extract it.

## Step 3. Extract and check every file

```sh
tar -xzf primeruntime-v0.1.0-x86_64-unknown-linux-gnu.tar.gz
cd primeruntime-v0.1.0-x86_64-unknown-linux-gnu
sha256sum -c SHA256SUMS
cd ..
```

Expected: one `OK` line per file, and no `FAILED` line. If any line says `FAILED`, do not install that package. In the 0.1.0 test, all 45 lines were `OK`.

## Step 4. Install for your user

```sh
V=0.1.0
SRC=primeruntime-v0.1.0-x86_64-unknown-linux-gnu
mkdir -p "$HOME/.local/prime/runtime/versions" "$HOME/.local/bin"
mv "$SRC" "$HOME/.local/prime/runtime/versions/$V"
ln -sfn "$HOME/.local/prime/runtime/versions/$V" "$HOME/.local/prime/runtime/current"
ln -sfn "$HOME/.local/prime/runtime/current/bin/prt" "$HOME/.local/bin/prt"
```

Layout after install:

| Path | Meaning |
| --- | --- |
| `~/.local/prime/runtime/versions/0.1.0/` | The installed package |
| `~/.local/prime/runtime/current` | Symbolic link to the active version (used for upgrade and rollback) |
| `~/.local/bin/prt` | Command on your `PATH` |

The product installs under the Prime family, `~/.local/prime/`, next to the PrimeShell kernel it wraps. The kernel ships inside the package as `bin/primeshell`; its name is kept.

Make sure `~/.local/bin` is on your `PATH`. Open a new shell after the change.

## Step 5. Check the version

```sh
prt version
```

Expected output:

```text
primeruntime 0.1.0 (kernel: primeshell 0.5.2)
```

## Step 6. Check the health

```sh
prt health
echo "exit=$?"
```

Expected: a JSON object with `"healthy": true`, the version, and `executable_sha256`, followed by `exit=0`.
Exit codes are described in [TROUBLESHOOTING.md](TROUBLESHOOTING.md#health-exit-codes).

## Step 7. Confirm the kernel digest

```sh
sha256sum "$HOME/.local/prime/runtime/current/bin/primeshell"
grep -A4 '"binary"' "$HOME/.local/prime/runtime/current/PROVENANCE.json"
```

Expected: the `sha256` value under `binary` in `PROVENANCE.json` is identical to the first hash. The key path is `binary.sha256`.

## Next steps

- Register the server with a client: [MCP_INTEGRATION.md](MCP_INTEGRATION.md) or [QUICKSTART.md](QUICKSTART.md).
- Run the guarded write example: [EXAMPLE_GUARDED_WRITE.md](EXAMPLE_GUARDED_WRITE.md).
- Run it as a service: [OPERATIONS.md](OPERATIONS.md).

## Uninstall

```sh
claude mcp remove prt --scope user      # only if you registered it with Claude Code
rm -f "$HOME/.local/bin/prt"
rm -rf "$HOME/.local/prime/runtime"
```

Your project files and any state directory you created (receipts and journal) are not removed.
Delete the state directory yourself only after you no longer need its receipts.
