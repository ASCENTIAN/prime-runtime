# Contributing

Thank you for looking at PrimeRuntime.

## Current state

The source code is not published with version 0.1.0. You can use the released package, report problems and suggest documentation changes.

- **Issues:** welcome. Open one for a bug, an unclear document, a refusal you do not understand, or a platform you would like supported.
- **Documentation fixes:** welcome as an issue that quotes the sentence and proposes the change.
- **Pull requests:** not accepted yet. Open an issue before you send any change.
- **Security problems:** do not use issues. Follow [SECURITY.md](SECURITY.md).

## What to include in an issue

- Output of `prt version`.
- Output of `prt health; echo "exit=$?"`, with any path you consider private removed.
- The command you ran, the result you expected, and the result you got.
- The refusal code, if there is one. Find it in [the refusal table](docs/TROUBLESHOOTING.md#refusal-codes).
- Your distribution and kernel version.

Do not include file content, receipts that contain content, credentials or private keys.

## Documentation conventions

- Write for a reader who has not seen the project before. Define every term, or link to the [glossary](docs/GLOSSARY.md).
- Give every new capability claim the command that checks it.
- Use the same field and error names as the released binary.

## Licensing

PrimeRuntime and documentation are licensed under Apache-2.0. See [LICENSES](docs/LICENSES.md). Contributions, when they are accepted, will be under the same license.
