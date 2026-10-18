# s3d Smart Autopilot v0.2.0-alpha.1

Second alpha release of the s3d fork: s3d now sends transfer-speed measurements to the indexer, which `indexd` uses to calculate host Performance scores.

| Parameter | Value |
|---|---|
| Build version | `v0.2.1-ba6f8c9` |
| Upstream base | `s3d v0.2.1` |
| Required | `indexd Smart Autopilot v0.2.0-alpha.1` (`v0.6.1-7838380a`) for speed scoring, user policy, and transfer limits; with upstream indexd, standard behavior applies |

## What's new

- **Transfer measurements for the indexer.** s3d subscribes to the public SDK options `WithUploadProgress` and `WithDownloadProgress` and sends the result of every successful shard transfer to indexd: host, size, and duration (`POST /hosts/speed`). indexd uses these measurements to calculate host Performance scores and display them on the Hosts page.
- **Transfers are not slowed down.** Handlers only enqueue measurements (4096 entries) and do not wait for the indexer response; excess measurements are dropped when the queue is full. Background submission runs once per minute or when 500 measurements accumulate, in batches of up to 1000. On network errors, measurements remain in a buffer (up to 5000; older excess entries are dropped) and are sent during the next cycle.
- **Measurements below 64 KiB are not sent.** Short reads mainly represent host response time rather than throughput, so they are skipped on the s3d side.
- **Upstream indexd compatibility.** If the indexer does not accept measurements (`404` or `405`), s3d disables submission until restart and sends no further requests; other functions continue to work normally.
- **`s3d status`** now includes transfer-measurement information: sent, skipped (too small), dropped, and buffered counts, plus the last successful submission time; the same data is available in administrative statistics.

## Updating

1. Update `indexd` to `indexd Smart Autopilot v0.2.0-alpha.1` first.
2. Stop `s3d` and back up its data directory.
3. Replace the executable and start `s3d`. The `erasure coding scheme accepted` and `indexer transfer limits` log messages remain as in alpha.1; transfer-submission state is visible in `s3d status`.
4. Performance scoring appears after the first uploads or downloads through s3d and the next indexd recalculation (up to 10 minutes). Hosts that have not transferred data through s3d remain at score 1.

## Known issues and notes

- **Alpha.** Tested on one Windows node.
- **Successful transfers only.** Upload scoring uses the winning race attempt; download scoring uses the first successful shards. As a result, slow hosts may be scored slightly optimistically because losing, cancelled, and timed-out attempts are not included.
- **Download speed mainly reflects response time.** The SDK reads part of a sector (about 100 KiB per shard), so download Mbps is useful for comparing hosts but does not represent full link capacity. Upload is measured using complete sectors.
- **Measurements are sent only by this fork's s3d.** The client supplies the values, and they are not protected against spoofing (indexd checks plausibility and packet size).
- Transfer limits are upper-bound estimates: retries, parallel requests to slow shards, and completion of the final slab can produce additional traffic.
- Moving or renaming an object through S3 copy + delete may download the object's data from Sia. This behavior has not yet been investigated.

## License

s3d is distributed under the MIT license (Copyright (c) The Sia Foundation); the license text is included in the release (`LICENSE-s3d.txt`). The binary also contains dependencies under their respective licenses.
