# s3d Smart Autopilot

English | [Українська](README.ua.md) | [Русский](README.ru.md)

A fork of [SiaFoundation/s3d](https://github.com/SiaFoundation/s3d), the S3-compatible gateway for Sia, designed to work together with **indexd Smart Autopilot**.

> **Status: release candidate.** This release has been tested on two Windows machines (a work PC and the remote location). The API and behavior may change before the final release.

## Architecture

```text
S3 client
   |
   v
s3d Smart Autopilot
   |  receives the redundancy policy, transfer limits, host scores
   |  and Autopilot Mode; sends transfer measurements, usage
   |  reports and transfer attempt summaries
   v
indexd Smart Autopilot
   |  provides the account policy, placement control
   |  and host scores
   v
Sia hosts
```

The responsibilities are deliberately separated. `indexd` provides policy values, transfer limits, host scores and Autopilot Mode, and `s3d` uses them to validate erasure coding schemes, manage transfers, place new data and send real transfer measurements. The full host scoring and portfolio selection system lives on the `indexd` side, not inside `s3d`.

## Release

| Parameter | Value |
|---|---|
| Release | `v0.3.2-rc.1` |
| Build version | `v0.3.2-rc.1` (commit `75de75a`) |
| Upstream base | `s3d v0.3.0` (merged into the fork on the Go SDK); previous base `s3d v0.2.1` |
| Client library | our `sia-storage-go` fork, branch `next/host-errors`, commit `007f415` (on top of `f9961f7`) |
| Compatible indexer | `indexd Smart Autopilot v0.4.1-rc.1` |
| Previous release | `v0.3.1-rc.1` |

For custom redundancy policies, transfer limits, Performance scoring, Smart placement, spending reports and the host work share report, use a compatible `indexd Smart Autopilot` release. With upstream `indexd` the standard behavior applies; if the indexer does not support `/hosts/speed` or `/hosts/usage`, sending is switched off automatically until restart.

## What's new in v0.3.2-rc.1

- **Merge of upstream `s3d v0.3.0`** on the Go SDK: scheduled snapshots, object lock, checksums, bucket policies, tags, POST Object, user and key management through the admin API (details below).
- **Database:** migrations 13 → 20, forward-only. Rolling back to the previous exe requires restoring `s3d.db` from a backup (see "Updating").
- Fixed a defect: `WithUploadProgress` used one shared observer for several uploads.
- Our features from `v0.3.1-rc.1` (write modes, buffers, summaries of host attempts and errors) are unchanged.

Details, the list of files with hashes and the update procedure: the release notes [s3d Smart Autopilot v0.3.2-rc.1](https://github.com/MainQuestion/s3d-smart-autopilot/releases/tag/v0.3.2-rc.1).

## Implemented features

### Upstream s3d v0.3.0 features (new)

The list is compiled from the upstream commit list between `v0.2.1` and `v0.3.0`; we have not fully verified the behavior details (see "Known limitations").

- **Snapshots:** a cron-style schedule with calendar retention (set through the admin API and the command line), the `snapshots restore` command; buffered objects are left out of the snapshot image; lost uploads are removed on the first start after a restore.
- **Object lock:** bucket configuration, retention and legal hold, protection from deletion.
- **Bucket policies, tags** for buckets and objects.
- **POST Object** (form and policy).
- **Checksums:** `x-amz-checksum-*` for PutObject and UploadPart, composite checksums of multipart uploads, verification on completion.
- **Lifecycle:** expiry of noncurrent versions and expired delete markers, object size filters.
- **Users and access keys** are managed through the admin API (the `users` and `keys` commands work through it; upstream marked the change as breaking). User names are restricted to the AWS IAM character set, access key IDs to the AWS set, responses to invalid requests match AWS.
- Automatic object flush at a configurable interval, parallel pinning of uploaded objects, the upload rate is derived from shard progress.

**Not carried over** (needs the Rust SDK): registration from the environment (`-env`) and pre-authorized keys in login.

### Network write modes

Selected in `s3d.yml`; no rebuild needed.

| Mode | Settings | How it works |
|---|---|---|
| **Default** | (nothing) | as in the original: write with races (a spare attempt on another host if the first is slow) |
| **No races** | `uploadRacing: false` and `uploadAttemptTimeoutSeconds: 30` | a shard is written to one host; if it does not finish in time, another host is taken. No extra traffic to the "losing" hosts |
| **Limited hedging** | `uploadHedgeBudget: 0.05` | an extra attempt is started only for noticeably slow writes: the wait time is taken from the host's write history (Kaplan–Meier estimate), and the share of such attempts is limited by the budget (5% of first attempts per minute) |

Allowed values: `uploadAttemptTimeoutSeconds` from 5 to 300 (`0` means the SDK default, 90 seconds); `uploadHedgeBudget` from 0.01 to 0.2 (`0` is off). With `uploadRacing: false` there is no hedging (switched-off racing has the priority) and a warning is written to the log.

In our measurements on one PC with three hosts, the best result came from the **no-races mode with a buffer of 4 slabs**. The result depends on your hosts and connection, so the mode is worth choosing by measurement.

### Buffer settings

- `uploadMaxBufferedSlabs`: how many slabs are held in memory at once during an upload. This is the key speed parameter: more is faster, but needs more memory and puts more simultaneous load on hosts. `0` keeps the SDK default (about 10% of the process memory).
- `downloadMaxBufferedChunks`: the same for downloads (chunks of up to 1 MiB).

An example `s3d.yml` fragment we used:

```yaml
uploadThreads: 1
uploadRacing: false
uploadAttemptTimeoutSeconds: 30
uploadMaxBufferedSlabs: 4
```

### Redundancy policy from the indexer

- At startup and in every upload cycle `s3d` reads `redundancyPolicy` from the indexer's `GET /account` response.
- `sia.dataShards` and `sia.parityShards` are checked against this policy.
- Custom schemes, for example 3-of-5 or 10-of-20, can be used when the indexer policy allows them.
- If the indexer rejects the configured scheme, uploads are paused with a clear explanation instead of an endless retry loop. They resume when the policy allows the scheme again.
- If the indexer does not provide a custom policy, the standard behavior applies.

### Transfer limits from the indexer

- `s3d` reads `transferLimits` from the indexer's `GET /account` response.
- The upload and download limits set for an account or quota in `indexd` are applied to active transfers and refreshed every minute.
- Network limits are converted into concurrency limits. The current conversion uses upload × K/N and download ÷ 1.5 as cautious upper estimates.
- If no limits are provided, there is no extra speed limit.

Transfer limits are estimates, not strict guarantees of the total network traffic. Retries, parallel requests to slow shards and the completion of the last slab can increase the actual traffic.

### Smart placement (follows the indexd Autopilot Mode)

The mode and the Smart Overall of the hosts arrive with the host list from `indexd` (refreshed every 10 minutes).

- `Original`: the host order is as before.
- `Smart Shadow`: the order is as before; the order Smart would have chosen is only logged.
- `Smart Active`: new sectors are written to the hosts with the highest Smart Overall, strictly in this order (equal scores in random order). A host with recent failures is skipped and the next one in order is taken; a host without a score goes last.
- Reading and downloading do not change. `indexd` without these fields works as `Original`.
- `s3d` writes its own `smart-autopilot.log` next to its data: a `smart placement mode` line when the mode is first received or changes, and a `smart placement` line for each upload operation (mode, slabs, shards, chosen hosts, hosts Smart would have chosen, match, hosts without a score). Identical entries are not repeated, and an entry is written at least once every 30 minutes.

### Transfer measurements for Performance (G2)

`s3d Smart Autopilot` collects real measurements of successful transfers and sends them to `indexd`:

- the public SDK callbacks `WithUploadProgress` and `WithDownloadProgress` are used for uploads and downloads;
- each measurement contains the host, the transfer size and the transfer time;
- measurements are sent through `POST /hosts/speed`;
- the transfer handlers only queue measurements and do not wait for the indexer's answer;
- background sending runs once a minute or when 500 measurements have accumulated, in batches of up to 1000;
- when the network is unavailable, measurements stay in the buffer for later sending;
- measurements below 64 KiB are skipped, because they mostly reflect the host response time and not throughput.

`indexd` uses these measurements to compute **Performance (G2)** and to show the raw speed values on the Hosts page.

### Usage reports for spending accounting

- `s3d` sums successful transfers per host and direction (without the 64 KiB filter that applies to speed measurements) and sends them once a minute through `POST /hosts/usage`: batches of up to 1000 records with an identifier, a retry with the same identifier; the last report is sent on shutdown.
- `indexd` estimates the cost from the host prices and shows the spending per host on the Contracts page.

### Transfer attempt summaries for indexd

For each host and direction (read or write) `s3d` counts attempts and their outcomes and sends a summary to `indexd` once a minute:

- attempts (only transfers of whole sectors of 4 MiB; shorter ones are counted separately as `shortAttempts`);
- lost races, timeouts, connection failures;
- **host error responses** (`hostErrors`): the host itself answered with an error ("not enough storage", "internal error", "sector not found" and the like). Client-side errors (a bad request, payment, expired prices) and transport failures are not included.

`indexd` uses this data in the host work share report and for diagnostics. The effect on transfer speed is negligible (hundreds of nanoseconds per event).

### Client library changes

`s3d` embeds our `sia-storage-go` branch (`next/host-errors`, commit `007f415` on top of `f9961f7`) with three changes:

- transfer attempt events (success, timeout, cancellation, loss, connection failure, host error) are passed to the application;
- the `ConnectError` type distinguishes a failure to connect to a host from errors after the connection is established;
- upload improvements on which the write modes above rely.

### Compatibility with upstream indexd

If the indexer does not accept `POST /hosts/speed` or `POST /hosts/usage` and returns `404` or `405`, `s3d` switches sending off until the next restart. The other features keep working normally.

### Status and version

- `s3d status` shows **Redundancy**, **Limits**, information about transfer measurements, the **Host usage** section (reports sent, queued, dropped, last success), the **Attempts counted** and **Host errors** lines broken down by host and direction, as well as the buffer values and write attempts.
- The build version is shown in the startup logs and by the `s3d version` command.

## Building from source

Requirements:

- Go 1.27
- A C toolchain (MinGW-w64) for SQLite
- PowerShell 7
- A compatible `indexd Smart Autopilot` checkout in the neighboring directory `..\indexd` and our `sia-storage-go` fork in `..\sia-storage-go` (or the paths through `-IndexdRepo` and `-SdkRepo`)

Run from the s3d working directory:

```powershell
pwsh -File scripts\build-fork.ps1
```

The executable is created as `dist\s3d-<version>.exe`. For a release build pass the release version: `pwsh -File scripts\build-fork.ps1 -ReleaseVersion v0.3.2-rc.1` (a working tree with uncommitted changes is rejected). Other paths to `indexd` and `sia-storage-go` are set with the `-IndexdRepo` and `-SdkRepo` parameters.

The fork uses a local replacement in `go.mod`:

```go
replace go.sia.tech/indexd => ../indexd
```

Before building, place compatible working repositories of `s3d` and `indexd` in neighboring directories.

## Updating

1. First update `indexd` to `indexd Smart Autopilot v0.4.1-rc.1`.
2. Stop `s3d` and back up `s3d.exe`, `s3d.yml` and the data directory, **including `s3d.db` (and `s3d.db-wal`, `s3d.db-shm` if present)**. If `s3d` runs as a service with dependent services (for example a disk mount through rclone), stop and start them in the right order: `s3d` first, then the dependent ones.
3. Replace the executable and start `s3d`. There are no new mandatory settings; the old `s3d.yml` works as is. The log should show the database migrations (13 → 20) and a clean start.
4. Check the startup log and run `s3d version` and `s3d status`.
5. **`s3d` follows the `indexd` mode.** If `indexd` is in `Smart Active`, the new `s3d` starts placing by Smart Overall within 10 minutes. To look first, switch `indexd` to `Smart Shadow` before starting `s3d`, check `smart-autopilot.log`, then switch to `Smart Active`.
6. After the first uploads or downloads through `s3d`, the measurements and usage reports reach `indexd`, and Performance is recomputed in the next cycle (up to 10 minutes).

**Rollback.** Database migrations are forward-only, so you cannot roll back by replacing `s3d.exe` alone: you need to restore the previous `s3d.exe` **and restore `s3d.db` from the backup**. Data written after the update is lost on rollback.

Always back up the data directory before updating.

## Known limitations

- This release candidate has been tested on two Windows machines.
- **Not verified:** the s3-tests suite (tox), `snapshots restore`, scheduled snapshots on live data, object lock, bucket tags and policies, multipart upload with checksums, the behavior of `uploadHedgeBudget` under load.
- Our changes are based on `sia-storage-go` (`f9961f7`). In upstream `s3d v0.3.0` the transport engine was replaced with Rust (`sia-sdk-rs` through cgo), and our patches do not carry over to it (which is why `-env` and pre-authorized keys are not carried over): as long as we stay on the Go SDK, updating the library is not possible without rework.
- The write modes were tuned on one system of three hosts; results may differ on other configurations.
- The mode and scores are refreshed together with the host list (up to 10 minutes after a change).
- In `Smart Active` the order is strict, but a host that has just failed (including one that lost a race to a faster one) is skipped for a while, so a large parallel upload can spread over more hosts than the best N.
- Measurements are sent only for successful transfers. Uploads use the winning attempt of a race and downloads use the first successful shards, so the scores may be slightly optimistic for slower hosts.
- Download speed mostly reflects the host response time: the SDK reads part of a sector (about 100 KiB per shard), so download Mbit/s are useful for comparing hosts but do not reflect the full throughput of the connection.
- Measurements are supplied by the client and are not protected against forgery; `indexd` checks plausibility and batch size.
- Usage reports are an estimate: failed and interrupted transfers are not counted, and `indexd` computes the cost from the host prices.
- Transfer limits are upper estimates. Retries, parallel requests to slow shards and the completion of the last slab can increase the actual traffic.
- `s3d` first accepts the file and sends it to the hosts afterwards; a client speed limit (for example rclone `--bwlimit`) applies only to the leg up to `s3d`.
- Moving or renaming an object through S3 copy + delete may download the object data from Sia. This behavior has not been investigated yet.

## Project direction

The general goal of Smart Autopilot is to make storage decisions configurable and understandable in all components. In this release candidate `s3d` applies policies, places new uploads by Smart Overall in `Smart Active`, offers configurable write modes and collects transfer measurements, usage reports and attempt summaries, while the host score groups **G2–G5**, Smart Overall and the portfolio plan are computed by `indexd`.

The full seven-group model — reliability, performance, resources, economics, risk/collateral, technical compatibility and network independence — and portfolio management beyond ranking by scores remain directions for future development. `s3d` does not implement the full host scoring and portfolio selection mechanism inside itself.

## Related projects

- [s3d upstream](https://github.com/SiaFoundation/s3d)
- [indexd Smart Autopilot](https://github.com/MainQuestion/indexd-smart-autopilot)

## License

This is an independent fork based on upstream s3d. It is not an official Sia Foundation release and is not affiliated with the Sia Foundation. The licensing terms and required notices are in `LICENSE`.
