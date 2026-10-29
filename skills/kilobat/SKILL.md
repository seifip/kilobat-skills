---
name: kilobat
description: Entry point for Kilobat, the macOS network utility with a CLI and MCP server. Read first for any Mac network task, such as slow internet, Wi-Fi problems, DNS, video calls, hotspot data, per-app bandwidth, blocking apps or domains, approving new apps, a site or API that won't connect, or keeping the Mac awake for a long agent run. Holds setup, safety rules and which focused skill to load.
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
| The router, ISP or carrier shows more (or less) data than Kilobat | `mac-bandwidth-monitor`, "When a router or provider shows a different total" |
| Block or allow an app, a domain or an IP; approve apps that are asking to connect; what an app is supposed to connect to | `mac-app-firewall` |
| Keep the Mac awake for a long build, download or agent run; did the agent finish or stall; is now a good time for a big download | `mac-agent-runs` |
| One site, API or app can't connect while others work | `kilobat why <url\|host>` or `diagnose_connection` first, then `mac-network-troubleshooting` if more is wrong |

If the matching skill is not installed, install the set with `npx skills add seifip/kilobat-skills`, or follow the rules below and `kilobat <command> --help`.

## Setup

1. Kilobat runs on macOS 14 or later only.
2. Prefer the MCP tools when they are connected (`diagnose_network`, `get_status`, …). They return the same JSON as the CLI.
3. Otherwise check for the CLI: `command -v kilobat`. If it is missing, ask before installing it with `brew install seifip/kilobat/kilobat`. Do not claim it is installed until the command succeeds.
4. To connect the MCP server, add `kilobat mcp` as a stdio server, for example `claude mcp add kilobat -- kilobat mcp`.
5. `kilobat doctor` reports whether the Kilobat app is running, whether its network filter is on, and whether Kilobat Pro is active.
6. This set of skills targets CLI 0.5 (`kilobat --version`); `awake`, `sessions`, `why` and `conditions` need 0.4, and `bandwidth`, `access-policy`, `approvals`, `approve`, `deny`, `block-host` and `unblock-host` need 0.5. If a command is unknown, the CLI is older: suggest `brew upgrade kilobat`, and use `kilobat --help` to find the older equivalent.

## Conventions

- Output is JSON when piped; pass `--json` where a command supports it. Some expert commands always print JSON and reject the flag. Errors go to stderr with a non-zero exit code.
- Run one-shot commands only. `status --watch` and `radar --watch` never exit; suggest them for the user's own Terminal.
- `null` or missing fields mean "not measured", never zero or a pass.
- History (`report`, `health-history`, `outages`, `speed-history`, `spots`, `scan-history`) is recorded by the Kilobat app while it runs. A gap is not evidence that nothing happened.
- macOS shows Wi-Fi names only to apps with Location permission. The CLI gets them from the running Kilobat app; without it, `ssid` is null and scans list "Hidden network". Signal, channel and band still work.
- App IDs are bundle IDs (`com.example.App`) or `process:<name>` for processes without one, exactly as `status` prints them.

## Safety: read freely, ask before changing

Run without asking: `diagnose`, `status`, `conditions`, `why`, `sessions`, `scan`, `channels`, `radar --json`, `wifi status`, `wifi known`, `report`, `health-history`, `networks`, `rules`, `settings` with no value, `profile`, `limit` and `dns` with no argument, `dns-benchmark`, `http-check`, `trace`, `interfaces`, `destinations`, `devices`, `outages`, `speed-history`, `spots`, `scan-history`, `doctor`, `bandwidth`, `access-policy`, `approvals`, `blocked-history`, `stutter` (sends pings only), `airdrop` with no argument, `bluetooth`.

Name the exact change and wait for a yes before:

- `speedtest`: uses up to 80 MB. On a metered network pass `--budget-mb 10` or less, or skip it.
- `profile <name>`, `allow`/`block`/`reset <app>`, `limit <GB>`, `remember-network`, `networks forget`, `settings <name> <value>`, `block-host`/`unblock-host`: change saved settings.
- `approve`/`deny <app>`: answer an app waiting to connect, which also saves a rule. Show the user its signer and destinations first.
- `wifi on|off|restart|disconnect|join`: change the connection. Turning Wi-Fi off or joining another network can cut off a remote session.
- `wifi password` and `wifi share`: reveal a saved password after a macOS prompt. Only when the user asks for it.
- `quit <PID>`: quits a process. `low-latency on [--for 15m|1h|2h] [--until-quit <app>]`: pauses AirDrop and AWDL while it runs (4 hours at most), with an administrator prompt; the user runs it in their own Terminal. `airdrop everyone|contacts|off`: opens System Settings for the user to change AirDrop visibility.
- `dns <provider>`, `dns flush`: change system settings behind a macOS administrator prompt. Record the current resolvers first.
- `capture`, `discover`, `wake`, `lan-test`, `blocklist`, `connection-rule`, `domain-exception`, `scope`: expert tools; only when the user asks for them.

Keeping the Mac awake (`keep_awake`, `kilobat awake`) is low risk: it changes no settings and only stops idle sleep; closing the lid still sleeps the Mac. Tell the user for how long, ask first on battery, and release the hold when the work is done. Running with the lid closed (`lid_closed`, `--lid-closed`) works only after the user allowed it in Kilobat Settings, and a closed Mac in a bag can overheat or run flat: get the user's explicit confirmation every time (see `mac-agent-runs`). Never change sleep settings with `pmset`.

Before a large download or upload (more than a few hundred MB) or a long unattended run, check `get_conditions` (`kilobat conditions --json`). On a metered connection or in Low Data Mode, ask first.

Rules and budgets are saved by the CLI and enforced only by the Kilobat app with its network filter enabled (Kilobat Pro). `status --json` and `rules --json` report `filteringActive` when the app is running; when it is `false` or `null`, say "saved", not "blocked".

Traffic history, network names and rules stay on the Mac. Do not send them anywhere without the user asking.

## Commands at a glance

| Need | Command | MCP tool |
|---|---|---|
| Health of each layer, with advice | `kilobat diagnose --json` | `diagnose_network` |
| Live per-app traffic, metered flag, profile | `kilobat status --json` | `get_status` |
| Current Wi-Fi link | `kilobat wifi status --json` | `get_wifi` |
| Nearby networks and channels | `kilobat scan --json` | `scan_wifi` |
| Least crowded channel | `kilobat channels --json` | `suggest_wifi_channel` |
| One signal sample (score 0–100) | `kilobat radar --json` | — |
| Speed and bufferbloat | `kilobat speedtest --json [--budget-mb N]` | `run_speedtest` |
| Latency spikes and whether they repeat | `kilobat stutter --json [--target router\|internet]` | `run_stutter_test` |
| AirDrop link and its users, AirDrop visibility, Bluetooth on 2.4 GHz | `kilobat airdrop --json`, `bluetooth --json` | `get_nearby_radios`, `set_airdrop_visibility` |
| Usage history | `kilobat report --summary --json --days N` | `get_usage_report` |
| A router's or provider's figure beside Kilobat's for the same period | `kilobat report compare --total 40GB --period last-month --json` | `compare_usage` |
| Drops, past speed tests, health over time | `kilobat outages`, `speed-history`, `health-history --json` | `get_network_history` |
| Resolver comparison, one site's timing | `kilobat dns-benchmark`, `http-check <URL> --json` | `benchmark_dns`, `check_url` |
| Sockets per process, local devices, route | `kilobat destinations --json`, `devices --json`, `trace <host>` | `list_connections`, `list_devices`, `trace_route` |
| Networks and their remembered settings | `kilobat networks --json` | `list_networks` |
| A profile's rules | `kilobat rules --json [--profile name]` | `list_rules` |
| Select a profile | `kilobat profile travel\|work\|unrestricted` | `set_profile` |
| Allow, block or reset an app | `kilobat allow\|block\|reset <app> [--minutes N] [--profile name]` | `set_app_rule` |
| Data budget | `kilobat limit <GB>` | `set_budget` |
| Read, set or flush DNS | `kilobat dns [automatic\|cloudflare\|google\|quad9\|flush]` | `set_dns` |
| Settings such as new-app mode (allow, ask, block) and auto Travel | `kilobat settings [name value]` | `set_new_app_mode` for new apps |
| One app's traffic over an hour, day, week or month | `kilobat bandwidth <app> day --json` | `get_app_bandwidth` |
| What an app's developer says it connects to | `kilobat access-policy <bundle-id> --json` | `get_access_policy` |
| Apps waiting to connect, and answering them | `kilobat approvals --json`, `approve\|deny <app>` | `list_approvals`, `decide_approval` |
| Block a domain, IP or range | `kilobat block-host <target> [--app <id>]` | `block_destination` |
| What the filter blocked recently | `kilobat blocked-history` | `list_blocked_connections` |
| Metered, Low Data Mode, VPN, battery and heat, with advice | `kilobat conditions --json` | `get_conditions` |
| Updaters paused on metered networks, update bytes that got through, apps that update inside themselves | `kilobat updaters --json`, `updaters on\|off` | `get_updaters`, `set_updater_pause` |
| Why one site, API or app can't connect | `kilobat why <url\|host> [--app <id>] --json` | `diagnose_connection` |
| Keep the Mac awake for a run or a command | `kilobat awake --for 90m`, `awake --until-quiet`, `awake -- <command>` | `keep_awake`, `release_awake` |
| Keep it awake only while the user reads or presents in one app | `kilobat awake --while-frontmost <app> [--display]`, in the user's Terminal | None; it is the user's, not an agent's |
| Earlier awake sessions and AI agent runs | `kilobat sessions --json [--days N]` | `list_sessions` |

Run `kilobat --help` for the full list, including expert tools.
