---
name: mac-app-firewall
description: Control which apps on a Mac may use the internet with Kilobat's firewall. Covers allowing or blocking apps, choosing whether apps without a rule are allowed, asked about or blocked, answering apps that are waiting for approval, blocking domains, IP addresses and ranges for every app or one app, checking what an app's developer says it connects to, and reading what the filter blocked. Use for "block this app", "stop X from phoning home", "approve the apps asking to connect" or "block this domain".
---

# Mac app firewall with Kilobat

Kilobat saves rules locally and enforces them with its signed network filter, which the user turns on in the app (Kilobat Pro). The CLI and MCP tools change the same rules the app uses. Every change here needs the user's yes: name the exact rule first. The `kilobat` skill lists the general rules.

Commands need CLI 0.5 (`kilobat --version`); if one is unknown, suggest `brew upgrade kilobat`.

## 1. Read the current policy

`kilobat rules --json` (MCP `list_rules`) returns the active `profile`, `newApps` (`allow`, `ask` or `block`), `allowApple`, saved app `rules`, `temporaryRules`, `connectionRules` (one app to one host), `blockedHosts` (blocked for every app), the size of an imported blocklist and `filteringActive`. When `filteringActive` is false or null, rules are saved but not enforced: say "saved", not "blocked". In the Unrestricted profile every app is allowed; rules need Travel or Work.

## 2. How apps without a rule are treated

`kilobat settings new-apps allow|ask|block` (MCP `set_new_app_mode`):

- `allow`: apps without a rule connect. Block rules and blocked hosts still apply.
- `ask`: an app's first connections are held, and the Kilobat app asks the user to allow or block it, showing its signer, path and destinations. Unanswered connections give up after a minute and the app asks again. If the Kilobat app isn't running, new apps are allowed rather than held.
- `block`: only apps with an allow rule connect. Good for a phone hotspot or a strict work setup; warn that new apps fail silently until allowed.

`kilobat settings apple-software on` lets Apple's own software (macOS services, Safari, Mail…) through when asking or blocking, which keeps updates, iCloud and the system working. On a hotspot it also lets iCloud and software updates use data; mention that.

## 3. Apps waiting for approval

With `new-apps` set to `ask`, `kilobat approvals --json` (MCP `list_approvals`) lists waiting apps: `app` (the rule key), `process`, `path`, `signer`, `signatureValid`, `requests`, `waiting` and `destinations` (`host`, `address`, `port`). It needs the Kilobat app running.

Present each app with who signed it and where it wants to go, then let the user decide. `kilobat approve <app>` or `kilobat deny <app>` (MCP `decide_approval`) releases or drops the held connections and saves the rule in the current profile; `--minutes N` decides for a while only. Never approve an app whose signature isn't valid, or one the user doesn't recognize, without saying so plainly.

## 4. Block or allow a whole app

`kilobat block <id>` or `kilobat allow <id>` (MCP `set_app_rule`), with the `id` from `kilobat status --json` (`apps[].id`: a bundle ID or `process:<name>`). `--minutes N` makes it temporary; `kilobat reset <id>` removes the rule so the app follows the new-app mode.

## 5. Block a destination

`kilobat block-host <target>` blocks it for every app; add `--app <id>` to block it for one app only, as a connection rule in the active profile (MCP `block_destination`, with `action: unblock` to undo). A target is a domain (its subdomains are included; `*.example.com` means the same), an IP address or a CIDR range such as `203.0.113.0/24`. Domains match only when macOS gives the filter a hostname; addresses always match.

Before blocking a domain an app needs, check `kilobat access-policy <bundle-id> --json` (MCP `get_access_policy`): when the app declares its connections, `essential` ones and their `denyConsequences` say what breaks. Blocking a CDN or a shared address can affect other apps; prefer a domain or a per-app rule.

## 6. Check it worked

New connections follow the rule; an existing connection can continue until it reconnects. `kilobat blocked-history` (MCP `list_blocked_connections`) shows what the filter blocked, with the `reason` (app rule, blocklist entry, connection rule, budget, default deny or blocked when asked). `kilobat why <host> --app <id> --json` explains why one app can or can't reach a host, including Kilobat's own rules.
