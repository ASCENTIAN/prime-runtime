# Licenses and attribution

## PrimeRuntime

The PrimeRuntime (launcher, kernel and sandbox scripts) and its documentation are licensed under the Apache License, Version 2.0.

- The full license text is in `LICENSE` at the top of the release.
- Copyright and attribution are stated in `NOTICE` at the top of the release.

## Third-party software

The release binary is built from third-party Rust crates. Each crate keeps its own license.

| Item | Where to find it |
| --- | --- |
| The list of crates the binary links, with versions | `SBOM.cdx.json` (CycloneDX), also published next to the release |
| License texts of the linked crates | `licenses/<crate>/` inside the package |
| Combined list of dependency license texts | `LICENSES-dependencies.txt` inside the package |
| Build and toolchain record | `PROVENANCE.json` inside the package and next to the release |

Every crate linked into `bin/primeshell` has a license text in `licenses/`, and the SBOM lists the linked set. In the 0.1.0 check, the linked set was nine crates: block-buffer, cfg-if, cpufeatures, crypto-common, digest, generic-array, libc, sha2 and typenum. Each has a license directory, and the SBOM's required set lists the same nine.

The SBOM lists components that are not linked into this build. Its presence does not mean that they ship in the binary.

## What this license does not cover

- **Capability content.** This release ships no capability data, snapshot, skill text, routing table or knowledge base. Nothing in the package grants a license to such content, and none is included.
- **Model weights.** None are included.
- **Your data.** Files you write through PrimeRuntime remain yours.
- **Third-party trademarks.** Product names such as Claude Code are used only to identify the client that works with PrimeRuntime. No endorsement is implied.

## Source code

The source code of the kernel is not published with 0.1.0. The Apache-2.0 grant above covers the binary and the documentation in this release.

## Reporting a license problem

Open an issue in the public repository (see its CONTRIBUTING.md). Include the file and the line. For a legal notice that cannot be public, use the contact in SECURITY.md of the public repository.
