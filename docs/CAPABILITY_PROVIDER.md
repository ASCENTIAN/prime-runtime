# Capability catalog and capability provider

PrimeRuntime offers a fixed list of operations. The list lives in a catalog file. Some operations need data that does not ship with this release. That data is supplied as an external snapshot. This page explains both, and what happens when a snapshot is missing or refused.

## Status of this release

- The release ships **no capability content**: no snapshot, no table of capabilities, no routing data.
- The default catalog offers one entry, `project.read`, which reads files under the project root. It needs no snapshot.
- PrimeRuntime does not fall back to any built-in capability data. The shipped catalog has no operation that needs a snapshot.
- Admission of a snapshot is decided by the party that curates the snapshot, outside this release. This release does not grant, claim or demonstrate admission for any snapshot.

## Catalog format

A catalog is a JSON file with `schema_version` 1 and a list of `capabilities`. Each capability has an `id`, a `version`, a `kind` (`tool`) and one or more `executors`.

| Executor field | Meaning |
| --- | --- |
| `id` | Executor name, for example `rc.local.read` |
| `adapter` | `native` in this release |
| `state` | `ready` to offer the executor |
| `priority` | Order when several executors exist (lower first) |
| `permissions` | Required permissions, from `read_project`, `write_project`, `run_command`, `network_read` |
| `path_prefix` | For writes: the only directory the write may target, relative to the project root |
| `max_input_bytes`, `max_output_bytes` | Byte limits for the call |

Unknown permissions are refused when the catalog loads (`FABRIC_UNKNOWN_PERMISSION`). Duplicate permissions are refused (`FABRIC_DUPLICATE_PERMISSION`).

The default catalog has one entry:

| Capability | Executor | Permission | What it does |
| --- | --- | --- | --- |
| `project.read` | `rc.local.read` | `read_project` | Reads one file under the project root. Input: `path`, optional `offset_lines`, `limit_lines`. |

In guarded mode through the launcher, no command execution and no network executor are offered. Passing `--allow-command`, `--allow-http` or `--allow-write` to the launcher leaves the tool list at three. Started directly, the kernel binary serves a wider tool set; see [COMPATIBILITY.md](COMPATIBILITY.md#not-in-this-release).

## Pinning

The catalog is pinned by its SHA-256 digest, given at startup with `--fabric-server-sha256`. The kernel:

- refuses to start if the digest does not match the file;
- checks the digest again on each call and refuses with `FABRIC_CATALOG_CHANGED` if the file changed.

The launcher `prt` also checks the default catalog against the digest in `catalog/default.json.sha256`, and refuses with exit code 2 when they differ.

## Snapshot binding

A snapshot is a directory of files that describe capabilities, with a manifest that lists each file's SHA-256. You bind it at startup with `--atlas-snapshot <absolute directory>`. Through the launcher:

```sh
prt health --atlas-snapshot /absolute/path/to/snapshot
```

The kernel binds the snapshot only if all of these hold. The table gives the checks and the refusal codes observed in the 0.1.0 test.

| Check | Refusal code | Observed in the 0.1.0 test |
| --- | --- | --- |
| The snapshot path is absolute | `ATLAS_SNAPSHOT_ABSOLUTE_PATH_REQUIRED` | A relative path was refused, exit 2 |
| The snapshot directory is not a symbolic link | `ATLAS_SNAPSHOT_DIR_REFUSED` | A symbolic link was refused, exit 2 |
| The snapshot directory exists | `ATLAS_SNAPSHOT_DIR_UNAVAILABLE` | A missing directory was refused, exit 2 |
| The manifest `capability-snapshot.json` exists | `ATLAS_SNAPSHOT_MANIFEST_MISSING:capability-snapshot.json` | An empty directory was refused, exit 2 |
| The manifest parses | `ATLAS_SNAPSHOT_MANIFEST_INVALID` | Invalid JSON was refused, exit 2 |
| The manifest has the expected schema | `ATLAS_SNAPSHOT_SCHEMA` | A wrong schema was refused, exit 2 |

The other checks in the loader (the listed files and their digests, the pins, the revision and language-version fields, and the admission status) were not exercised in the 0.1.0 test. The manifest format is not published with this release, and no snapshot is shipped.

Binding is all or nothing. A snapshot that fails one check is not partly used. The health report then exits with code 2.

## Registry pin labels

`prt health` and `ps_status` print nine registry pin labels. Each label is the SHA-256 digest of one registry table that an external snapshot must supply. A snapshot is bound only when every one of its tables matches its digest (see [Snapshot binding](#snapshot-binding)). The labels are names and digests only. This release ships none of those tables and no capability content.

| Pin label | Table the snapshot supplies |
| --- | --- |
| `ANCHOR_PIN` | `anchors.tsv` |
| `BOND_PIN` | `bonds.tsv` |
| `CARD_PIN` | `cards.tsv` |
| `COUNCIL_PIN` | `council.tsv` |
| `LEXICON_PIN` | `lexicon.tsv` |
| `ONTOLOGY_PIN` | `ontology.tsv` |
| `ROUTE_PIN` | `routes.tsv` |
| `SEAT_PIN` | `seats.tsv` |
| `SKILL_PIN` | `skills.tsv` |

The digest is the SHA-256 of the table's content. Among the registry identifiers printed by health and status, these nine are the only ones observed in the 0.1.0 test.

## Behaviour without a snapshot

| Operation | Result |
| --- | --- |
| `project.read` | Works. It does not need a snapshot. |
| Any capability not in the catalog, for example `faculty.builtin` | Refused with `CAPABILITY_NOT_FOUND`. The shipped catalog does not offer `faculty.builtin`. |

`prt health` reports the snapshot state under `atlas_snapshot`. With no snapshot bound, its `atlas_capabilities` is `refused: no admitted snapshot bound` and its `source` is `absent`.

## What a snapshot does not do

- It does not add operations to the catalog. The catalog decides which operations exist.
- It does not change the startup permissions.
- It does not make any output more trustworthy. A snapshot supplies data; the receipt rules are the same.

## Writing your own catalog

To offer your own operations, write a catalog following the format above, pin it, and start the kernel with its own flags. See [EXAMPLE_GUARDED_WRITE.md](EXAMPLE_GUARDED_WRITE.md). Keep every write path under a narrow `path_prefix`.
