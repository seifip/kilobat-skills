---
name: video-call-network-check
description: Check whether a Mac's connection can handle Zoom, Google Meet, Teams, FaceTime or Slack calls, and fix stuttering, robotic audio and frozen video. Uses Kilobat to measure latency, jitter, packet loss, upload speed and bufferbloat, and to find apps uploading in the background. Use before an important call or when calls break up.
---

# Video call network check with Kilobat

Calls fail differently from downloads. A connection with plenty of Mbps can still make calls stutter when latency jumps under load (bufferbloat), when packets are lost, or when something else is uploading. This skill checks those, in order of cost.

## Setup

Use the Kilobat MCP tools if connected, otherwise the `kilobat` CLI (`brew install seifip/kilobat/kilobat`, macOS 14+; ask before installing). Steps 1–2 only read. The speed test in step 3 uses data and needs a yes; blocking in step 4 changes settings and needs a yes. The `kilobat` skill lists the rules.

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

## 2. Find uploads competing with the call

Run `kilobat status --json` and sort `apps` by `uploadBytesPerSecond`. Calls need steady upload; a sync or backup can take all of it. Typical culprits: iCloud Photos and iCloud Drive, Dropbox, Google Drive, OneDrive, Time Machine to a network disk, cloud backups, and large email attachments. Run it twice, 30 seconds apart, since sync works in bursts.

## 3. Measure capacity and bufferbloat, with approval

Ask first: the test uses up to 80 MB and takes up to about 30 seconds. On a hotspot use `--budget-mb 10`.

Run `kilobat speedtest --json`. Read:

- `measurements.downloadMbps` and `measurements.uploadMbps`.
- `addedLatencyMs`: how much latency rose while the line was busy. `null` means the test could not load the line long enough to tell.
- `bufferbloat`: `Low` (under 15 ms added), `Moderate` (under 60), `High` (60 or more) or `Inconclusive`. Added latency above 100 ms makes calls stutter whenever anything else downloads or uploads.

The JSON does not include a call verdict; work it out from these bandwidth tiers (download / upload, in Mbps):

| Call quality | Download | Upload |
|---|---|---|
| 1080p video | 3 | 3.8 |
| 720p video | 1.8 | 2.6 |
| SD video | 0.6 | 1 |
| Voice only | 0.1 | 0.1 |

Name the highest tier the result meets, then lower the verdict to "may stutter" if any threshold from step 1 or the added latency is exceeded. For group calls with many video tiles, more download helps.

## 4. Recommend

Lead with the measurement that explains the symptom, then at most three fixes, most likely first.

- Something uploading: pause it from its own menu, or, with approval, block it for the length of the call: `kilobat block <app id> --minutes 90`. With the Kilobat app's filter on (Kilobat Pro) and the Travel or Work profile active, this stops new connections; otherwise the rule is only saved. Remind the user that the rule expires on its own.
- High bufferbloat: the router's queue fills under load. Turn on SQM, Smart Queue or QoS in the router if it has it, or keep big transfers off during calls.
- Weak Wi-Fi or high router latency: move closer, use 5 GHz, or plug in Ethernet.
- High latency or loss beyond the router with good Wi-Fi: it is past the user's network. Suggest turning off video, switching to audio-only, or using a phone hotspot as a backup.
- VPN on (`kilobat interfaces --json` lists VPN interfaces): latency includes the tunnel; try the call without it if allowed.

After a change, run diagnose again and report before and after. `kilobat speed-history` lists earlier speed tests for comparison.
