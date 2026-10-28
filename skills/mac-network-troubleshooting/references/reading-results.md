# Reading Kilobat results

These thresholds are the ones Kilobat itself uses to compute `health` and `advice`. Quote Kilobat's `advice` strings when they apply; use the numbers here to explain them.

## `kilobat diagnose --json`

| Field | Meaning |
|---|---|
| `health` | `poor` if a captive portal is detected, the Wi-Fi signal is poor, or internet loss is 20% or more. Otherwise `degraded` if the signal is degraded, router latency is above 20 ms, internet latency above 120 ms, DNS above 150 ms or jitter above 30 ms. If neither condition applies and internet ping has no average, `unknown`; otherwise `good`. A lack of ping replies does not by itself establish an outage. |
| `advice` | Plain-language findings, one per detected issue. |
| `signalScore` | 0–100, computed as `(rssi + 90) × 2`: −40 dBm scores 100, −65 dBm scores 50 and −90 dBm scores 0. `null` on Ethernet. |
| `signalNoiseRatio` | `rssi − noise` in dB. Higher is better. |
| `routerLatencyMs` | Mean ping to the default gateway. Wi-Fi to a nearby router is usually under 10 ms. |
| `internetLatencyMs` | Mean ping to 1.1.1.1. |
| `internetJitterMs` | Mean difference between consecutive internet pings. |
| `measurements.radio` | `rssi` and `noise` in dBm, `channel`, `band` (2.4, 5 or 6 GHz), `transmitMbps` (negotiated link rate), `interface`. `ssid` may be absent without Location permission. |
| `measurements.router`, `measurements.internet` | `host`, `samples` in ms, `lossPercent`. Empty `samples` means no replies. Some networks block ICMP, so this alone does not prove an outage. |
| `measurements.dnsMilliseconds` | `dig` query time for example.com through the system resolver. Near 0 means a cached answer, so repeat if it matters. |
| `measurements.dnsServers` | Current resolvers. Record them before changing DNS. |
| `measurements.captivePortal` | `true` if the network intercepts traffic for a login page. |
| `measurements.networkService` | macOS network service name, for example `Wi-Fi`. DNS changes apply to it. |

### Wi-Fi signal

| RSSI | SNR | Kilobat rating |
|---|---|---|
| −65 dBm or stronger | 25 dB or more | good |
| −65 to −75 dBm | 15–25 dB | degraded |
| weaker than −75 dBm | under 15 dB | poor |

The weaker of the two decides. A low `transmitMbps` with a good RSSI points to interference or an old router; 2.4 GHz is slower and more crowded than 5 or 6 GHz but reaches further.

### Reading layers together

- Slow router and slow internet: fix the local link first; it adds to everything beyond it.
- Fast router, slow or lossy internet: the test suggests a problem beyond the local link, but a single destination, ICMP filtering or a VPN can also explain it. Compare targets and networks before attributing it to a provider.
- Good latency but slow pages: suspect DNS (`dnsMilliseconds`) or a specific site (`http-check`).
- Good everything but calls stutter: check jitter, then bufferbloat with a speed test.

## `kilobat speedtest --json`

| Field | Meaning |
|---|---|
| `measurements.downloadMbps`, `measurements.uploadMbps` | Throughput to Cloudflare. |
| `measurements.baseline` | Idle ping samples before the test. |
| `measurements.downloadLoaded`, `measurements.uploadLoaded` | Ping samples while the link was busy. |
| `measurements.transferredBytes` | Data used by the test. |
| `addedLatencyMs` | Loaded latency minus idle latency. `null` if the test could not load the link for at least 2 seconds or the idle ping was unreliable. |
| `bufferbloat` | `Low` under 15 ms added, `Moderate` under 60 ms, `High` at 60 ms or more, `Inconclusive` without a reliable `addedLatencyMs`. |
| `voiceCalls`, `videoCalls`, `streaming` | Kilobat's verdicts, such as `1080p · likely suitable` or `720p · may stutter`. |
| `callWarnings` | Why a call verdict says "may stutter", for example high jitter or delay under load. |

Calls may stutter when idle latency is above 300 ms, jitter above 30 ms, loss above 2% or added latency above 100 ms. Bandwidth needs for video calls: 1080p about 3 Mbps down and 3.8 up, 720p 1.8 and 2.6, SD 0.6 and 1. Streaming: 4K about 15 Mbps, 1080p 5, 720p 3.

## `kilobat status --json`

| Field | Meaning |
|---|---|
| `connected`, `network` | Path status and kind: `Wi-Fi`, `Ethernet`, `Cellular` or `Network`. |
| `metered` | macOS marks the path as expensive, for example a phone hotspot. |
| `profile` | `travel`, `work` or `unrestricted`. |
| `downloadBytesPerSecond`, `uploadBytesPerSecond` | Whole-Mac rates over the last sample. |
| `sessionBytes` | Bytes since this command started sampling. Use `report` for history. |
| `apps[]` | `id` (bundle ID or `process:<name>`), `name`, per-app rates, `sessionBytes`, `allowedRule` (the saved rule, not proof of enforcement). |
| `processIDs` | Process IDs behind each app, for `destinations --pid` or `quit`. |
| `filteringActive` | `true` or `false` from the running Kilobat app; `null` when the app is not running. |
| `ssid`, `sessionBudgetGB`, `budgetUsed` | From the running app: the network name, this network's budget and the share used (0–1). |

## `kilobat report --summary --json [--days N]`

`downloadBytes` and `uploadBytes` totals, `daily[]` per calendar day and `topApps[]` (10 apps by bytes). History is recorded while the Kilobat app is running; gaps can have several causes and do not establish that no data was used.

## `kilobat radar --json`

One sample: `radio` (as in diagnose), `score` (0–100) and `signalNoiseRatio`. Compare samples taken in different spots; a gain of 10 points or more is meaningful.

## `kilobat scan --json`

Array of `ssid`, `rssi`, `noise`, `channel`, `band`, `widthMHz`. Several strong networks on the user's channel indicate congestion. On 2.4 GHz only channels 1, 6 and 11 don't overlap. Wide channels (80 or 160 MHz) are faster but more exposed to neighbors.
