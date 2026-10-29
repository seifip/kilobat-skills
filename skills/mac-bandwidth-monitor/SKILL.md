---
name: mac-bandwidth-monitor
description: See which apps and processes use the internet on a Mac, how much they upload and download, and where they connect, with Kilobat. Covers live per-app bandwidth, daily usage history, background uploads, unknown processes and exporting usage to CSV. Use for "what is using my bandwidth", a slow network caused by one app, or checking a suspicious process.
---

# Mac bandwidth monitor with Kilobat

macOS has no built-in per-app network usage view. Kilobat measures traffic per app, keeps a local history and shows the hosts each process talks to. This skill answers "who is using the connection, how much, and to where".

## Setup

Use the Kilobat MCP tools if connected, otherwise the `kilobat` CLI (`brew install seifip/kilobat/kilobat`, macOS 14+; ask before installing). Commands here need CLI 0.2 or later (`kilobat --version`); if one is unknown, suggest `brew upgrade kilobat`. Everything here only reads, except blocking in step 5, which needs a yes. The `kilobat` skill lists the rules.

Live traffic works without the Kilobat app. History needs the app running; it records locally and gaps mean it was not running, not that nothing was used.

## 1. Snapshot live traffic

Run `kilobat status --json`; it samples for a few seconds. Read:

- `downloadBytesPerSecond`, `uploadBytesPerSecond`: the whole Mac.
- `apps[]`: `id` (bundle ID, or `process:<name>` for processes without one), `name`, per-app `downloadBytesPerSecond` and `uploadBytesPerSecond`, `sessionBytes` (since sampling started), `processIDs` and `allowedRule` (the saved rule, not proof of enforcement). `sessionBytes` is everything the app moved; `internetBytes` and `localBytes` split it into Internet and local network (LAN, AirDrop), and `tunnelBytes` is the part inside a VPN. Needs CLI 0.6 or newer; older versions lack these fields.
- `executablePath`: present for a command-line tool that is neither an app nor part of macOS (node, python, a binary in the home folder or Homebrew). Mention it when the user doesn't recognise the process.
- Top-level `sessionBytes` counts the Mac's Internet bytes only, from the Wi-Fi and Ethernet counters when `countsInterfaces` is true; `localBytes` is left out of it, and `unattributedBytes` is Internet traffic no process accounts for: mostly packet headers and acknowledgments (usually 5–10% on top of the apps), plus short-lived processes, Internet Sharing or a VPN macOS runs itself. Never assign unattributed bytes to an app.
- `network`, `metered`, `profile`.

Sort by total rate and name the top apps with numbers in readable units (KB/s, MB/s). Traffic is bursty: take two or three snapshots 20–30 seconds apart before calling an app the culprit. For continuous watching, suggest the user run `kilobat status --watch` in their own Terminal (`s` sorts, `/` searches, `q` quits).

Background work on macOS often runs under system processes rather than the app's name: for example `nsurlsessiond` (downloads for many apps), `mobileassetd` (Apple Intelligence and Siri models, voices), `cloudd` and `bird` (iCloud), `softwareupdated` (system updates). These rows carry `systemDownload` (what the service does) and `onBehalfOf`: the bundle IDs the network filter saw it download for, such as `com.apple.Music`. `onBehalfOf: null` means the filter is off, so which app asked is unknown; say so rather than guessing. With a proxy (system proxy or `HTTPS_PROXY`), traffic shows under the proxy process and the app behind it can't be known.

## 2. Look back in time

- `kilobat report --summary --json --days 7` returns `downloadBytes`, `uploadBytes`, `daily[]` (`day`, `downloadBytes`, `uploadBytes`) and `topApps[]` (10 apps by `bytes`), all Internet bytes, and `localBytes`, the local network and AirDrop traffic left out (recorded since Kilobat 1.3). `--days` takes 1–365 (the app keeps up to a year, set in Settings).
- `kilobat report --csv --days N` exports one row per app per sample (`timestamp,app,download_bytes,upload_bytes,bytes`; the row with an empty `app` is the whole Mac) for a spreadsheet. `--json` without `--summary` gives raw samples, which can be large. Prefer `--summary` unless the user asks for raw data.

Point out unusual days and compare the top apps with the live snapshot.

### When a router or provider shows a different total

`kilobat report compare --total 40GB --from 2026-10-01 --to 2026-10-31 --json` (MCP: `compare_usage` with `total`, `unit`, `direction`, `from` and `to` or `period`, and `network`) sets the router's, ISP's or carrier's figure beside what Kilobat recorded for the same period. Needs CLI 0.6 or newer. It only reads.

1. Ask for the figure exactly as shown, its unit and what it counts: download, upload or both (most plans count both). Pass the unit as written; many providers write GB for binary GiB, so read `alternateBytes`, the figure in the other system.
2. Match the period: the billing-cycle dates, or `--period last-month`, `this-month`, `last-7-days`, `yesterday`, `today` or `last-24-hours`. A date-only `--to` includes that day. If the provider counts days in UTC or another zone, pass ISO 8601 with an offset (`2026-10-01T00:00:00Z`).
3. Limit it to the network the figure covers with `--network <name>` (names from `kilobat networks`; repeat for more). Networks with the same name are all counted.
4. Read `kilobatBytes` and `kilobatAmount` (the same unit), `differenceBytes` (the figure minus Kilobat), `recordedHours` of `periodHours`, `gaps` (`asleep`: the Mac slept, recorded from 1.3 on; `notRecorded`: off, offline or Kilobat not running, and sleep before 1.3; `otherNetwork`; `beforeHistory`; `notYet`), `localBytes` (local network and AirDrop, left out), `unattributedBytes` (headers and the like, already counted) and `reasons`, most likely first.
5. Relay the reasons in plain words. Kilobat counts only this Mac and only while it runs, so a router or provider total for the whole network is normally higher. Never say either figure is wrong, and don't conclude malware, fraud or a billing error from a mismatch. Offer `--csv` or `--json` for a file with the scope and every gap that the user can send their provider.

For one app over time, `kilobat bandwidth "<app name>" day --json` (or `hour`, `week`, `month`; `--back 1` is the previous period) returns `sentBytes`, `receivedBytes`, `peakBytesPerSecond` with `peakAt`, and `buckets[]` with sent and received per minute, hour, six hours or day. Use the names from `topApps`; helpers count under the app that owns them. Omit the app for the whole Mac. History from before Kilobat 1.2 has no sent/received split per app, so it is estimated from the whole Mac's ratio; say so when it matters.

## 3. See where an app connects

`kilobat destinations --json --pid <PID>` lists that process's open sockets: `pid`, `process`, `transport`, `endpoint` (local -> remote address and port) and `state`. Take the PID from `processIDs` in step 1, or omit `--pid` to see every process. Remote addresses are IPs; say which service they probably belong to only when the hostname or a reverse lookup supports it. The Kilobat app's Connections map shows each address's registration country; CDN and VPN addresses show their owner's country, not where the server stands.

`kilobat access-policy <bundle-id> --json` shows what the app's developer says it connects to and why (`connections[]` with `hosts`, `purpose`, `denyConsequences`, `essential`), when the app ships an Internet Access Policy (`declared` is false otherwise). Use it to tell expected traffic from surprises.

## 4. Check an unknown process

When the user does not recognize a process, run `kilobat signature <PID>`. It prints the executable path, whether the code signature verifies, and the signer. A valid signature identifies who signed the code; it does not prove the traffic is safe. Unsigned code in an unusual location is worth a closer look; recommend the user's security tools rather than deleting anything.

## 5. Act, with approval

- Pause the app from its own settings, for example sync or auto-update options.
- Quit it: `kilobat quit <PID>` asks it to quit; `--force` kills it. Only for a process the user named or agreed to stop; unsaved work may be lost.
- Block one destination instead of the whole app: `kilobat block-host <domain|IP> --app <id>`; see `mac-app-firewall`.
- Temporarily block it: `kilobat block <id> --minutes 60`, using the `id` from `status`. Without `--minutes` the rule stays until `kilobat allow <id>`. Rules belong to the current profile; in the Unrestricted profile everything is allowed, so blocking needs Travel or Work.
- Rules are enforced only by the Kilobat app with its network filter on (Kilobat Pro). Otherwise they are saved, not applied; say so.

## Other questions

- VPN in use: `kilobat interfaces --json` lists physical and VPN interfaces with `received`, `sent` and `up`. VPN counters overlap with the physical ones; do not add them.
- Other devices on the network: `kilobat devices --json` lists the neighbor cache (`address`, `mac`, `interface`, `name`). `kilobat devices name <MAC> <name>` labels one (with approval). Kilobat cannot measure their traffic from the Mac; a router with UPnP can report whole-home totals via `kilobat router-counters <control-URL>` if the user provides the URL.
- Is it the network or the app? If every app is slow and nothing is using much bandwidth, switch to `mac-network-troubleshooting`.
