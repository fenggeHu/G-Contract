# G3 Contract Repository (G-Contract)

Public contract and integration repository for **content creators**. It is a **read-only mirror** published by the platform from `G_Shared`; do not edit contracts directly in this repository (all changes go through an Issue → platform contract PR).

> **Working language: English.** Please file issues, pull requests, and comments in English so all parties can collaborate. See [CONTRIBUTING.md](CONTRIBUTING.md).

## Contents
- `shared/contract/`: `capabilities.json`, `content-<ver>.json` (content Schema), entry points `def/manifest/defs-file.json`
- `shared/protocol/`, `shared/model/`: protocol IDL and shared models
- `docs/contracts/`: field-level contracts and tooling (Schema / examples / package / CLI / SDK / events / hooks / migrations / protocol / save)
- `docs/contracts/content-integration.md`: **start here for integration** — player appearance (`species`) and entering the persistent world (`enter_world`)

## Usage (no platform source required)
```bash
# 1) Get the contract (see Releases for pinned versions)
# 2) Validate: capability/schema compatibility + references/dependencies/naming/localization/licensing
content verify-release --release release.json --caps shared/contract/capabilities.json
content validate --sdk <dir> --content <your content root>   # <dir> contains capabilities.json and schema/content-<ver>.json
content lint --sdk <dir> --content <your content root>
```
Schema version `MAJOR.MINOR` is independent of the engine; the engine's supported **major** versions are listed in `capabilities.json.schemaSupported`.

## Artifact naming (Releases)
`content` (CLI), `contract.zip`, `release.json`, `engine-devkit-lite-<ver>.zip`.

## Feedback and capability requests
- Bugs / contract ambiguity / new capabilities: use this repository's issue templates (`bug` / `capability-request` / `content-handoff`).
- Change flow: Issue → platform evaluation → contract PR → release announcement (CHANGELOG) → consumers pin the version themselves.

## License and security
- Contract and example text: **Apache-2.0** (see `LICENSE`).
- For security issues, report privately per [SECURITY.md](SECURITY.md).