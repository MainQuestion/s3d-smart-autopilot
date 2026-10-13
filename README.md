# s3d Smart Autopilot

[English](README.md) | [Українська](README.ua.md) | [Русский](README.ru.md)

A fork of [SiaFoundation/s3d](https://github.com/SiaFoundation/s3d), the S3-compatible gateway for Sia, built to work with **indexd Smart Autopilot**.

> **Status: alpha.** This release has been tested on one Windows node. APIs and behavior may change between alpha releases.

## Architecture

```text
S3 client
   |
   v
s3d Smart Autopilot
   |  consumes redundancy policy and transfer limits
   v
indexd Smart Autopilot
   |  provides account policy and storage-placement controls
   v
Sia hosts
```

The responsibilities are deliberately split between the components. `indexd` provides the policy and transfer-limit values; `s3d` consumes them when validating its erasure-coding scheme and controlling transfers. This alpha does not implement the complete host-scoring or host-portfolio selection system inside `s3d`.

## Release

| Item | Value |
|---|---|
| Release | `v0.1.0-alpha.1` |
| Build version | `v0.2.1-a538024` |
| Upstream base | `s3d v0.2.1` |
| Companion indexer | `indexd Smart Autopilot v0.1.0-alpha.1` (`v0.6.1-9fd0bf52`) |

For custom redundancy policies and transfer limits, use the compatible `indexd Smart Autopilot` release. With upstream `indexd`, which does not report the custom policy values, standard behavior applies.

## Implemented features

### Redundancy policy from the indexer

- On startup and during every upload cycle, `s3d` reads `redundancyPolicy` from the indexer's `GET /account` response.
- The configured `sia.dataShards` and `sia.parityShards` values are checked against the policy.
- Custom schemes, such as 3-of-5 or 10-of-20, can be accepted when permitted by the indexer policy.
- If the indexer rejects the configured scheme, uploads pause with a clear reason instead of entering a retry loop. They resume when the policy permits the scheme again.
- If the indexer does not report a custom policy, standard behavior applies.

### Transfer limits from the indexer

- `s3d` reads `transferLimits` from the indexer's `GET /account` response.
- Upload and download limits configured for the account or quota in `indexd` are applied to running transfers and refreshed once per minute.
- Network limits are converted to stream limits. The current conversion uses upload limit × K/N and download limit ÷ 1.5 as a conservative upper bound.
- If no limits are reported, no additional transfer limit is applied.

Transfer limits are estimates, not strict guarantees of total network traffic. Retries, racing requests for slow shards, and padding of the final slab can increase actual traffic.

### Status and version reporting

- `s3d status` displays **Redundancy** and **Limits** sections, including the active values and their source.
- The build version is shown in the startup log and by `s3d version`.
- The fork build version combines the upstream version with the short fork commit, for example `v0.2.1-a538024`.

## Build from source

Requirements:

- Go 1.27
- A C toolchain (MinGW-w64) for SQLite
- PowerShell 7
- A compatible `indexd` Smart Autopilot checkout in the sibling directory `..\indexd`

Run from the s3d working directory:

```powershell
pwsh -File scripts\build-fork.ps1
```

The executable is written to `dist\s3d-<version>.exe`.

The fork uses a local replacement in `go.mod`:

```go
replace go.sia.tech/indexd => ../indexd
```

Clone the compatible `s3d` and `indexd` working repositories side by side before building.

## Upgrade from upstream s3d

1. Upgrade `indexd` first if required by the release.
2. Stop `s3d`.
3. Back up the s3d data directory. Upstream `s3d v0.2.1` migrates its SQLite database.
4. Replace the executable and start `s3d`. Existing configuration and keys should remain unchanged.
5. Check the startup log and run `s3d status`. Look for messages such as `erasure coding scheme accepted` and `indexer transfer limits`.

Always keep a backup of the data directory before upgrading.

## Known limitations

- Alpha testing has been performed on one Windows node only.
- Transfer limits are estimates. Retries, slow-shard races, and padding of the final slab may increase actual traffic.
- Moving or renaming an object through S3 copy-and-delete may download the object data from Sia. This behavior has not yet been investigated.

## Project direction

The broader Smart Autopilot project aims to make storage decisions configurable and explainable across components. Its design direction includes separate host-evaluation dimensions—reliability, performance, resources, economics, collateral/risk, technical compatibility, and network diversity—and portfolio-level placement instead of relying on one opaque score.

These are project goals, not features implemented by this s3d alpha. In this release, `s3d` consumes policies provided by the indexer; it does not implement the complete host-scoring or portfolio-selection system itself.

## Related projects

- [s3d upstream](https://github.com/SiaFoundation/s3d)
- [indexd Smart Autopilot](https://github.com/MainQuestion/indexd-smart-autopilot)

## License

This is an independent fork based on the upstream s3d project and is not an official Sia Foundation release or affiliated with the Sia Foundation. See the repository's `LICENSE` file for the license terms and required notices.
