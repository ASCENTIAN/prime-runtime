# Operations: service, logs, upgrade, rollback

This page covers running PrimeRuntime for longer than one client session. For a first install, see [INSTALL.md](INSTALL.md).

## Run as a user service

The package includes a systemd user unit template at `share/prtd.service`. It needs no root.

```sh
mkdir -p "$HOME/.config/systemd/user"
cp "$HOME/.local/prime/runtime/current/share/prtd.service" "$HOME/.config/systemd/user/"
```

By default the service serves the package directory, read-only, so the unit runs without an edit. To serve a project, copy the catalog into the project and set `PRT_ROOT` to its absolute path in a drop-in (see [MCP_INTEGRATION.md](MCP_INTEGRATION.md#serve-a-project-instead-of-the-package)):

```sh
cp -R "$HOME/.local/prime/runtime/current/catalog" /absolute/path/to/project/
systemctl --user edit prtd
```

In the editor, add:

```ini
[Service]
Environment=PRT_ROOT=/absolute/path/to/project
```

If `catalog/default.json` is missing under the root, the launcher exits with code 2 and a message on stderr, and the service fails to start.

Then start it:

```sh
systemctl --user daemon-reload
systemctl --user enable --now prtd
systemctl --user status prtd
```

The unit file was checked with `systemd-analyze verify` (syntax only). Starting the unit was not part of the 0.1.0 test, so `active (running)` is not verified for this release.

The service listens on `$XDG_RUNTIME_DIR/prime/runtime/kernel.sock` (mode `0600`). Clients connect through the stdio bridge:

```sh
prt connect "$XDG_RUNTIME_DIR/prime/runtime/kernel.sock"
```

The unit as shipped runs the kernel in guarded, read-only mode. It does not pass `--fabric-allow-write` or `--result-state-root`. A service that performs writes needs its own unit, with a write-enabled catalog, the flags, and a state directory. That setup is not part of the 0.1.0 tests.

Stop and disable:

```sh
systemctl --user disable --now prtd
```

## Logs

| Source | How to read |
| --- | --- |
| Service | `journalctl --user -u prtd` (the unit sets `PRIMESHELL_LOG=json`) |
| Direct run | Start with `PRIMESHELL_LOG=json` and read stderr |

Each log line is one JSON object. In the 0.1.0 test, a captured write and read produced the fields `category`, `duration_us`, `event`, `evidence_id`, `request_id`, `tool`, `tool_error`, `transport` and `unix_ms`. Their logs carried no argument values, file content or paths.

Logs are not sent off the host.

## Health

```sh
prt health; echo "exit=$?"
```

Run it after every start, upgrade and rollback. Exit codes are in [TROUBLESHOOTING.md](TROUBLESHOOTING.md#health-exit-codes).

## Upgrade

Upgrades are side by side. The old version stays in place until the new one is verified.

1. Download and verify the new archive, then extract it (see [INSTALL.md](INSTALL.md), steps 1 to 3). The upgrade steps on this page were not run in the 0.1.0 test.
2. Move the extracted directory to `~/.local/prime/runtime/versions/<new-version>`.
3. Stop the service, if you run one:
   ```sh
   systemctl --user stop prtd
   ```
4. Switch the active version:
   ```sh
   ln -sfn "$HOME/.local/prime/runtime/versions/<new-version>" "$HOME/.local/prime/runtime/current"
   ```
5. Check the new version:
   ```sh
   prt version
   prt health; echo "exit=$?"
   ```
   Do not continue on exit `1` or `2`. Roll back (below) and investigate.
6. Start the service again:
   ```sh
   systemctl --user start prtd
   ```

State directories and project files are not part of the package. They stay where they are. A new version may read or write receipts differently. Do not assume a newer version reads older receipts, unless the release notes say so.

## Rollback

Switch the symbolic link back to the previous version directory, restart, and check:

```sh
systemctl --user stop prtd 2>/dev/null || true
ln -sfn "$HOME/.local/prime/runtime/versions/<previous-version>" "$HOME/.local/prime/runtime/current"
prt version
prt health; echo "exit=$?"
systemctl --user start prtd 2>/dev/null || true
```

Keep the previous version directory until the new version has been used successfully.
A rollback does not delete state directories, receipts or project files.

## Backup

Back up these, in this order of importance:

1. The state directories (receipts and journal). They are the evidence of what the server wrote.
2. Your project directory.
3. The package directory, if you have changed the catalog. The default package needs no backup. You can download it again and check its digest.

## Uninstall

See [INSTALL.md](INSTALL.md#uninstall). Stop and disable the service first:

```sh
systemctl --user disable --now prtd
rm -f "$HOME/.config/systemd/user/prtd.service"
systemctl --user daemon-reload
```

## Resource use

Resource use of the service was not measured for 0.1.0. No figure is claimed.
