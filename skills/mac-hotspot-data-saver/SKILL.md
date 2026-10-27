---
name: mac-hotspot-data-saver
description: Save mobile data on a Mac tethered to a phone hotspot, on plane or hotel Wi-Fi, or with a data cap. Uses Kilobat to show which apps use data now and today, switch to its Travel profile, set a data budget and block background apps. Use for "what ate my data", roaming, tethering and metered connections.
---

# Save hotspot data on a Mac with Kilobat

A Mac on a phone hotspot behaves as if it were on home broadband: cloud sync, photo libraries, updates and backups all start at once. This skill finds what is using data and stops what the user does not need, with the user's approval at every step.

## Setup

Use the Kilobat MCP tools if connected, otherwise the `kilobat` CLI (`brew install seifip/kilobat/kilobat`, macOS 14+; ask before installing). Reading is safe; every change below needs a clear yes from the user. The `kilobat` skill lists the rules.

Blocking and budgets are saved by the CLI and enforced by the Kilobat app with its network filter enabled (Kilobat Pro). Without it, rules are saved but nothing is stopped. The CLI cannot see the filter, so say "saved" and ask the user to confirm the filter is on in the app.

## 1. Confirm the connection is metered

Run `kilobat status --json`. `metered: true` means macOS marks the connection as expensive, which is typical for a phone hotspot. `network: "Cellular"` is the same. Plane, hotel and roaming Wi-Fi often are not marked; take the user's word for it.

Do not run a speed test on a metered connection unless the user wants one. If they do, use `--budget-mb 10` or less.

## 2. Show what is using data

- Right now: from the same `status` output, sort `apps` by `downloadBytesPerSecond + uploadBytesPerSecond`. Run it again 30 seconds later; background sync comes in bursts.
- Today: `kilobat report --summary --json --days 1` gives `downloadBytes`, `uploadBytes` and `topApps` (10 apps by bytes). Use `--days 7` to compare with a normal week.

Name the top apps in plain words, with their `id`. Common culprits on macOS: iCloud Drive and Photos sync, Software Update, App Store downloads, Dropbox, Google Drive, OneDrive, browser updaters, game launchers and cloud backups. Uploads count too; a photo library sync can use more than browsing.

History is recorded only while the Kilobat app runs, so gaps do not prove nothing was used.

## 3. Offer the Travel profile

Travel is Kilobat's "only the essentials" profile. Each profile keeps its own rules, and `rules`, `allow` and `block` act on the active profile only. Apps without a rule follow the app's "Allow new apps by default" setting, which is off unless the user turned it on.

1. Run `kilobat rules --json`. If `profile` is already `travel`, its `rules` and `defaultAllow` show what is allowed; skip to step 3.
2. Otherwise, explain that Travel may block everything without an allow rule, including apps in use right now, and ask to switch: `kilobat profile travel`. Then run `kilobat rules --json` to see Travel's rules.
3. From what the user is doing and the `id` values in `status`, propose a short allow list, for example the browser, Slack and Zoom: `kilobat allow com.tinyspeck.slackmacgap`. Save the ones the user approves.

The Kilobat app can switch to Travel automatically on metered networks ("Switch to Travel on metered networks" in Profiles, on by default). Mention it rather than assuming it is on.

## 4. Set a budget

`kilobat limit <GB>` sets the session budget (0.1 to 10000 GB; `kilobat limit` reads it). With the filter on, once the budget is used up Kilobat stops new connections in the Travel and Work profiles; in Unrestricted it only tracks usage. Suggest a number from the user's plan, for example 2 GB for a day of tethering.

`kilobat remember-network` saves the current profile and budget for this network, so Kilobat applies them the next time the Mac joins it.

## 5. Block specific apps

For a single heavy app the user does not need now, block it temporarily rather than for good: `kilobat block com.example.App --minutes 120` (1–1440 minutes). Without `--minutes` the rule stays until removed with `kilobat allow`. Rules apply to the current profile, so check which one is active first.

Settings outside Kilobat the user can change themselves:

- System Settings › Wi-Fi › Details for the hotspot network › Low Data Mode, which asks macOS and some apps to pause background work.
- Pausing iCloud Photos sync, Dropbox or Google Drive from their own menus.
- Turning off automatic updates in System Settings › General › Software Update while travelling.

## 6. Check it worked

Run `kilobat status --json` again a minute later and compare the totals and top apps with step 2. Report what changed, what is still using data, and how much of the budget is left. When the user is back on their normal network, remind them to switch to their usual profile (`kilobat profile work` or `unrestricted`, with approval) so blocked apps catch up.
