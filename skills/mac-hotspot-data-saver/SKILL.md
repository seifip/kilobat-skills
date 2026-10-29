---
name: mac-hotspot-data-saver
description: Save mobile data on a Mac tethered to a phone hotspot, on plane or hotel Wi-Fi, or with a data cap. Uses Kilobat to show which apps use data now and today, switch to its Travel profile, set a data budget that resets with the plan, cap single apps and block background apps. Use for "what ate my data", roaming, tethering and metered connections.
---

# Save hotspot data on a Mac with Kilobat

A Mac on a phone hotspot behaves as if it were on home broadband: cloud sync, photo libraries, updates and backups all start at once. This skill finds what is using data and stops what the user does not need, with the user's approval at every step.

## Setup

Use the Kilobat MCP tools if connected, otherwise the `kilobat` CLI (`brew install seifip/kilobat/kilobat`, macOS 14+; ask before installing). Commands here need CLI 0.2 or later, and `conditions` and `sessions` need 0.4 (`kilobat --version`); if one is unknown, suggest `brew upgrade kilobat`. Reading is safe; every change below needs a clear yes from the user. The `kilobat` skill lists the rules.

Blocking and budgets are saved by the CLI and enforced by the Kilobat app with its network filter enabled (Kilobat Pro). Without it, rules are saved but nothing is stopped. `kilobat doctor` says whether the app is running and the filter is on; say "saved", not "blocked", unless it is.

## 1. Confirm the connection is metered

Run `kilobat conditions --json` (MCP: `get_conditions`). `metered` is what Kilobat acts on; `osMetered` is what macOS says and `meteredSource` is `macOS` or `you` when the user set the network's data cost. macOS flags an iPhone hotspot and some Android ones, but often not other Android hotspots, a phone tethered over USB or Ethernet (it shows as `network: "Ethernet"`), or plane, hotel and roaming Wi-Fi. Take the user's word for it.

When the user says the connection costs data and `metered` is false, offer to mark it (with approval): `kilobat networks cost metered` (MCP: `set_network_cost` with `cost: metered`). Kilobat then treats it like a hotspot on every visit: it switches to Travel when automatic Travel is on and no profile is remembered for that network, and skips scheduled speed tests. `kilobat networks cost automatic` follows macOS again. Kilobat refuses a network it can't tell apart; for Wi-Fi that means the app needs Location access to see the name. Needs CLI 0.6 or newer.

The same `conditions` output has `lowDataMode` (the Low Data Mode switch for this network), `vpn` and battery. Check it before any large download or upload you are about to start yourself, such as a model, a container image or a dependency install, and ask first when `metered` or `lowDataMode` is true.

Do not run a speed test on a metered connection unless the user wants one. If they do, use `--budget-mb 10` or less.

## 1b. Updaters pause by themselves

On a metered network, the Kilobat app's network filter pauses updater helpers: Google Updater and Keystone, Microsoft AutoUpdate, macOS Software Update, App Store downloads, Sparkle's download service and Sparkle's and Squirrel's installers (Autoupdate, Updater.app, ShipIt). The apps they update keep working, and updates carry on once the network isn't metered. It is on by default; `conditions` reports `updaters.paused`. `kilobat updaters --json` (MCP: `get_updaters`; CLI 0.6 or newer) gives the detail:

- `covered`: the helpers Kilobat recognizes. `downloads: false` marks installers that only install what the app already downloaded.
- `gotThrough`: update bytes on this network: an updater's own, or a System downloads service such as `nsurlsessiond` that connected for one (`shared: true`, so its bytes include other downloads). Null when the app isn't running; that isn't the same as nothing. The filter stops new connections, so a download already running when the network became metered can finish.
- `pausedConnections`: updater connections stopped in the last 24 hours; blocked connections read "Updater paused on a metered network".
- `inProcess`: installed apps that download updates inside their own process (most Sparkle and Electron apps). Say plainly that Kilobat can't pause these without blocking the app; the user can turn off automatic updates in that app, or block it for a while (step 5).

If the user needs one update now, a temporary allow lets it through: `kilobat allow com.google.GoogleUpdater --minutes 30`. `kilobat updaters off` (MCP: `set_updater_pause` with `pause: off`, with approval) stops pausing everywhere. Kilobat changes no other app's settings, staging folders or System Settings, so there is nothing to restore.

## 2. Show what is using data

- Right now: from the same `status` output, sort `apps` by `downloadBytesPerSecond + uploadBytesPerSecond`. Run it again 30 seconds later; background sync comes in bursts.
- Today: `kilobat report --summary --json --days 1` gives `downloadBytes`, `uploadBytes` and `topApps` (10 apps by bytes). Use `--days 7` to compare with a normal week.
- Only Internet bytes use the plan. Per app, compare `internetBytes` with `localBytes` (AirDrop, a NAS, a printer, a phone on the same hotspot): local traffic doesn't count against the budget, so a big AirDrop transfer isn't what ate the data. VPN traffic is counted once, where the tunnel leaves the Mac.
- System downloads: `nsurlsessiond`, `mobileassetd` (Apple Intelligence and Siri models), `softwareupdated`, `appstored`, `appstoreagent`, `cloudd` and `bird` download for apps and macOS. With the filter on, `onBehalfOf` names the app a download was for; otherwise it is null and unknown. Travel blocks these unless *Allow Apple software* is on.

Name the top apps in plain words, with their `id`. Common culprits on macOS: iCloud Drive and Photos sync, Software Update, App Store downloads, Dropbox, Google Drive, OneDrive, browser updaters, game launchers and cloud backups. Uploads count too; a photo library sync can use more than browsing.

AI coding agents: `kilobat sessions --json --days 1` (MCP: `list_sessions`) lists each Claude Code, Codex or other agent run the app recorded, with `received`, `sent` and bytes per helper process in `apps` (npm, git, pip and the like). Agents that install dependencies or pull images can use gigabytes; on a hotspot, suggest pausing those until the user is back on their usual network.

History is recorded only while the Kilobat app runs, so gaps do not prove nothing was used.

## 3. Offer the Travel profile

Travel is Kilobat's "only the essentials" profile. Each profile keeps its own rules. Apps without a rule follow the "new-apps" setting (`kilobat settings new-apps`), which is `block` unless the user changed it. Build the allow list before switching, so nothing the user needs is cut off in between:

1. Run `kilobat rules --json --profile travel` to see Travel's `rules` and `defaultAllow`, and whether it is already `active`.
2. From what the user is doing and the `id` values in `status`, propose a short allow list, for example the browser, Slack and Zoom. Save the approved ones to Travel without switching: `kilobat allow com.tinyspeck.slackmacgap --profile travel`.
3. Then, with approval: `kilobat profile travel`.

To undo one rule later: `kilobat reset <id> --profile travel`.

The Kilobat app can switch to Travel automatically on metered networks. Check with `kilobat settings auto-travel` and offer `kilobat settings auto-travel on` (with approval) if it is off.

## 4. Set a budget

`kilobat limit <GB>` sets the current network's budget (0.1 to 10000 GB; `kilobat limit` reads it with this cycle's use). With the filter on, once the budget is used up Kilobat stops new connections in the Travel and Work profiles; in Unrestricted it only tracks usage. Suggest a number from the user's plan, for example 2 GB for a day of tethering.

Match the budget to the plan with `--cycle` (CLI 0.6 or newer, so check `kilobat limit --help`; MCP: `set_budget` with `cycle`, `reset_day`, `alerts`):

- `session` (the default) counts until the user resets it in the app.
- `daily`, `weekly --reset-day 1` (1 Monday to 7 Sunday) or `monthly --reset-day 14` (the day the plan renews; 31 means the last day of shorter months). A cycle counts the usage history of that network, so it survives restarts.
- `--alert 50,80,100` sets which percentages notify, once per cycle; 80 and 100 are the default for cycles, `none` turns them off.

For one heavy app, offer a quota instead of a block: `kilobat quota com.getdropbox.dropbox 1 --upload` lets Dropbox upload 1 GB per budget cycle on this network (daily when the budget is per session); `--all-networks` counts everywhere, `kilobat quota --list --json` shows use, `kilobat quota <id> --remove` takes it off. With the filter on, the app's new connections stop when it is used up; a temporary allow still gets through. MCP: `set_app_quota`.

Kilobat samples traffic about once a second, so budgets and quotas are approximate: say "about", and expect an app to run slightly over before it stops. Its numbers are what this Mac sent and received, not the carrier's bill: the phone's own use, other devices on the hotspot and the carrier's rounding are not in them. The budget counts what Wi-Fi or Ethernet carried to the Internet, including `unattributedBytes` that no app accounts for, and leaves out local network and AirDrop traffic.

When the carrier's app shows a different number, compare the same period instead of guessing: `kilobat report compare --total 3.2GB --period this-month --network "<hotspot name>" --json` (MCP: `compare_usage`; CLI 0.6 or newer). It reports Kilobat's total in the carrier's unit, the hours it recorded and its gaps, and likely reasons such as the phone's own use and binary GB. `mac-bandwidth-monitor` has the full steps. Never call the carrier's figure wrong.

`kilobat remember-network` saves the current profile and budget for this network, so Kilobat applies them the next time the Mac joins it. A remembered profile wins over automatic Travel. `kilobat networks` lists networks with their remembered profile, budget and data cost; `kilobat networks forget <name>` drops them.

`status --json` includes `budgetUsed` (0–1) and `filteringActive` when the Kilobat app is running, and `budget` (limit, `cycle`, `usedBytes` this cycle, `resets`) and `quotas` always. If `filteringActive` is false, the budget and rules are saved but nothing is stopped.

## 5. Block specific apps

For a single heavy app the user does not need now, block it temporarily rather than for good: `kilobat block com.example.App --minutes 120` (1–1440 minutes). Without `--minutes` the rule stays until removed with `kilobat allow`. Rules apply to the current profile, so check which one is active first.

Settings outside Kilobat the user can change themselves:

- System Settings › Wi-Fi › Details for the hotspot network › Low Data Mode, which asks macOS and some apps to pause background work.
- Pausing iCloud Photos sync, Dropbox or Google Drive from their own menus.
- Turning off automatic updates in System Settings › General › Software Update, or in apps that update from inside themselves (`inProcess` above), while travelling.

## 6. Check it worked

Run `kilobat status --json` again a minute later and compare the totals and top apps with step 2. Report what changed, what is still using data, and how much of the budget is left. When the user is back on their normal network, remind them to switch to their usual profile (`kilobat profile work` or `unrestricted`, with approval) so blocked apps catch up.
