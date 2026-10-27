---
name: mac-network-troubleshooting
description: Diagnose slow internet, Wi-Fi drops, high latency, DNS and per-app bandwidth on macOS using Kilobat. Use for troubleshooting unreliable connections, video calls, hotspots, data use or bufferbloat when the Kilobat CLI or MCP server is available.
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

### 2. Find the failing layer

Work from the Mac outward and stop at the first layer that fails. [references/reading-results.md](references/reading-results.md) has every threshold and field.

| Symptom in the JSON | Layer | What to tell the user |
|---|---|---|
| `captivePortal: true` | Network login | Open the network's login page in a browser, then retest. Nothing else is meaningful until then. |
| `rssi` below −67 dBm, `signalNoiseRatio` below 25 or `signalScore` below 50 | Wi-Fi signal | Move closer to the router or remove obstructions. Suggest Radar (below). |
| `routerLatencyMs` above 20 or router loss | Local link | Wi-Fi congestion or a busy router. Check `scan` for crowded channels and `status` for heavy apps; restart the router last. |
| Router fast, `internetLatencyMs` above 120 or internet loss of 5% or more | Internet path | The path to the test target may be slow or lossy; it does not identify the provider as the cause. Compare another target or network and retest. |
| `internetJitterMs` above 30 | Unstable latency | Calls and games will stutter. Run a speed test to check bufferbloat. |
| `dnsMilliseconds` above 150 | DNS | Benchmark resolvers with `kilobat dns-benchmark`; offer to switch DNS (ask first). |
| `internet.samples` empty | Inconclusive | Some networks block ping. Confirm with `kilobat http-check https://www.apple.com`. |
| All good | Capacity or a specific app | Check `status`, then run a speed test. |

When a VPN is on, latency includes the tunnel: `kilobat interfaces --json` shows VPN interfaces separately.

### 3. Find bandwidth hogs

Run `kilobat status --json` (samples for 2–8 seconds). Sort `apps` by `downloadBytesPerSecond + uploadBytesPerSecond` and name the top few, such as cloud sync, backups or updates. `metered: true` means macOS reports an expensive connection such as a phone hotspot. For past usage, run `kilobat report --summary --json --days 7`; it returns daily totals and the top apps by bytes.

### 4. Measure capacity and bufferbloat

Run `kilobat speedtest --json` when the user needs throughput numbers or calls stutter while other traffic runs. It takes up to about 30 seconds. Read `measurements.downloadMbps`, `measurements.uploadMbps`, `bufferbloat` (`Low`, `Moderate`, `High` or `Inconclusive`) and `addedLatencyMs`. High bufferbloat with good Mbps means the router queues too much under load: pause large uploads or downloads, or enable SQM/smart queueing on the router. Video calls need about 3.8 Mbps up and 3 Mbps down for 1080p.

### 5. Wi-Fi signal: suggest Radar

When the signal is the problem and the Mac is a laptop, Radar finds a better spot. It is interactive, so the user runs it: `kilobat radar --watch` in Terminal, or the Signal guide in the Kilobat app. The user walks around while the score (0–100) updates, and Radar says when an earlier spot was stronger. To compare rooms yourself, ask the user to move, then sample `kilobat radar --json` in each spot. Use `kilobat scan --json` to look for a crowded channel (many networks on the same `channel` with a strong `rssi`) or a stronger access point for the same network. Scanning may need Location permission for the Kilobat app; an empty list means it is unavailable, not that no networks exist.

### 6. Hotspot and travel: save data

When `metered` is true, `network` is `Cellular`, or the user mentions a phone hotspot, plane or hotel Wi-Fi, or a data cap:

1. Show what is using data: `status` for right now, `report --summary --days 1` for today.
2. Offer the Travel profile: `kilobat profile travel`. Review its saved rules and filter status before describing their effect; automatic switching depends on the user's settings and the app running.
3. Offer a session budget: `kilobat limit 2` sets 2 GB; `kilobat remember-network` keeps the profile and budget for this network.
4. Offer to block specific heavy apps by the bundle ID from `status`: `kilobat block com.example.App`, or `--minutes 60` for a temporary rule.
5. Run speed tests only with `--budget-mb 10` or less.

Each of these is a change: describe it and wait for a yes.

### 7. Report and verify

Lead with the diagnosis and the evidence, for example: "Your Wi-Fi signal is fine (−52 dBm, SNR 40 dB) and the router answers in 3 ms, but internet latency is 180 ms with 8% loss, so the slowdown is upstream of your router." Then give at most three next steps, most likely first. After any change, run the same measurement again and report before and after.

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
