---
name: mac-network-troubleshooting
description: Diagnose slow or unreliable internet on a Mac with Kilobat's CLI or MCP server. Measures Wi-Fi signal, router and internet latency, packet loss, jitter, DNS, captive portals and bufferbloat, finds which layer is failing and suggests one fix at a time. Use when the internet is slow, drops or lags and the cause is unclear, or when one site, API or app can't connect.
---

# Mac network troubleshooting with Kilobat

Kilobat is a macOS network utility. Its CLI (`kilobat`) measures Wi-Fi signal, router, internet and DNS latency, per-app traffic and speed. This skill turns those measurements into a diagnosis: find the failing layer, show the numbers, suggest one fix at a time, then measure again.

## Setup

1. Prefer the Kilobat MCP tools when they are connected (`diagnose_network`, `get_status`, …). They return the same JSON as the CLI. See the table at the end.
2. Otherwise check for the CLI: `command -v kilobat`. Kilobat runs on macOS only. If it is missing, install it with `brew install seifip/kilobat/kilobat` (macOS 14+ and Xcode 16+ required); do not claim it is installed until the command succeeds.
3. Pass `--json` to commands that support it; some expert commands already return JSON and do not accept that flag. Errors go to stderr and return a non-zero exit code.
4. Use one-shot commands only. `status --watch` and `radar --watch` never exit; they are for the user's own Terminal.

## Safety: read freely, ask before changing

Read-only, run without asking: `diagnose`, `why`, `status`, `scan`, `radar --json`, `report`, `rules`, `profile` (no argument), `limit` (no argument), `dns` (no argument), `dns-benchmark`, `http-check`, `trace`, `route-history`, `interfaces`, `ip` (without `--public`, which contacts Cloudflare), `speed-compare`, `destinations`, `devices`, `outages`, `speed-history`, `doctor`.

Ask first, naming the exact change:

- `speedtest`: transfers up to 80 MB. On a metered connection, ask and pass `--budget-mb 10`.
- `profile <name>`, `allow`/`block <app>`, `limit <GB>`, `remember-network`: change the user's saved settings.
- `dns <provider>` and `dns flush`: change system settings and show a macOS administrator prompt. Record the current resolvers first and offer to restore `automatic` if it does not help.
- `capture`, `discover`, `wake`, `blocklist`, `connection-rule`, `scope`: expert tools. Use them only when the user asks.

Kilobat saves rules; blocking only happens while the Kilobat app runs with its network filter enabled. `status --json` reports `filteringActive` when the app is running; unless it is `true`, say "saved", not "blocked".

## Workflow

### One site, API or app can't connect

When the internet works but one destination fails ("this app can't connect", "the agent can't reach the API", a timeout, a reconnect loop or a certificate error), start with the doctor: `kilobat why <url|host> --json` (MCP: `diagnose_connection` with `target`). It needs CLI 0.4 or later; CLIs up to 0.5.1 report `VPN & proxy` and `Connection` instead of the interface, proxy, IPv4 and IPv6 steps below, and lack `--proxy`, redirects, the HEAD-to-GET retry and `Open connections`; CLIs up to 0.5.1 also lack `--failed-at`, `client`, `method`, `path`, `App setting` and `Transfer`. Pass the exact URL that fails, path and query included: the doctor requests it as given and shows query values only as `[redacted]`. Don't repeat a signed URL's values yourself. Pass `--app <id>` (MCP: `app`) with the `id` from `status` when one app is affected, so its Kilobat rules, the proxy variables in its environment and its open connections to the host are checked too. When the app uses a proxy address from its own settings, which Kilobat can't read, ask for it and pass `--proxy <url>` (MCP: `proxy`). A setting with no address, such as LM Studio's Hugging Face proxy switch, can't be passed: for LM Studio and huggingface.co the doctor adds an `App setting` step on its own. When the user says when the app failed, pass `--failed-at <time>` (MCP: `failed_at`, ISO 8601 or 14:05). Its probes run inside Kilobat, not through `curl` or `nc`, so a strict profile can't block the check.

The steps come in order: `Network`, `Interface` (Wi-Fi, Ethernet, a metered phone hotspot or a VPN, and the source address), `Proxy` (the system proxy or what its PAC file picks), `App proxy` (`HTTPS_PROXY`, `ALL_PROXY`, `NO_PROXY` in the app's environment, or the proxy passed with `--proxy`, credentials removed), `DNS` (addresses, the resolver, a VPN's private resolver, encrypted DNS), `Kilobat rules`, `IPv4` and `IPv6` (TCP and TLS each, direct), `Other networks` (the same connection over each interface when there are several), `Secure connection` or `Request`, `Redirects` (each hop's host and answer), `Server` (the final HTTP answer), `Redirect IPv4` and `Redirect IPv6` (the host the redirects end at), `Direct path` (with a VPN or proxy on: direct against the tunnel or proxy), `App setting` (an app setting Kilobat can't read or go through, with dated user reports), `Open connections` (with `--app`: to the host, or to the proxy it goes through), `Transfer` (whether bytes still arrive on those connections a few seconds apart), and `Context` or `Provider`. Each step has `status` (`ok`, `warning`, `failed`, `skipped`) and `detail`; `hops` lists each request (`url` redacted, `host`, `method`, `status` or `error`); `verdict` and `next` say what it means and what to do, and `ok` is false (exit 1) when a layer failed. Quote them. The report also records `client` (`id`, `name` and `version` as installed), `method` (what the check sent: HEAD, or a one-byte GET), `path` (`kind`: `appProxy`, `environmentProxy`, `systemProxy` or `direct`, the `proxy` as host:port, and `untested`, an app setting the check couldn't go through), `date` (when the check ran) and `failedAt`.

The request goes the way the app's would: through the proxy passed with `--proxy`, else the app's environment proxy, else the system proxy. A passing check means this request from this Mac over that path works now. It doesn't prove the app is at fault, and it can't see a connection the app already has open. When the user describes a slow or failing transfer already under way, say that, and watch it with `kilobat bandwidth <app>` (MCP: `get_app_bandwidth`) or the app's bandwidth panel in Kilobat.

When the request through the app's own proxy fails while direct connections work, the verdict says the network is fine and the app's path isn't; when `path.untested` is set, it says the network reaches the host, not that the app's request works. Report it that way. A `Transfer` warning means the app's connection went quiet: that can be a stalled download or a finished one whose connection is still open, so never say the download finished. Suggest checking the app's own setting as `next` words it, citing it as user reports with their date; don't change another app's settings, and don't say the app will resume a download on its own.

- `IPv4` ok and `IPv6` failed (or the reverse): that IP version is broken for this host on this network. Apps that prefer it hang or keep reconnecting while browsers seem fine. Relay `next` (Link-local only for IPv6 on this network, or ask the network owner); don't blame the server.
- `App proxy` failed: the app points at a proxy that doesn't answer, often a stopped Clash or V2Ray port in a shell profile. A warning means the app's proxy differs from the system's; most command-line tools ignore the system proxy.
- `Direct path` ok with "Through … it works; direct connections … don't": the app's proxy reaches the host and direct connections are blocked, common on company networks. Direct `IPv4`/`IPv6` failures show as warnings then. Tools that ignore that proxy need `HTTPS_PROXY`.
- `Server` "refuses HEAD requests but answers GET": the server is fine; many download links and APIs reject HEAD. Nothing beyond one byte was downloaded.
- `Redirects` failed or `Redirect IPv6` failed: the first server answers but the host it redirects to (often a CDN) fails, or fails over one IP version. Name that host; don't blame the first one.
- `Open connections` warning: the app's open connection uses another IP version or server than the fresh check, so the check may not reflect what the app sees. A new connection (pausing and resuming the transfer) may take the fresh path; don't claim it will fix it.
- `Direct path` failed: direct connections work but the VPN or proxy path fails. After switching networks (Wi-Fi to a tethered phone and back) a tunnel can keep using the old network. Suggest restarting the tunnel; don't name a cause the report doesn't show.
- `DNS` naming a private resolver that doesn't answer: the VPN, Tailscale or NetBird resolver for that domain is down while public DNS works.
- `Other networks` warning: it connects over another interface, such as a tethered iPhone, but not this one.
- A `Kilobat rules` failure means Kilobat itself blocks it. Offer the specific allow rule or blocklist change, with approval.
- A `Server` warning such as 401, 403, 404 or 429 means the network is fine; the request needs a login, a key, another path or a slower pace.
- Code that runs in a container, VM or remote machine has its own network, DNS and proxy. Pass `--context container|vm|remote` (MCP: `context`); the `Context` step ends with a command to run there, such as `docker run --rm curlimages/curl -sv https://host/`. Names like `db` or `*.svc.cluster.local` get that note on their own. Never present the Mac's result as the container's.
- A hosted service whose own servers make the request, such as Warp's custom inference endpoint (Warp's docs, Oct 2026: requests go through Warp's servers, so the endpoint needs a public URL), can't reach `localhost`, a LAN or Tailscale address, or a name only private DNS knows, however well it works from the Mac. Pass `--context provider` (MCP: `context`) with `--app dev.warp.Warp-Stable` to name Warp; picking Warp as the app adds the note on its own, worded as an if. The `Provider` step says whether the target is private, and `next` gives the options: an endpoint the user controls with a public address, or a route the provider supports. Don't suggest SSH into the provider, a tunnel or exposing the Mac's service, or changing the Mac's DNS, firewall or VPN for it. When the local check fails too, fix that first; what the provider's servers reach stays unmeasured.
- All layers pass but the app still fails: this request works on the path tested, so the app may take another one, such as its own proxy setting (ask for it and rerun with `--proxy`), a sandbox, a container or a remote machine, or an older connection still open. Say which you can't rule out; don't conclude the app is at fault or that another network can't help.
- A failure at `Network`, `DNS` or both `IPv4` and `IPv6` for several sites: continue with step 1.

### 1. Measure the connection

Run `kilobat diagnose --json` (a few seconds). Read the top-level fields first:

- `health`: `good`, `degraded`, `poor` or `unknown`.
- `advice`: Kilobat's own plain-language findings. Quote them; they apply its thresholds.
- `signalScore` (0–100), `signalNoiseRatio` (dB), `routerLatencyMs`, `internetLatencyMs`, `internetJitterMs`.
- `measurements`: radio (`rssi`, `noise`, `channel`, `band`, `transmitMbps`), `router` and `internet` ping samples with `lossPercent`, `dnsMilliseconds`, `dnsServers`, `captivePortal`, `networkService`.

One sample is a snapshot. For "flaky" or "drops sometimes", run diagnose three times about 20 seconds apart and compare, and read `kilobat health-history --hours 24 --json`, `kilobat outages --days 7` and `kilobat speed-history` for earlier events. These need CLI 0.2 or later; if a command is unknown, suggest `brew upgrade kilobat`.
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
| Wi-Fi fine but one channel is crowded | Channel | `kilobat channels --json` names the least crowded channel to set on the router. |
| `internet.samples` empty | Inconclusive | Some networks block ping. Confirm with `kilobat http-check https://www.apple.com`. |
| All good | Capacity or a specific app | Check `status`, then run a speed test. |

When a VPN is on, latency includes the tunnel. `kilobat ip --json` (MCP: `get_exit_path`) says where traffic leaves, read on the Mac: `vpn` is `full` (the default route goes through a tunnel), `partial` (a VPN such as Tailscale is connected but carries only some destinations) or `off`, with `vpnName`. Add `--public` (`public_ip`) only when the user asks for their public IP or VPN exit: it asks Cloudflare, and the country is where the address is registered. Report what was measured ("traffic leaves through utun3 (WireGuard), exit IP registered in Germany"); never say a VPN or an exit country makes the user private, anonymous or safe.

### 3. Find bandwidth hogs

Run `kilobat status --json` (samples for 2–8 seconds). Sort `apps` by `downloadBytesPerSecond + uploadBytesPerSecond` and name the top few, such as cloud sync, backups or updates. `metered: true` means macOS reports an expensive connection such as a phone hotspot. For past usage, run `kilobat report --summary --json --days 7`; it returns daily totals and the top apps by bytes.

### 4. Measure capacity and bufferbloat

Run `kilobat speedtest --json` when the user needs throughput numbers or calls stutter while other traffic runs. It takes up to about 30 seconds. Read `measurements.downloadMbps`, `measurements.uploadMbps`, `bufferbloat` (`Low`, `Moderate`, `High` or `Inconclusive`) and `addedLatencyMs`. High bufferbloat with good Mbps means the router queues too much under load: pause large uploads or downloads, or enable SQM/smart queueing on the router. Video calls need about 3.8 Mbps up and 3 Mbps down for 1080p.

When the user says another speed test (Ookla, fast.com, their provider's) shows more: Kilobat uses one connection by default, `method` in the result says how, and those sites open several. Run `kilobat speedtest --parallel --json` (MCP `run_speedtest` with `parallel`) to see the gap on this connection. When two Kilobat tests differ, `kilobat speed-compare --json` (MCP `compare_speed_tests`) lists what differed: network, time, server, connections, Wi-Fi band and signal, VPN, other traffic on the Mac, idle latency. Relay those as differences, not proven causes; other devices on the network are invisible to Kilobat.

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

## When the cause isn't clear: look up a guide

Kilobat's site has guides for common macOS networking problems and step-by-step recipes, with sources and the macOS versions they apply to. When the measurements don't point to a cause, or the symptom is a known macOS pattern (Wi-Fi drops after sleep, Private Relay breaking a site, a container that can't reach localhost), fetch `https://kilobat.app/guides/index.json` once. Match the user's words and what you measured against each entry's `title`, `searches` and `symptoms`, and check `appliesTo` against the Mac's macOS version. Open the best match's `url`, follow its checks before its fixes, and give the user the link. A guide is general advice: prefer what you measured on this Mac, and still ask before any change.

## MCP tools

| MCP tool | CLI equivalent | Changes settings |
|---|---|---|
| `diagnose_network` | `kilobat diagnose --json` | No |
| `get_status` | `kilobat status --json` | No |
| `scan_wifi` | `kilobat scan --json` | No |
| `run_speedtest` (`budget_mb`, `parallel`) | `kilobat speedtest --json --budget-mb N [--parallel]` | No, but uses data |
| `compare_speed_tests` (`first`, `second`) | `kilobat speed-compare [<test> <test>] --json` | No |
| `get_exit_path` (`public_ip`) | `kilobat ip --json [--public]` | No; `public_ip` contacts Cloudflare |
| `get_usage_report` (`days`) | `kilobat report --summary --json --days N` | No |
| `get_wifi`, `suggest_wifi_channel` | `kilobat wifi status --json`, `channels --json` | No |
| `diagnose_connection` (`target`, `app`, `proxy`, `context`, `failed_at`) | `kilobat why <target> --json [--app <id>] [--proxy <url>] [--context container\|vm\|remote\|provider] [--failed-at <time>]` | No |
| `get_network_history` (`kind`, `days`) | `kilobat outages`, `speed-history`, `health-history --json` | No |
| `benchmark_dns`, `check_url` | `kilobat dns-benchmark`, `http-check <URL> --json` | No |
| `list_rules` | `kilobat rules --json` | No |
| `set_profile` | `kilobat profile <name>` | Yes, ask first |
| `set_app_rule` | `kilobat allow` or `block <app> [--minutes N]` | Yes, ask first |
| `set_budget` | `kilobat limit <GB>` | Yes, ask first |
| `set_dns` (incl. `flush`) | `kilobat dns <provider>` | Yes, ask first |

To connect the server, add `kilobat mcp` as a stdio MCP server, for example `claude mcp add kilobat -- kilobat mcp`.

For deeper investigation (route traces, resolver benchmarks, HTTP timing, sockets per process), see [references/expert-commands.md](references/expert-commands.md).
