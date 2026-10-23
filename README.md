# s3d Smart Autopilot

English | [Українська](README.ua.md) | [Русский](README.ru.md)

A fork of [SiaFoundation/s3d](https://github.com/SiaFoundation/s3d), the S3-compatible gateway for Sia, designed to work together with **indexd Smart Autopilot**.

> **Status: release candidate.** This release has been tested on one Windows node. APIs and behavior may still change before the final release.

## Architecture

```text
S3 client
   |
   v
s3d Smart Autopilot
   |  receives redundancy policy, transfer limits, host scores and
   |  the Autopilot Mode; sends transfer measurements and usage reports
   v
indexd Smart Autopilot
   |  provides account policy, placement management
   |  and host scores
   v
Sia hosts
```

Responsibilities are intentionally separated. `indexd` provides policy values, transfer limits, host scores and the Autopilot Mode, while `s3d` uses them to validate erasure-coding schemes, manage transfers, place new data, and submit real transfer measurements. The full host-scoring and portfolio-selection system lives on the `indexd` side, not inside `s3d`.

## Release

| Parameter | Value |
|---|---|
| Release | `v0.3.0-rc.1` |
| Build version | `v0.3.0-rc.1` (commit `446c1b5`) |
| Upstream base | `s3d v0.2.1` |
| Compatible indexer | `indexd Smart Autopilot v0.3.0-rc.1` |

For user redundancy policies, transfer limits, Performance scoring, Smart placement and spending reports, use the compatible `indexd Smart Autopilot` release. With upstream `indexd`, standard behavior applies; if the indexer does not support `/hosts/speed` or `/hosts/usage`, the submission is automatically disabled until restart.

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

### Smart placement (follows the indexd Autopilot Mode)

The mode and the Smart Overall of the hosts arrive with the host list from `indexd` (refreshed every 10 minutes).

- `Original`: the host order is as before.
- `Smart Shadow`: the order is as before; the order that Smart would choose is only logged.
- `Smart Active`: new sectors are written to the hosts with the highest Smart Overall, strictly in that order (equal scores in random order). A host with recent failures is skipped and the next one in the order is taken; a host without a score goes last.
- Reading and downloading are not changed. An `indexd` without these fields works as `Original`.
- `s3d` writes its own `smart-autopilot.log` next to its data: a `smart placement mode` line when the mode is first received or changes, and a `smart placement` line per upload operation (mode, slabs, shards, chosen hosts, the hosts Smart would choose, the overlap, hosts without a score). Equal records are not repeated and a record is written at least every 30 minutes.

### Transfer measurements for Performance (G2)

`s3d Smart Autopilot` collects real measurements from successful transfers and sends them to `indexd`:

- public SDK callbacks `WithUploadProgress` and `WithDownloadProgress` are used for uploads and downloads;
- each measurement contains the host, transferred size, and transfer time;
- measurements are sent through `POST /hosts/speed`;
- transfer handlers only enqueue measurements and do not wait for the indexer response;
- background submission runs once per minute or when 500 measurements accumulate, in batches of up to 1000;
- when the network is unavailable, measurements remain buffered for later submission;
- measurements below 64 KiB are skipped because they mainly represent host response time rather than throughput.

`indexd` uses these measurements to calculate **Performance (G2)** and display raw speed values on the Hosts page.

### Usage reports for spending accounting

- `s3d` sums the successful transfers per host and direction (without the 64 KiB filter that applies to the speed measurements) and sends them once a minute through `POST /hosts/usage`: batches of up to 1000 records with an id, retried with the same id; a last report is sent on shutdown.
- `indexd` estimates the cost from the host prices and shows the spending per host on the Contracts page.

### Compatibility with upstream indexd

If the indexer does not accept `POST /hosts/speed` or `POST /hosts/usage` and returns `404` or `405`, `s3d` disables the submission until the next restart. Other functions continue to work normally.

### Status and version

- `s3d status` shows **Redundancy**, **Limits**, transfer-measurement information and a **Host usage** section (reports sent, queued, dropped, last success).
- The build version is shown in startup logs and by `s3d version`.

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

The executable will be created as `dist\s3d-<version>.exe`. For a release build, pass the release version: `pwsh -File scripts\build-fork.ps1 -ReleaseVersion v0.3.0-rc.1` (a working tree with uncommitted changes is refused).

The fork uses a local replacement in `go.mod`:

```go
replace go.sia.tech/indexd => ../indexd
```

Place compatible working `s3d` and `indexd` repositories in adjacent directories before building.

## Updating

1. Update `indexd` to `indexd Smart Autopilot v0.3.0-rc.1` first.
2. Stop `s3d` and back up its data directory.
3. Replace the executable and start `s3d`.
4. Check the startup log and run `s3d status`.
5. **`s3d` follows the `indexd` mode.** If `indexd` is in `Smart Active`, the new `s3d` starts to place by Smart Overall within 10 minutes. To watch first, switch `indexd` to `Smart Shadow` before starting `s3d`, check `smart-autopilot.log`, then switch to `Smart Active`.
6. After the first uploads or downloads through `s3d`, measurements and usage reports will reach `indexd`, and Performance will be recalculated during the next cycle (up to 10 minutes).

Always back up the data directory before upgrading.

## Known limitations

- This release candidate has been tested on one Windows node.
- The mode and the scores are refreshed with the host list (up to 10 minutes after a change).
- In `Smart Active` the order is strict, but a host that has just failed (including one that lost a race to a faster host) is skipped for a while, so a large parallel upload can spread over more hosts than the best N.
- Measurements are submitted only for successful transfers. Uploads use the winning race attempt, while downloads use the first successful shards, so scores may be slightly optimistic for slower hosts.
- Download speed mainly reflects host response time: the SDK reads part of a sector (about 100 KiB per shard), so download Mbps is useful for comparing hosts but does not represent full link capacity.
- Measurements are supplied by the client and are not protected against spoofing; `indexd` checks plausibility and packet size.
- Usage reports are an estimate: failed and interrupted transfers are not counted, and `indexd` calculates the cost from the host prices.
- Transfer limits are upper-bound estimates. Retries, parallel requests to slow shards, and completion of the final slab can increase actual traffic.
- Moving or renaming an object through S3 copy + delete may download the object's data from Sia. This behavior has not yet been investigated.

## Project direction

The overall Smart Autopilot goal is to make storage decisions configurable and understandable across all components. In this release candidate, `s3d` applies policies, places new uploads by Smart Overall in `Smart Active`, and collects transfer measurements and usage reports, while host-scoring groups **G2–G5**, Smart Overall and the portfolio plan are calculated by `indexd`.

The full seven-group model — reliability, performance, resources, economics, risk/collateral, technical compatibility, and network independence — and portfolio management beyond ranking by scores remain future development areas. `s3d` does not implement the complete host-scoring and portfolio-selection mechanism internally.

## Related projects

- [s3d upstream](https://github.com/SiaFoundation/s3d)
- [indexd Smart Autopilot](https://github.com/MainQuestion/indexd-smart-autopilot)

## License

This is an independent fork based on upstream s3d. It is not an official Sia Foundation release and is not affiliated with the Sia Foundation. Licensing terms and required notices are provided in `LICENSE`.
