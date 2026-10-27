---
name: kilobat
description: Entry point for Kilobat, the macOS network utility with a CLI and MCP server. Read first for any Mac network task, such as slow internet, Wi-Fi problems, DNS, video calls, hotspot data or per-app bandwidth. Holds setup, safety rules and which focused skill to load.
---

# Kilobat

Kilobat is a macOS menu-bar app and a free, open-source CLI (`kilobat`, alias `kb`) for the network. It measures Wi-Fi signal, router, internet and DNS latency, per-app traffic and speed, and saves blocking rules that the app enforces. This skill sets up the tools and routes to the skill that does the work. It does not hold the workflows itself: load the matching skill before running commands for a task.

## Route the request

Pick by what the user describes. A task may need more than one skill; load them in the order you act.

| The user says | Load |
|---|---|
| Internet is slow, drops, lags or "something is wrong"; unclear where | `mac-network-troubleshooting` |
| Weak or dropping Wi-Fi, dead zones, where to put the router, crowded channels, mesh roaming | `mac-wifi-troubleshooting` |
| Phone hotspot, tethering, plane or hotel Wi-Fi, roaming, data cap, "what ate my data" | `mac-hotspot-data-saver` |
| Pages slow to start, "server not found", which DNS is fastest, changing DNS, encrypted DNS, captive portal not appearing | `mac-dns-troubleshooting` |
| Zoom, Meet, Teams or FaceTime stutters, robotic audio, frozen video, check before a call | `video-call-network-check` |
| Which app uses the internet, background uploads, usage per app or per day, unknown connections | `mac-bandwidth-monitor` |

If the matching skill is not installed, install the set with `npx skills add seifip/kilobat-skills`, or follow the rules below and `kilobat <command> --help`.

## Setup

1. Kilobat runs on macOS 14 or later only.
2. Prefer the MCP tools when they are connected (`diagnose_network`, `get_status`, …). They return the same JSON as the CLI.
3. Otherwise check for the CLI: `command -v kilobat`. If it is missing, ask before installing it with `brew install seifip/kilobat/kilobat`. Do not claim it is installed until the command succeeds.
4. To connect the MCP server, add `kilobat mcp` as a stdio server, for example `claude mcp add kilobat -- kilobat mcp`.
5. `kilobat doctor` reports whether monitoring and the network filter are ready.

## Conventions

- Output is JSON when piped; pass `--json` where a command supports it. Some expert commands always print JSON and reject the flag. Errors go to stderr with a non-zero exit code.
- Run one-shot commands only. `status --watch` and `radar --watch` never exit; suggest them for the user's own Terminal.
- `null` or missing fields mean "not measured", never zero or a pass.
- History (`report`, `outages`, `speed-history`, `spots`, `scan-history`) is recorded by the Kilobat app while it runs. A gap is not evidence that nothing happened.
- App IDs are bundle IDs (`com.example.App`) or `process:<name>` for processes without one, exactly as `status` prints them.

## Safety: read freely, ask before changing

Run without asking: `diagnose`, `status`, `scan`, `radar --json`, `report`, `rules`, `profile`, `limit` and `dns` with no argument, `dns-benchmark`, `http-check`, `trace`, `interfaces`, `destinations`, `devices`, `outages`, `speed-history`, `spots`, `scan-history`, `doctor`.

Name the exact change and wait for a yes before:

- `speedtest`: uses up to 80 MB. On a metered network pass `--budget-mb 10` or less, or skip it.
- `profile <name>`, `allow`/`block <app>`, `limit <GB>`, `remember-network`: change saved settings.
- `dns <provider>`, `dns flush`: change system settings behind a macOS administrator prompt. Record the current resolvers first.
- `capture`, `discover`, `wake`, `lan-test`, `blocklist`, `connection-rule`, `domain-exception`, `scope`: expert tools; only when the user asks for them.

Rules and budgets are saved by the CLI and enforced only by the Kilobat app with its network filter enabled (Kilobat Pro). The CLI cannot see the filter (`filteringActive` is always `null`), so say "saved", not "blocked", unless the user confirms the filter is on.

Traffic history, network names and rules stay on the Mac. Do not send them anywhere without the user asking.

## Commands at a glance

| Need | Command | MCP tool |
|---|---|---|
| Health of each layer, with advice | `kilobat diagnose --json` | `diagnose_network` |
| Live per-app traffic, metered flag, profile | `kilobat status --json` | `get_status` |
| Nearby networks and channels | `kilobat scan --json` | `scan_wifi` |
| One signal sample (score 0–100) | `kilobat radar --json` | — |
| Speed and bufferbloat | `kilobat speedtest --json [--budget-mb N]` | `run_speedtest` |
| Usage history | `kilobat report --summary --json --days N` | `get_usage_report` |
| Profile and rules | `kilobat rules --json` | `list_rules` |
| Select a profile | `kilobat profile travel\|work\|unrestricted` | `set_profile` |
| Allow or block an app | `kilobat allow\|block <app> [--minutes N]` | `set_app_rule` |
| Read, set or flush DNS | `kilobat dns [automatic\|cloudflare\|google\|flush]` | `set_dns` (set only) |

Run `kilobat --help` for the full list, including expert tools.
