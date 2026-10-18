# s3d Smart Autopilot

English | [Українська](README.ua.md) | [Русский](README.ru.md)

A fork of [SiaFoundation/s3d](https://github.com/SiaFoundation/s3d), the S3-compatible gateway for Sia, designed to work together with **indexd Smart Autopilot**.

> **Status: alpha.** This release has been tested on one Windows node. APIs and behavior may change between alpha releases.

## Architecture

```text
S3 client
   |
   v
s3d Smart Autopilot
   |  receives redundancy policy and transfer limits
   |  and sends transfer measurements
   v
indexd Smart Autopilot
   |  provides account policy, placement management
   |  and host scores
   v
Sia hosts
```

Responsibilities are intentionally separated. `indexd` provides policy values, transfer limits, and host scores, while `s3d` uses them to validate erasure-coding schemes, manage transfers, and submit real transfer measurements. The full host-scoring and portfolio-selection system lives on the `indexd` side, not inside `s3d`.

## Release

| Parameter | Value |
|---|---|
| Release | `v0.2.0-alpha.1` |
| Build version | `v0.2.1-ba6f8c9` |
| Upstream base | `s3d v0.2.1` |
| Compatible indexer | `indexd Smart Autopilot v0.2.0-alpha.1` (`v0.6.1-7838380a`) |

For user redundancy policies, transfer limits, and Performance scoring, use the compatible `indexd Smart Autopilot` release. With upstream `indexd`, standard behavior applies; if the indexer does not support `/hosts/speed`, measurement submission is automatically disabled until restart.

## Implemented features

### Redundancy policy from the indexer

- At startup and during each upload cycle, `s3d` reads `redundancyPolicy` from the indexer's `GET /account` response.
- `sia.dataShards` and `sia.parityShards` are validated against this policy.
- User schemes such as 3-of-5 or 10-of-20 can be used when allowed by the indexer policy.
- If the indexer rejects the configured scheme, uploads pause with a clear explanation instead of entering an endless retry loop. They resume when the policy allows the scheme again.
- If the indexer does not provide a user policy, standard behavior applies.

### Transfer limits from the indexer

- `s3d` reads `transferLimits` from the indexer's `GET /account` response.
- Upload and download limits configured for the account or quota in `indexd` are applied to active transfers and refreshed every minute.
- Network limits are converted into concurrency limits. The current conversion uses upload × K/N and download ÷ 1.5 as conservative upper estimates.
- If no limits are provided, no additional rate limiting is applied.

Transfer limits are estimates rather than strict guarantees for total network traffic. Retries, parallel requests to slow shards, and completion of the final slab can increase actual traffic.

### Transfer measurements for Performance (G2)

`s3d Smart Autopilot` collects real measurements from successful transfers and sends them to `indexd`:

- public SDK callbacks `WithUploadProgress` and `WithDownloadProgress` are used for uploads and downloads;
- each measurement contains the host, transferred size, and transfer time;
- measurements are sent through `POST /hosts/speed`;
- transfer handlers only enqueue measurements and do not wait for the indexer response;
- background submission runs once per minute or when 500 measurements accumulate, in batches of up to 1000;
- when the network is unavailable, measurements remain buffered for later submission;
- measurements below 64 KiB are skipped because they mainly represent host response time rather than throughput;
- `s3d status` shows submitted, skipped, dropped, and buffered measurements, plus the last successful submission time.

`indexd` uses these measurements to calculate **Performance (G2)** and display raw speed values on the Hosts page.

### Compatibility with upstream indexd

If the indexer does not accept `POST /hosts/speed` and returns `404` or `405`, `s3d` disables measurement submission until the next restart. Other functions continue to work normally.

### Status and version

- `s3d status` shows **Redundancy**, **Limits**, and transfer-measurement information.
- The build version is shown in startup logs and by `s3d version`.
- The fork build version combines the upstream version with a short fork commit hash.

## Building from source

Requirements:

- Go 1.27
- C toolchain (MinGW-w64) for SQLite
- PowerShell 7
- A compatible `indexd Smart Autopilot` checkout in the adjacent `..\indexd` directory

Run from the s3d working directory:

```powershell
pwsh -File scripts\build-fork.ps1
```

The executable will be created as `dist\s3d-<version>.exe`.

The fork uses a local replacement in `go.mod`:

```go
replace go.sia.tech/indexd => ../indexd
```

Place compatible working `s3d` and `indexd` repositories in adjacent directories before building.

## Updating

1. Update `indexd` to `indexd Smart Autopilot v0.2.0-alpha.1` first.
2. Stop `s3d` and back up its data directory.
3. Replace the executable and start `s3d`.
4. Check the startup log and run `s3d status`.
5. After the first uploads or downloads through `s3d`, measurements will reach `indexd` and Performance will be recalculated during the next cycle (up to 10 minutes).

Always back up the data directory before upgrading.

## Known limitations

- This alpha release has been tested on one Windows node.
- Measurements are submitted only for successful transfers. Uploads use the winning race attempt, while downloads use the first successful shards, so scores may be slightly optimistic for slower hosts.
- Download speed mainly reflects host response time: the SDK reads part of a sector (about 100 KiB per shard), so download Mbps is useful for comparing hosts but does not represent full link capacity.
- Measurements are supplied by the client and are not protected against spoofing; `indexd` checks plausibility and packet size.
- Transfer limits are upper-bound estimates. Retries, parallel requests to slow shards, and completion of the final slab can increase actual traffic.
- Moving or renaming an object through S3 copy + delete may download the object's data from Sia. This behavior has not yet been investigated.

## Project direction

The overall Smart Autopilot goal is to make storage decisions configurable and understandable across all components. In this release, `s3d` applies policies and collects transfer measurements, while host-scoring groups **G2–G5** are calculated and displayed by `indexd` in Shadow mode.

The full seven-group model — reliability, performance, resources, economics, risk/collateral, technical compatibility, and network independence — and portfolio management remain future development areas. `s3d` does not implement the complete host-scoring and portfolio-selection mechanism internally.

## Related projects

- [s3d upstream](https://github.com/SiaFoundation/s3d)
- [indexd Smart Autopilot](https://github.com/MainQuestion/indexd-smart-autopilot)

## License

This is an independent fork based on upstream s3d. It is not an official Sia Foundation release and is not affiliated with the Sia Foundation. Licensing terms and required notices are provided in `LICENSE`.
