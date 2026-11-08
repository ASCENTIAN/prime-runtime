# Security and authority boundaries

This page says what PrimeRuntime enforces, what it does not, and how receipts can be trusted. Read it before you give any client write access.

Short version: PrimeRuntime narrows what a client can do through its tools and records writes that go through its tools. It does not sandbox your user account, and it does not sandbox the whole filesystem.

## Layers

| Layer | What it limits | Scope |
| --- | --- | --- |
| Catalog and startup flags | Which operations exist and whether writes are on | Every call through PrimeRuntime |
| Path prefix and byte budgets | Where a write may go, and how large inputs and outputs are | Every call through PrimeRuntime |
| Journal and receipts | Records each write and replays it instead of repeating it | Writes with a state directory |
| Client sandbox (`prt-sandbox`) | Filesystem writes, environment and kernel state visible to one client process tree | Only processes started through the sandbox |

## The kernel binary started directly

`bin/primeshell` is the full kernel. The guarantees on this page apply to the launcher, which starts the kernel with guarded flags and offers three tools. Started directly, without those flags, the kernel serves its complete tool set (33 tools). Its startup flags enable command execution (`--allow-command`), HTTP GET (`--allow-http`, which runs the named `curl` as a child process) and file writes (`--allow-write`). Start it directly only for the worked example in [EXAMPLE_GUARDED_WRITE.md](EXAMPLE_GUARDED_WRITE.md), and only with the flags you need.

## What the client sandbox covers

These are the properties of `bin/prt-sandbox` that the 0.1.0 test observed when it started a client.

- Writes to the managed directory (the `--ro` path) fail with a read-only file system error.
- The `--hide` path appears as an empty directory inside the sandbox.
- The environment is cleared. Only an allowlist of variables is passed.
- The client has its own process, IPC and hostname namespaces.
- With `--pin <dir>`, the parent of the project directory is read-only, so the project root cannot be renamed while it is writable.
- The kernel socket has mode `0600`. It checks the peer's user ID (`SO_PEERCRED`); a connection from a second account was not tested.
- The sandbox refuses to start without `--ro` and `--hide`.

## What the client sandbox does not cover

These are not enforced. Plan around them.

- **Writes outside the managed directory.** The sandbox binds the host filesystem read-write, except the paths it protects. A sandboxed client, running as your user, can write anywhere else you can.
- **Network access.** The client keeps network access unless you pass `--no-network`. That option also cuts off a model API, so most MCP clients cannot use it.
- **The X11 socket** when the network is shared. Your X authority cookie is hidden, but an unauthenticated X server can still be reached.
- **Processes started outside the launcher**, even with the same user ID.
- **Root.** The sandbox is a user-level tool. Root is not tested and not supported.
- **Clients not started through the sandbox.** A client registered only as an MCP server is limited only in the tools it calls through PrimeRuntime. Its own built-in tools (shell, file editing) still run with your permissions.
- **Other platforms.** Only Linux x86_64 with glibc and bubblewrap is tested.

## Read-only server, read-write client: the difference

| Question | Client with its own tools | Client with the PrimeRuntime server only | Client in `prt-sandbox` |
| --- | --- | --- | --- |
| Can it write files directly? | Yes | Yes, with its own tools | Yes, outside the managed directory |
| Can it write to the managed directory directly? | Yes | Yes | No |
| Are writes through PrimeRuntime recorded? | Yes, when writes are enabled | Yes, when writes are enabled | Only with a write-enabled service of your own. The shipped service refuses writes (`CAPABILITY_NOT_FOUND`). |
| Can it read the kernel's state directory? | Not applicable | Not applicable | No |

## Receipts: what they prove

- Each receipt is sealed with SHA-256 over its payload. The sealed value is `receipt_sha256`.
- A changed receipt is detected when the same request is replayed (`DEVELOPMENT_REPLAY_INVALID`).
- Receipts are tamper-evident. They are not signed. There is no signing key and no third-party timestamp.
- A process running as your user, outside the sandbox, can rewrite the state directory. A sealed receipt does not stop that. It shows that a change happened.
- A receipt shows that the kernel recorded a write under a `request_id`. It does not prove who sent the request beyond the client that connected to the kernel.

## Audit records (logs)

With `PRIMESHELL_LOG=json`, the kernel writes one JSON line per event to stderr. Lines carry identifiers and categories, a timestamp (`unix_ms`) and a duration (`duration_us`). In the 0.1.0 test, the log of a write and a read carried no argument values, file content or paths.

Logs are not sent anywhere. They stay on the host.

## Credentials and data

- The kernel binary contains no model-client or API-key strings (static check in the 0.1.0 test).
- The sandbox clears the environment. It passes only these variables, and only when they are set: `ANTHROPIC_API_KEY`, `ANTHROPIC_AUTH_TOKEN`, `ANTHROPIC_BASE_URL`, `CLAUDE_CONFIG_DIR`, `NO_PROXY`, `HTTPS_PROXY`, `HTTP_PROXY`, plus a fixed set of identity and locale variables. Your other variables are removed. In the 0.1.0 test, `ANTHROPIC_API_KEY` and `HTTPS_PROXY` were passed and `MY_SECRET_VAR` was removed.
- Receipts contain `path`, `content_sha256` and `expected_sha256`. They do not contain the file content (observed in a write receipt).

## Recommended setup

1. Serve the smallest directory you can. Do not serve your home directory.
2. Enable writes only for the paths you need, with a narrow `path_prefix`.
3. Use a state directory you own, with mode `0700`.
4. Run an untrusted client in `prt-sandbox`, with `--ro` on the managed directory and `--hide` on the kernel state.
5. Keep the install directory owned by your user. Do not let other users write to it.

## Reporting a vulnerability

See SECURITY.md in the public repository. Do not open a public issue for a security problem.
