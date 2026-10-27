---
name: mac-network-troubleshooting
description: Diagnose slow or unreliable internet on a Mac with Kilobat's CLI or MCP server. Measures Wi-Fi signal, router and internet latency, packet loss, jitter, DNS, captive portals and bufferbloat, finds which layer is failing and suggests one fix at a time. Use when the internet is slow, drops or lags and the cause is unclear.
---

# Mac network troubleshooting with Kilobat

Kilobat is a macOS network utility. Its CLI (`kilobat`) measures Wi-Fi signal, router, internet and DNS latency, per-app traffic and speed. This skill turns those measurements into a diagnosis: find the failing layer, show the numbers, suggest one fix at a time, then measure again.

## Setup

1. Prefer the Kilobat MCP tools when they are connected (`diagnose_network`, `get_status`, …). They return the same JSON as the CLI. See the table at the end.
2. Otherwise check for the CLI: `command -v kilobat`. Kilobat runs on macOS only. If it is missing, install it with `brew install seifip/kilobat/kilobat` (macOS 14+ and Xcode 16+ required); do not claim it is installed until the command succeeds.
3. Pass `--json` to commands that support it; some expert commands already return JSON and do not accept that flag. Errors go to stderr and return a non-zero exit code.
4. Use one-shot commands only. `status --watch` and `radar --watch` never exit; they are for the user's own Terminal.

## Safety: read freely, ask before changing

Read-only, run without asking: `diagnose`, `status`, `scan`, `radar --json`, `report`, `rules`, `profile` (no argument), `limit` (no argument), `dns` (no argument), `dns-benchmark`, `http-check`, `trace`, `interfaces`, `destinations`, `devices`, `outages`, `speed-history`, `doctor`.

Ask first, naming the exact change:

- `speedtest`: transfers up to 80 MB. On a metered connection, ask and pass `--budget-mb 10`.
- `profile <name>`, `allow`/`block <app>`, `limit <GB>`, `remember-network`: change the user's saved settings.
- `dns <provider>` and `dns flush`: change system settings and show a macOS administrator prompt. Record the current resolvers first and offer to restore `automatic` if it does not help.
- `capture`, `discover`, `wake`, `blocklist`, `connection-rule`, `scope`: expert tools. Use them only when the user asks.

Kilobat saves rules; blocking only happens while the Kilobat app runs with its network filter enabled. The CLI cannot verify that (`filteringActive` is `null`), so say "saved", not "blocked".

## Workflow

### 1. Measure the connection

Run `kilobat diagnose --json` (a few seconds). Read the top-level fields first:

- `health`: `good`, `degraded`, `poor` or `unknown`.
- `advice`: Kilobat's own plain-language findings. Quote them; they apply its thresholds.
- `signalScore` (0–100), `signalNoiseRatio` (dB), `routerLatencyMs`, `internetLatencyMs`, `internetJitterMs`.
- `measurements`: radio (`rssi`, `noise`, `channel`, `band`, `transmitMbps`), `router` and `internet` ping samples with `lossPercent`, `dnsMilliseconds`, `dnsServers`, `captivePortal`, `networkService`.

One sample is a snapshot. For "flaky" or "drops sometimes", run diagnose three times about 20 seconds apart and compare, and read `kilobat outages` and `kilobat speed-history` for earlier events.
Treat missing or null measurements as unavailable, not zero or a passing result. Report what was actually measured and avoid naming a cause without corroboration.

### 2. Find the failing layer

Work from the Mac outward, considering each measured layer; more than one issue may be present. [references/reading-results.md](references/reading-results.md) explains the fields and Kilobat's thresholds.

| Symptom in the JSON | Layer | What to tell the user |
|---|---|---|
| `captivePortal: true` | Network login | Open the network's login page in a browser, then retest. Nothing else is meaningful until then. |
| `rssi` below −65 dBm, `signalNoiseRatio` below 25 or `signalScore` below 50 | Wi-Fi signal | Move closer to the router or remove obstructions. Radar finds a better spot (step 5). |
| `routerLatencyMs` above 20 or router loss | Local link | Wi-Fi congestion or a busy router. Check `scan` for crowded channels and `status` for heavy apps; restart the router last. |
| Router fast, `internetLatencyMs` above 120 or internet loss of 5% or more | Internet path | The path to the test target may be slow or lossy; it does not identify the provider as the cause. Compare another target or network and retest. |
| `internetJitterMs` above 30 | Unstable latency | Calls and games may stutter. Consider a speed test to check latency under load. |
| `dnsMilliseconds` above 150 | DNS | Benchmark resolvers with `kilobat dns-benchmark`; offer to switch DNS (ask first). |
| `internet.samples` empty | Inconclusive | Some networks block ping. Confirm with `kilobat http-check https://www.apple.com`. |
| All good | Capacity or a specific app | Check `status`, then run a speed test. |

When a VPN is on, latency includes the tunnel: `kilobat interfaces --json` shows VPN interfaces separately.

### 3. Find bandwidth hogs

Run `kilobat status --json` (samples for 2–8 seconds). Sort `apps` by `downloadBytesPerSecond + uploadBytesPerSecond` and name the top few, such as cloud sync, backups or updates. `metered: true` means macOS reports an expensive connection such as a phone hotspot. For past usage, run `kilobat report --summary --json --days 7`; it returns daily totals and the top apps by bytes.

### 4. Measure capacity and bufferbloat

Run `kilobat speedtest --json` when the user needs throughput numbers or calls stutter while other traffic runs. It takes up to about 30 seconds. Read `measurements.downloadMbps`, `measurements.uploadMbps`, `bufferbloat` (`Low`, `Moderate`, `High` or `Inconclusive`) and `addedLatencyMs`. High bufferbloat with good Mbps means the router queues too much under load: pause large uploads or downloads, or enable SQM/smart queueing on the router. Video calls need about 3.8 Mbps up and 3 Mbps down for 1080p.

### 5. Hand off to a focused skill

When one layer stands out, the matching Kilobat skill goes deeper. Load it if installed; the `kilobat` skill lists them all.

- Weak signal, crowded channel, drops or roaming: `mac-wifi-troubleshooting` (spot comparison with Radar, channel analysis, outage history).
- Phone hotspot, metered or capped connection: `mac-hotspot-data-saver` (Travel profile, data budget, blocking background apps). Run speed tests only with `--budget-mb 10` or less there.
- Slow DNS or a captive portal that will not appear: `mac-dns-troubleshooting`.
- Calls stutter: `video-call-network-check`.
- One app is using the connection: `mac-bandwidth-monitor`.

If those skills are not installed, the commands in step 3 and the expert reference cover the basics.

### 6. Report and verify

Lead with the evidence and its limits, for example: "Your Wi-Fi signal is strong (−52 dBm, SNR 40 dB) and the router answers in 3 ms, but the tested internet destination averaged 180 ms with 8% loss. Compare another target or network before attributing the cause." Then give at most three next steps, most likely first. After any change, run the same measurement again and report before and after.

## MCP tools

| MCP tool | CLI equivalent | Changes settings |
|---|---|---|
| `diagnose_network` | `kilobat diagnose --json` | No |
| `get_status` | `kilobat status --json` | No |
| `scan_wifi` | `kilobat scan --json` | No |
| `run_speedtest` (`budget_mb`) | `kilobat speedtest --json --budget-mb N` | No, but uses data |
| `get_usage_report` (`days`) | `kilobat report --summary --json --days N` | No |
| `list_rules` | `kilobat rules --json` | No |
| `set_profile` | `kilobat profile <name>` | Yes, ask first |
| `set_app_rule` | `kilobat allow` or `block <app> [--minutes N]` | Yes, ask first |
| `set_dns` | `kilobat dns <provider>` | Yes, ask first |

To connect the server, add `kilobat mcp` as a stdio MCP server, for example `claude mcp add kilobat -- kilobat mcp`.

For deeper investigation (route traces, resolver benchmarks, HTTP timing, sockets per process), see [references/expert-commands.md](references/expert-commands.md).
