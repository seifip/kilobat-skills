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
- `apps[]`: `id` (bundle ID, or `process:<name>` for processes without one), `name`, per-app `downloadBytesPerSecond` and `uploadBytesPerSecond`, `sessionBytes` (since sampling started), `processIDs` and `allowedRule` (the saved rule, not proof of enforcement).
- `network`, `metered`, `profile`.

Sort by total rate and name the top apps with numbers in readable units (KB/s, MB/s). Traffic is bursty: take two or three snapshots 20–30 seconds apart before calling an app the culprit. For continuous watching, suggest the user run `kilobat status --watch` in their own Terminal (`s` sorts, `/` searches, `q` quits).

Background work on macOS often runs under system processes rather than the app's name: for example `nsurlsessiond` (downloads for many apps), `cloudd` and `bird` (iCloud), `softwareupdated` (system updates). Explain the likely owner instead of only quoting the process name.

## 2. Look back in time

- `kilobat report --summary --json --days 7` returns `downloadBytes`, `uploadBytes`, `daily[]` (`day`, `downloadBytes`, `uploadBytes`) and `topApps[]` (10 apps by `bytes`). `--days` takes 1–365 (the app keeps up to a year, set in Settings).
- `kilobat report --csv --days N` exports one row per app per sample (`timestamp,app,download_bytes,upload_bytes,bytes`; the row with an empty `app` is the whole Mac) for a spreadsheet. `--json` without `--summary` gives raw samples, which can be large. Prefer `--summary` unless the user asks for raw data.

Point out unusual days and compare the top apps with the live snapshot.

## 3. See where an app connects

`kilobat destinations --json --pid <PID>` lists that process's open sockets: `pid`, `process`, `transport`, `endpoint` (local -> remote address and port) and `state`. Take the PID from `processIDs` in step 1, or omit `--pid` to see every process. Remote addresses are IPs; say which service they probably belong to only when the hostname or a reverse lookup supports it.

## 4. Check an unknown process

When the user does not recognize a process, run `kilobat signature <PID>`. It prints the executable path, whether the code signature verifies, and the signer. A valid signature identifies who signed the code; it does not prove the traffic is safe. Unsigned code in an unusual location is worth a closer look; recommend the user's security tools rather than deleting anything.

## 5. Act, with approval

- Pause the app from its own settings, for example sync or auto-update options.
- Quit it: `kilobat quit <PID>` asks it to quit; `--force` kills it. Only for a process the user named or agreed to stop; unsaved work may be lost.
- Temporarily block it: `kilobat block <id> --minutes 60`, using the `id` from `status`. Without `--minutes` the rule stays until `kilobat allow <id>`. Rules belong to the current profile; in the Unrestricted profile everything is allowed, so blocking needs Travel or Work.
- Rules are enforced only by the Kilobat app with its network filter on (Kilobat Pro). Otherwise they are saved, not applied; say so.

## Other questions

- VPN in use: `kilobat interfaces --json` lists physical and VPN interfaces with `received`, `sent` and `up`. VPN counters overlap with the physical ones; do not add them.
- Other devices on the network: `kilobat devices --json` lists the neighbor cache (`address`, `mac`, `interface`, `name`). `kilobat devices name <MAC> <name>` labels one (with approval). Kilobat cannot measure their traffic from the Mac; a router with UPnP can report whole-home totals via `kilobat router-counters <control-URL>` if the user provides the URL.
- Is it the network or the app? If every app is slow and nothing is using much bandwidth, switch to `mac-network-troubleshooting`.
