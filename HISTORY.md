# History

PrimeRuntime is the public, data-free build of an execution kernel developed privately as PrimeShell. It installs under the Prime family at `~/.local/prime/runtime/`, alongside the kernel. The kernel's source repository stays private; this page records how the released runtime came to be. Each milestone below is backed by a commit in the private history. The public evidence for the release itself is the tagged archive, its checksum and `primeruntime-v0.1.0.provenance.json`, which names the exact private source commit the binary was built from.

## How it began

The project started on **30 September 2026** as a small local-first Rust tool set for MCP agents, working name *Rust Commander*. Within its first days it gained a JSON core with atomic patches and schema validation, offline packaging with integrity checks, and verification against the official Rust MCP SDK. The same week it introduced the idea that shaped everything after: **capabilities are dispatched only through a guarded, pinned path, where policy is checked before any effect**.

## Becoming a kernel

- **1 October 2026** - a 0.5.0 development line: 32 local MCP tools, offline packages for x86_64 and ARM64.
- **2 October 2026** - renamed **PrimeShell**, a governed local runtime for MCP tools.
- **15 October 2026** - the execution plane was rebuilt and the release surface reset.
- **2 November 2026** - a dependency advisory (RUSTSEC-2026-0386) was fixed and the remote-auth stack moved off the affected crate.

## The public, data-free line

- **8 November 2026** - the kernel gained an OS-enforced client sandbox (a bubblewrap launcher and a Unix-socket service), a health command and structured logs.
- **8 November 2026** - the build was split so the public binary carries **no capability data**: routing and skills come only from an external, admission-gated snapshot, and the public build ships none.
- **8 November 2026** - the public, data-free runtime took shape: a launcher, daemon, client sandbox and service unit over the kernel, carrying no capability data.
- **8 November 2026** - an internal candidate was briefly made public and withdrawn within minutes, after review found internal material in its archive. It was never generally available; the corrected release, PrimeRuntime 0.1.0, replaces it.

## First public release

**9 November 2026 - PrimeRuntime 0.1.0**, tag `primeruntime-v0.1.0`. The public product line begins at 0.1.0; the execution kernel inside it is PrimeShell 0.5.2, which `prt version` reports. The product is named PrimeRuntime, the agnostic runtime mechanism. The release is reproducible: two independent clean builds and an independent reviewer's build all produce the same archive and binary digests.

## Origins

- **Rust MCP tooling** - the earliest code, verified against the official Rust SDK; the kernel grew out of it.
- **PrimeShell** - the private codebase PrimeRuntime is cut from. PrimeShell integrates the kernel with its own capability data; PrimeRuntime ships the kernel without that data.
- **External capability source** - capability definitions are owned outside the kernel and enter only through an admitted snapshot whose digests match the compiled pins. The public build ships none.
- **Independent admission** - identity and admission of capability content are decided by an external review process, not by the runtime, which files requests and refuses until a decision exists.
- **Deterministic, content-addressed design** - the receipt, journal and replay model follows a design reference for deterministic, content-identified execution that was read, not imported: no code or runtime dependency was taken from it.

## Authorship

Author: Doron Tal. Development used AI coding assistants under the author's direction and review.
