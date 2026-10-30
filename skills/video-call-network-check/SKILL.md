---
name: video-call-network-check
description: Check whether a Mac's connection can handle Zoom, Google Meet, Teams, FaceTime or Slack calls, game streaming or remote desktops, and fix stuttering, robotic audio and frozen video. Uses Kilobat to find latency spikes and whether they repeat, and to measure jitter, packet loss, upload speed and bufferbloat, find apps uploading in the background, and check AirDrop/AWDL and Bluetooth. Use before an important call or when calls or streams break up.
---

# Video call network check with Kilobat

Calls fail differently from downloads. A connection with plenty of Mbps can still make calls stutter when latency spikes for a moment, when it jumps under load (bufferbloat), when packets are lost, or when something else is uploading. This skill checks those, in order of cost.

## Setup

Use the Kilobat MCP tools if connected, otherwise the `kilobat` CLI (`brew install seifip/kilobat/kilobat`, macOS 14+; ask before installing). Commands here need CLI 0.2 or later (`kilobat --version`), and `stutter`, `airdrop` and `bluetooth` a newer one; if one is unknown, suggest `brew upgrade kilobat`. Steps 1–3 only read; the stutter test sends pings, nothing else. The speed test in step 4 uses data and needs a yes; blocking in step 5 changes settings and needs a yes. The `kilobat` skill lists the rules.

If a call is in progress, skip the speed test: it will disrupt the call.

## 1. Check latency, jitter and loss

Run `kilobat diagnose --json`. Kilobat's call thresholds:

| Measure | Field | Calls may suffer when |
|---|---|---|
| Latency | `internetLatencyMs` | above 300 ms |
| Jitter | `internetJitterMs` | above 30 ms |
| Loss | `measurements.internet.lossPercent` | above 2% |
| Wi-Fi | `signalScore`, `routerLatencyMs` | score under 50, router above 20 ms |

Jitter and loss matter more than speed: audio breaks up at a few percent loss even on a fast line. If `routerLatencyMs` or router loss is high, the problem is between the Mac and the router; see `mac-wifi-troubleshooting`. Run diagnose two or three times about 20 seconds apart, because call problems are often intermittent.

## 2. Look for latency spikes before blaming bandwidth

Diagnose and the speed test average a handful of pings, so a 300 ms stall once a second disappears in them, yet it freezes video and makes game streams and remote desktops stutter. Run `kilobat stutter --json` (MCP: `run_stutter_test`). It pings the router every 50 ms for 20 seconds; `--target internet` pings 1.1.1.1 instead. Read:

- `latencyMs.p50` against `p99` and `max`: a low median with a high p99 means short spikes, not a slow line.
- `spikes` (each over 3× the median and over 50 ms) with `atSeconds`, and `pattern`: `steady`, `few`, `periodic`, `irregular` or `noReplies`.
- `periodSeconds`: set when the spikes repeat at a regular interval, such as about every second.
- `awdl.usedBy`: what used the AirDrop link during the test (AirDrop, Universal Control, Sidecar, iPhone Mirroring, Continuity).
- `pingRateLimited`: many routers answer only about ten pings a second. Then `lossPercent` is not real loss; don't report it as such.

Quote the `verdict`. How to read the pattern:

- Periodic spikes on the router test: they start between the Mac and the router. A common cause is AWDL, the link behind AirDrop and Universal Control, hopping channels on a schedule; one user's GreenLight test went from 450–500 ms spikes to none after pausing it. It is a lead, not a diagnosis: other periodic scans and power saving can look the same. Test with the low-latency pause (below) before concluding.
- Irregular spikes on the router: a busy network, interference or weak signal; continue with `mac-wifi-troubleshooting` and step 3.
- Spikes on the internet test but not the router: beyond the user's network; the ISP or the route.
- No spikes on either while the call still stutters: the cause is likely in the app, the other side or the service.

`kilobat bluetooth --json` (MCP: `get_nearby_radios`) shows Bluetooth headphones, keyboards and mice that share 2.4 GHz with Wi-Fi; when `wifiBand` is 2.4 GHz and one is connected, suggest a 5 GHz network.

## 3. Find uploads competing with the call

Run `kilobat status --json` and sort `apps` by `uploadBytesPerSecond`. Calls need steady upload; a sync or backup can take all of it. Typical culprits: iCloud Photos and iCloud Drive, Dropbox, Google Drive, OneDrive, Time Machine to a network disk, cloud backups, and large email attachments. Run it twice, 30 seconds apart, since sync works in bursts.

## 4. Measure capacity and bufferbloat, with approval

Ask first: the test uses up to 80 MB and takes up to about 30 seconds. On a hotspot use `--budget-mb 10`.

Run `kilobat speedtest --json`. Read:

- `measurements.downloadMbps` and `measurements.uploadMbps`.
- `addedLatencyMs`: how much latency rose while the line was busy. `null` means the test could not load the line long enough to tell.
- `bufferbloat`: `Low` (under 15 ms added), `Moderate` (under 60), `High` (60 or more) or `Inconclusive`. Added latency above 100 ms makes calls stutter whenever anything else downloads or uploads.

Kilobat's own verdicts are in the same JSON: `voiceCalls`, `videoCalls` (the highest of 1080p, 720p or SD the bandwidth supports) and `streaming`, each with "likely suitable" or "may stutter", plus `callWarnings` explaining any caution. Quote them. The bandwidth tiers behind them (download / upload, in Mbps):

| Call quality | Download | Upload |
|---|---|---|
| 1080p video | 3 | 3.8 |
| 720p video | 1.8 | 2.6 |
| SD video | 0.6 | 1 |
| Voice only | 0.1 | 0.1 |

For group calls with many video tiles, more download helps.

## 5. Recommend

Lead with the measurement that explains the symptom, then at most three fixes, most likely first.

- Something uploading: pause it from its own menu, or, with approval, block it for the length of the call: `kilobat block <app id> --minutes 90`. With the Kilobat app's filter on (Kilobat Pro) and the Travel or Work profile active, this stops new connections; otherwise the rule is only saved. Remind the user that the rule expires on its own.
- High bufferbloat: the router's queue fills under load. Turn on SQM, Smart Queue or QoS in the router if it has it, or keep big transfers off during calls.
- Weak Wi-Fi or high router latency: move closer, use 5 GHz, or plug in Ethernet.
- High latency or loss beyond the router with good Wi-Fi: it is past the user's network. Suggest turning off video, switching to audio-only, or using a phone hotspot as a backup.
- VPN on (`kilobat ip --json` says `vpn` full when the default route goes through it, partial when it carries only some destinations): latency includes the tunnel; try the call without it if allowed. `kilobat speed-compare --json` shows whether an earlier, better test ran without it.

- Periodic spikes on the router: try pausing AWDL (AirDrop, Sidecar, Universal Control). In the Kilobat app (Pro), Health › Stutter test › Compare with low-latency on runs the test again with it paused and says whether it helped. From the CLI, the user runs `kilobat low-latency on --for 1h` or `--until-quit <app>` (the call or game) in their own Terminal during the call; it asks for administrator approval, lasts 4 hours at most, and restores on expiry, when that app quits, on Ctrl-C or a network change. Run `kilobat stutter` in another window before and during the pause and compare. If the spikes stay, AWDL isn't the cause; say so and restore it with `kilobat low-latency off`.

After a change, run diagnose again and report before and after. `kilobat speed-history` lists earlier speed tests, and `kilobat health-history --hours 24` shows whether jitter or loss came and went during the day.
