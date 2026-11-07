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
| OBS drops frames, the stream buffers or lags, "how much upload do I need to stream", check before going live | `kilobat stream reports --json` or `get_stream_reports`, then `kilobat stream check --json` or `check_stream_readiness` |
| GeForce NOW, Xbox Cloud Gaming, Boosteroid, Amazon Luna or Shadow stutters, lags or goes blurry; data a cloud game uses on a hotspot | `kilobat cloud-gaming --json` (the session now) and `kilobat cloud-gaming reports --json`, or `get_cloud_gaming`, then `video-call-network-check` for live spikes |
| AirPods or Bluetooth headphones crackle, skip or cut out on the Mac | `kilobat bluetooth audio --json` (CLI 0.28) or `get_bluetooth_audio` while they play, then `video-call-network-check` if calls also freeze |
| Chromecast, AirPlay TV or printer doesn't show up, "can I cast or print here", a cast or AirPlay mirror stutters, the Mac joined a printer's or camera's own Wi-Fi | `kilobat local check --json` or `check_local_network` |
| Typing lags over ssh, a remote desktop (Screen Sharing, Parsec, Jump, Windows App) freezes, "should I use Mosh" | `kilobat remote --json` or `get_remote_session_reports`, then `video-call-network-check` for live spikes |
| Which app uses the internet, background uploads, usage per app or per day, unknown connections | `mac-bandwidth-monitor` |
| The router, ISP or carrier shows more (or less) data than Kilobat | `mac-bandwidth-monitor`, "When a router or provider shows a different total" |
| Block or allow an app, a domain or an IP; approve apps that are asking to connect; what an app is supposed to connect to | `mac-app-firewall` |
| Is a host known to be malicious; block malware, phishing and trackers; why Kilobat blocked a site or an agent's request | `mac-app-firewall`, "Protection" |
| Why Kilobat allowed or blocked one connection or app, and which rule to change | `kilobat verdict <host> [--app <id>] --json` or `explain_verdict`, then `mac-app-firewall` |
| Keep the Mac awake for a long build, download or agent run; did the agent finish or stall; is now a good time for a big download; what did the agent or its install connect to | `mac-agent-runs` |
| Hotel, café, airport or coworking Wi-Fi: connected but nothing loads, "keeps logging me out", login page won't appear | `kilobat portal --json` or `get_captive_portal` first, then `mac-dns-troubleshooting` if the page won't appear |
| One site, API or app can't connect while others work | `kilobat why <url\|host>` or `diagnose_connection` first, then `mac-network-troubleshooting` if more is wrong |
| What is this process or host (nsurlsessiond, cloudd, gateway.icloud.com)? Is it safe to block, and what breaks? | `kilobat explain <process\|host> --json` or `explain_connection` |
| Is Slack, Zoom, GitHub or another service down for everyone? | `kilobat down <service\|host> --json` or `service_status` |
| Is my own model provider down (Claude Code, Codex, Copilot, Gemini CLI, an API)? | `kilobat down claude-code\|codex\|api.openai.com --json` or `service_status`, then `mac-agent-runs` |
| The status page is green but the site still fails: blocked on this network, or not offered in this region? | `kilobat why <host> --json` or `diagnose_connection` (give the app's error with `--error` / `error`); read `block` |
| What public reports expect in a country | `kilobat restrictions --country XX --json` | `get_country_restrictions` |

Blocked here, not down: when `block.state` is `blocked`, say what Kilobat measured, as its `summary` and `evidence` say it, and suggest only what `next` offers (another network, mobile data or roaming, the network's operator). When it is `regionUnavailable`, the service itself refuses where the connection appears to be (`exitCountry`); say so plainly. Never suggest a VPN, proxy, Tor, another DNS resolver or any other way around a block or a regional restriction, and never call either an outage.

Internet down on hotel, café or airport Wi-Fi? Check `kilobat portal --json` (MCP `get_captive_portal`) before anything else. When `state` is `portal`, a sign-in page is holding every connection: DNS, ping and speed results mean nothing until the user signs in, and a network that logged them out is not an outage or the provider's fault. Tell them to sign in, or run `kilobat portal open` when they ask you to open the page. Never fill in or accept anything on a sign-in page.

If the matching skill is not installed, install the set with `npx skills add seifip/kilobat-skills`, or follow the rules below and `kilobat <command> --help`.

For a problem none of these skills covers, or a macOS quirk, `https://kilobat.app/guides/index.json` lists Kilobat's guides with their symptoms, the macOS versions they apply to and the commands they use; open the matching guide's `url` and follow it.

## Setup

1. Kilobat runs on macOS 14 or later only.
2. Prefer the MCP tools when they are connected (`diagnose_network`, `get_status`, …). They return the same JSON as the CLI.
3. Otherwise check for the CLI: `command -v kilobat`. If it is missing, ask before installing it with `brew install seifip/kilobat/kilobat`. Do not claim it is installed until the command succeeds.
4. To connect the MCP server, add `kilobat mcp` as a stdio server, for example `claude mcp add kilobat -- kilobat mcp`.
5. `kilobat doctor` reports whether the Kilobat app is running, whether its network filter is on, and whether Kilobat Pro is active.
6. This set of skills targets CLI 0.5 (`kilobat --version`); `awake`, `sessions`, `why` and `conditions` need 0.4, and `bandwidth`, `access-policy`, `approvals`, `approve`, `deny`, `block-host` and `unblock-host` need 0.5, and `agent-log` needs 0.21, and `household` needs 0.22, and `credit` needs 0.23, and `router` needs 0.24, and `remote` needs 0.25, and `protect` needs 0.26, and `week` needs 0.27, and `dns fastest`, `dns auto` and `dns exclude|include` need 0.29, and `service-speed` needs 0.30, and `wifi nodes` and `wifi reconnect` need 0.31, and `stream` needs 0.33, and `pack` needs 0.34, and `hotspot` needs 0.36, `dns fallback` needs 0.37, and `router devices`, `router connect` and `router disconnect` need 0.38. If a command is unknown, the CLI is older: suggest `brew upgrade kilobat`, and use `kilobat --help` to find the older equivalent.

## Conventions

- Output is JSON when piped; add `--json` where a command supports it. Some expert commands always print JSON and reject the flag. Errors go to stderr with a non-zero exit code.
- Run one-shot commands only. `status --watch` and `radar --watch` never exit; suggest them for the user's own Terminal.
- `null` or missing fields mean "not measured", never zero or a success.
- History (`report`, `health-history`, `outages`, `speed-history`, `spots`, `scan-history`) is recorded by the Kilobat app while it runs. A gap is not evidence that nothing happened.
- macOS shows Wi-Fi names only to apps with Location permission. The CLI gets them from the running Kilobat app; without it, `ssid` is null and scans list "Hidden network". Signal, channel and band still work.
- App IDs are bundle IDs (`com.example.App`) or `process:<name>` for processes without one, exactly as `status` prints them.

## Safety: read freely, ask before changing

Run without asking: `diagnose`, `status`, `conditions`, `why`, `sessions`, `agent-log` (not `--watch`, which runs until stopped), `scan`, `channels`, `radar --json`, `wifi status`, `wifi known`, `wifi nodes`, `report`, `health-history`, `support-report` (without `--output`), `plan`, `credit`, `router`, `service-speed` (without `--test`), `hotspot` (without `--detect`, which asks Cloudflare and RIPE NCC), `week`, `calls`, `remote`, `stream`, `stream status`, `stream reports`, `cloud-gaming`, `cloud-gaming reports`, `proxy`, `proxy routes`, `proxy test`, `vpn`, `networks`, `rules`, `settings` with no value, `profile`, `limit` and `dns` with no argument, `dns fastest`, `dns-benchmark`, `http-check`, `trace`, `route-history`, `interfaces`, `ip` (without `--public`), `speed-compare`, `destinations`, `devices`, `outages`, `speed-history`, `spots`, `scan-history`, `doctor`, `bandwidth`, `access-policy`, `approvals`, `blocked-history`, `verdict`, `local check` (Bonjour and a light local discovery sweep; `--quick` skips sweep and ping), `stutter` (sends pings only), `starlink` (reads the local dish only), `portal` (two plain http checks to captive.apple.com and Google's connectivity check), `airdrop` with no argument, `bluetooth`, `bluetooth audio`, `uploads` with no argument, `protect` and `protect status`, `protect check <host>` (checks Kilobat's downloaded lists on the Mac, sends nothing). `portal open` opens the sign-in page in the user's browser: run it when they ask, `down` (asks public status pages only).

Name the exact change and wait for a yes before:

- `speedtest`: uses up to 80 MB. On a metered network add `--budget-mb 10` or less, or skip it.
- `stream check` (`check_stream_readiness`): uploads to Cloudflare for about 8 seconds (at most 16 MB). Fine on an unmetered network when the user asks whether they can go live; on a metered one it uses the last speed test unless `--measure` (`allow_metered`), which needs a yes. Never run it while OBS is streaming.
- `vpn leaks` (`check_vpn_leaks`): asks bash.ws, Cloudflare, a STUN server and api6.ipify.org, which see the user's address. Run it when the user asks about leaks with their VPN on; report each fix as written, and never call the VPN private because it passed.
- `ip --public` (`get_exit_path` with `public_ip`): asks Cloudflare for the public IP. Only when the user wants their public IP or VPN exit; report the registry country as registry data, and never say a VPN or a country makes them private or anonymous.
- `profile <name>`, `allow`/`block`/`reset <app>`, `limit <GB>`, `remember-network`, `networks forget`, `settings <name> <value>`, `block-host`/`unblock-host`: change saved settings.
- `wifi reconnect` (`reconnect_wifi`): drops Wi-Fi for a few seconds to move the Mac to a closer mesh node, which interrupts calls, downloads and your own connection. Only after `wifi nodes` reports `stuck`. `wifi nodes name <node> <name>` changes what Kilobat calls a node.
- `devices name <MAC> <name>`, `devices mark <MAC> mine|unknown|ignored`: change what Kilobat remembers about a device on the network. Mark one only with the user's answer; a new or unknown device is one they haven't identified, never proof of an intruder.
- `approve`/`deny <app>`: answer an app waiting to connect, which also saves a rule. Show the user its signer and destinations first.
- `wifi on|off|restart|disconnect|join`: change the connection. Turning Wi-Fi off or joining another network can cut off a remote session.
- `pack add`, `pack correct`, `pack end` (`set_data_pack`): change the user's data packs. A new pack on the same networks ends the old one, and its networks count as metered.
- `uploads hold <app>` (`hold_upload`): pauses another app's upload, such as Dropbox, until an unmetered or faster network, a time, or `uploads resume <app>`. Say which app and until when.
- `quit <PID>`: quits a process. `low-latency on [--for 15m|1h|2h] [--until-quit <app>]`: pauses AirDrop and AWDL while it runs (4 hours at most), with an administrator prompt; the user runs it in their own Terminal. `airdrop everyone|contacts|off`: opens System Settings for the user to change AirDrop visibility.
- `dns <provider>`, `dns flush`: change system settings behind a macOS administrator prompt. Record the current resolvers first.
- `dns auto on|off`, `dns exclude|include`: change how the app picks and switches DNS per network. Ask first.
- `settings kill-switch off` (`set_kill_switch` off): lets every app connect without the VPN. When `killSwitch.state` is `blocking`, requests fail because the VPN is down: ask the user to reconnect their VPN, and never turn the kill switch off to make your own requests work.
- `protect on|off`, `protect allow|unallow <host>`, `protect update` (`set_protection`): change Protection or download its lists. When Kilobat blocks one of your own requests (`list_blocked_connections` with `protection`), tell the user the host, the list and the kind; never turn Protection off or allow the site to get your request through. The user decides.
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
| Speed and bufferbloat, one connection or four (`--parallel`) | `kilobat speedtest --json [--budget-mb N] [--parallel]` | `run_speedtest` |
| What differed between two speed tests | `kilobat speed-compare --json` | `compare_speed_tests` |
| Where traffic leaves: VPN full, split or off, and the public IP when asked | `kilobat ip --json [--public]` | `get_exit_path` |
| Latency spikes and whether they repeat | `kilobat stutter --json [--target router\|internet]` | `run_stutter_test` |
| Ping to each data center of a game and the closest one: CS2, Dota 2, Deadlock, TF2, Fortnite, Apex, Siege, Dead by Daylight, Age of Empires II/III/IV, Age of Mythology, Brawlhalla, Hunt, Marathon (Valorant, League, Overwatch, CoD and others publish nothing to ping; `kilobat games` lists why) | `kilobat games --json [--game <game>]` | `get_game_ping` |
| The server of the match being played, any game, and its ping ("doesn't answer ping" is not loss) | `kilobat games --match --json` | `get_game_ping` with `game: match` |
| That data center's ping live on stream | `kilobat overlay --target cs2:fra` | — |
| AirDrop link and its users, AirDrop visibility, Bluetooth on 2.4 GHz | `kilobat airdrop --json`, `bluetooth --json` | `get_nearby_radios`, `set_airdrop_visibility` |
| Why Bluetooth headphones crackle now, and what to change in order | `kilobat bluetooth audio --json` (CLI 0.28) | `get_bluetooth_audio` |
| Can devices here see each other (casting, AirPlay, AirPrint), is the Mac on a device's own Wi-Fi, what makes a cast stutter | `kilobat local check --json [--quick]` | `check_local_network` |
| Usage history | `kilobat report --summary --json --days N` | `get_usage_report` |
| A router's or provider's figure beside Kilobat's for the same period | `kilobat report compare --total 40GB --period last-month --json` | `compare_usage` |
| Drops, past speed tests, health over time | `kilobat outages`, `speed-history`, `health-history --json` | `get_network_history` |
| A report for the ISP or IT desk, without site names by default | `kilobat support-report [--host h] [--include-sites]` | `create_support_report` (`include_site_names`) |
| Getting the speed you pay for, over a month and at the evening peak | `kilobat plan --json` | `get_plan_check` |
| Whether outages may earn a credit from the provider, and a claim to send (the user sends it; say "may qualify", never promise money) | `kilobat credit --json [--provider id] [--price n] [--month last]`, `credit --draft`, `credit providers` | `get_outage_credit` |
| Which DNS is fastest here, switching DNS automatically per network | `kilobat dns fastest --json` (CLI 0.29) | `get_fastest_dns` |
| Who runs a resolver and in which country, how often each answered over two weeks, a fallback for when the fastest fails | `kilobat dns fastest --json` (`label`, `jurisdiction`, `history`), `dns fallback <resolver>\|off` (ask first) (newer than CLI 0.33) | `get_fastest_dns`, `set_dns_fallback` |
| "Should I restart my router?", the connection worse over days, whether restarts helped | `kilobat router --json` | `get_router_checkup` |
| Slow Wi-Fi in one room on a mesh: is the Mac stuck on a far node while a closer one is in range? (needs the app's Location access) | `kilobat wifi nodes --json`; `wifi reconnect` moves it (ask first) (CLI 0.31) | `get_wifi_nodes`, `reconnect_wifi` |
| Speed test fine but GitHub, Steam, iCloud or npm downloads are slow; the provider's route to one service | `kilobat service-speed --json` (`--test <service>` downloads up to 25 MB: ask on metered links) | `get_service_speed` |
| A phone hotspot stuck at the same slow speed: is the carrier capping hotspot data? (say "looks like", never how to hide hotspot use) | `kilobat hotspot --json` (`--carrier <id>`; `hotspot carriers`) (CLI 0.34) | `get_hotspot_throttling` |
| How last week went: outages and whose side, frozen calls, blocks, metered data, against the week before | `kilobat week --json [--weeks-ago N]` | `get_weekly_digest` |
| Why a video call froze | `kilobat calls` | `get_call_reports` |
| Why OBS dropped frames, and whether it was the network, the encoder or the GPU | `kilobat stream reports --json [--limit N]` (CLI 0.33) | `get_stream_reports` |
| Why a cloud game stuttered, what is in the way now, its data per hour on a hotspot | `kilobat cloud-gaming [status\|reports] --json` | `get_cloud_gaming` |
| Whether the upload can carry the OBS bitrate before going live | `kilobat stream check --json [--bitrate <kbps>]` | `check_stream_readiness` |
| Why ssh typing lagged or a remote desktop froze; whether Mosh would help | `kilobat remote --json [--kind ssh\|mosh\|desktop]` (CLI 0.25) | `get_remote_session_reports` |
| Wi-Fi sign-in page (captive portal): sign-in needed, logged out, minutes left, each network's usual sign-in length | `kilobat portal --json`, `portal open` | `get_captive_portal` |
| Starlink: dish state, obstructions and which way, outages split into your setup or Starlink's network | `kilobat starlink --json [--hours N]` | `get_starlink_status` |
| Proxy apps, routes per app, node speed | `kilobat proxy`, `proxy routes`, `proxy test` | `get_proxy_status`, `test_proxy_nodes` |
| VPN leaks and proxy/VPN conflicts, and the VPN kill switch's state | `kilobat vpn` | `check_vpn` |
| VPN leak test: DNS, IPv6 and WebRTC/UDP around the VPN, with the setting to fix | `kilobat vpn leaks --json` | `check_vpn_leaks` |
| VPN kill switch: block new connections while the VPN is down (Pro) | `kilobat settings kill-switch on\|off` | `set_kill_switch` |
| Resolver comparison, one site's timing | `kilobat dns-benchmark`, `http-check <URL> --json` | `benchmark_dns`, `check_url` |
| Sockets per process, local devices, route | `kilobat destinations --json`, `devices --json`, `trace <host>` | `list_connections`, `list_devices`, `trace_route` |
| A device on the home network nobody recognises | `kilobat devices --json --all` (`status` new, known, unknown or ignored); `devices mark <MAC> mine` after the user confirms | `list_devices` with `all` |
| What kind of device is it; are there cameras? | `kilobat devices --json` (`type`, `confidence`, `typeLabel`, `tags`); `devices type <MAC> camera` after the user confirms. Medium means "probably", low "maybe": say so | `list_devices` |
| A call froze because of another Mac at home | `kilobat household --json` (paired Macs: `live.apps`, `busy`); pairing is only in the app | `get_household` |
| Device traffic at home, now and today | `kilobat router devices --json` (from the router connected in the app; `router.measures` wholeHome means no per-device counters, so never blame one device for `others`; UniFi clients and RouterOS queues can include LAN, so perDevice is not an internet-only total or a camera upload share); `router connect` prints the steps, never takes a password | `get_home_usage` |
| Matter, Thread or HomeKit devices won't set up or keep dropping; separate Thread networks; is the network ready for Matter (IPv6, Bonjour, isolation, subnet, 2.4 GHz) | `kilobat smarthome --json` (read-only; never sets up or controls a device) | `check_smart_home` |
| A route over time, hop by hop like mtr | `kilobat trace <host> --rounds 3`, `route-history --host <host> --json` | `trace_route` (`rounds`), `get_route_history` |
| Throughput to a device on the LAN running `iperf3 -s` (ask first) | `kilobat lan-test <host> --json` | `run_lan_test` |
| Networks and their remembered settings | `kilobat networks --json` | `list_networks` |
| A profile's rules | `kilobat rules --json [--profile name]` | `list_rules` |
| Select a profile | `kilobat profile travel\|work\|unrestricted` | `set_profile` |
| Allow, block or reset an app | `kilobat allow\|block\|reset <app> [--minutes N] [--profile name]` | `set_app_rule` |
| Data budget | `kilobat limit <GB>` | `set_budget` |
| A travel eSIM or other data pack: what's left and the day it runs out at the recent pace; add, correct with the carrier's figure, end (ask first) | `kilobat pack --json`, `pack add <GB> <days>`, `pack correct <GB-left>`, `pack end` (CLI 0.31) | `get_data_pack`, `set_data_pack` |
| Read, set or flush DNS | `kilobat dns [automatic\|cloudflare\|google\|quad9\|flush]` | `set_dns` |
| Settings such as new-app mode (allow, ask, block) and auto Travel | `kilobat settings [name value]` | `set_new_app_mode` for new apps |
| One app's traffic over an hour, day, week or month | `kilobat bandwidth <app> day --json` | `get_app_bandwidth` |
| What an app's developer says it connects to | `kilobat access-policy <bundle-id> --json` | `get_access_policy` |
| Apps waiting to connect, and answering them | `kilobat approvals --json`, `approve\|deny <app>` | `list_approvals`, `decide_approval` |
| Block a domain, IP or range | `kilobat block-host <target> [--app <id>]` | `block_destination` |
| What the filter blocked recently | `kilobat blocked-history` | `list_blocked_connections` |
| Every rule that matched a connection, the one that decided, and how to change each | `kilobat verdict <host> [--app <id>] --json`, `verdict --blocked [N]` | `explain_verdict` |
| Before fetching from, installing from or sending to an unfamiliar host: is it on a malware, phishing, scam or tracker list | `kilobat protect check <host> --json` (CLI 0.26) | `check_host_reputation` |
| Protection: threats, trackers in apps, the agent guard, list freshness | `kilobat protect --json`, `protect on\|off [--threats\|--trackers\|--agents]` | `get_protection_status`, `set_protection` |
| Metered, Low Data Mode, VPN, battery and heat, with advice | `kilobat conditions --json` | `get_conditions` |
| Updaters paused on metered networks, update bytes that got through, apps that update inside themselves | `kilobat updaters --json`, `updaters on\|off` | `get_updaters`, `set_updater_pause` |
| Apps uploading a lot now, time left only when the total is known; hold one for a better network (ask first) | `kilobat uploads --json`, `uploads hold <app> --until unmetered`, `uploads resume <app>` | `get_uploads`, `hold_upload` |
| Why one site, API or app can't connect; blocked on this network or not offered in this region | `kilobat why <url\|host> [--app <id>] [--error <text>] --json` | `diagnose_connection` |
| Is a service down for everyone | `kilobat down [service\|host…] --json` | `service_status` |
| What public reports expect in a country. A guide, not a live block check. Do not treat `expected` as `block`, and do not suggest a way around a restriction | `kilobat restrictions --country XX --json` | `get_country_restrictions` |
| What a process or host is, whether it's safe to block and what breaks | `kilobat explain <process\|host…> --json` | `explain_connection` |
| Keep the Mac awake for a run or a command | `kilobat awake --for 90m`, `awake --until-quiet`, `awake -- <command>` | `keep_awake`, `release_awake` |
| Keep it awake only while the user reads or presents in one app | `kilobat awake --while-frontmost <app> [--display]`, in the user's Terminal | None; it is the user's, not an agent's |
| Keep it awake whenever it's plugged in (the user's setting; ask first) | `kilobat settings awake-on-power on` (the screen follows `kilobat settings keep-screen-on`) | None; `get_conditions` reports `awakeOnPower` |
| Earlier awake sessions and AI agent runs | `kilobat sessions --json [--days N]` | `list_sessions` |
| Forget a site in existing host histories, only at the user's request | `kilobat history forget <host> [--subdomains] [--json]` | `clear_history` (`action: forget`, `host`, `subdomains`) |
| Clear existing host histories, only at the user's request | `kilobat history clear --hour [--json]` | `clear_history` (`action: clear`, `range`) |
| Every host an agent and its commands contacted | `kilobat agent-log --json [--all] [--flagged]` | `agent_network_log` |

Run `kilobat --help` for the full list, including expert tools.
