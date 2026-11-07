---
name: mac-app-firewall
description: Control which apps on a Mac may use the internet with Kilobat's firewall. Covers allowing or blocking apps, choosing whether apps without a rule are allowed, asked about or blocked, answering apps that are waiting for approval, blocking domains, IP addresses and ranges for every app or one app, checking what an app's developer says it connects to, Protection (known malware, phishing and scam hosts, trackers in apps, the agent guard, checking a host's reputation), and reading what the filter blocked. Use for "block this app", "stop X from phoning home", "approve the apps asking to connect", "block this domain", "is this host malicious" or "why was this site blocked".
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

Present each app with who signed it and where it wants to go, then let the user decide. `kilobat deny <app>` (MCP `decide_approval` with `block`) drops the held connections and saves a block rule in the current profile at once; `--minutes N` blocks for a while only. `kilobat approve <app>` doesn't allow anything by itself: Kilobat opens its approval window on that app, marked as requested from the command line, and the person confirms it there (`confirming: true`). Tell the user to look for it. Never encourage allowing an app whose signature isn't valid, or one the user doesn't recognize.

## 4. Block or allow a whole app

`kilobat block <id>` or `kilobat allow <id>` (MCP `set_app_rule`), with the `id` from `kilobat status --json` (`apps[].id`: a bundle ID or `process:<name>`). `--minutes N` makes it temporary; `kilobat reset <id>` removes the rule so the app follows the new-app mode.

## 5. Block a destination

`kilobat block-host <target>` blocks it for every app; add `--app <id>` to block it for one app only, as a connection rule in the active profile (MCP `block_destination`, with `action: unblock` to undo). A target is a domain (its subdomains are included; `*.example.com` means the same), an IP address or a CIDR range such as `203.0.113.0/24`. Domains match only when macOS gives the filter a hostname; addresses always match.

Before blocking a domain an app needs, check `kilobat access-policy <bundle-id> --json` (MCP `get_access_policy`): when the app declares its connections, `essential` ones and their `denyConsequences` say what breaks. Blocking a CDN or a shared address can affect other apps; prefer a domain or a per-app rule.

## 6. Check it worked

New connections follow the rule; an existing connection can continue until it reconnects. `kilobat blocked-history` (MCP `list_blocked_connections`) shows what the filter blocked, with the `reason` (app rule, blocklist entry, connection rule, budget, default deny, blocked when asked, or "Updater paused on a metered network" for an updater helper on a hotspot; see `mac-hotspot-data-saver`). `kilobat why <host> --app <id> --json` explains why one app can or can't reach a host, including Kilobat's own rules.

When the user asks why Kilobat allowed or blocked something, run `kilobat verdict <host> --app <id> --json` (MCP `explain_verdict`), or `kilobat verdict --blocked 5 --json` for the latest blocked attempts. It lists every rule that matched in the filter's order: `summary` is the deciding one ("Blocked by Protection: … is on the phishing list (…)"), and `rules` with `role: alsoMatched` are rules that applied too but came later, such as an allow rule under a threat or Send later holding an app on a hotspot. Each rule has `change.label` (where to change it in the app) and `change.command` (the CLI command). `recorded: true` means the filter recorded it as it blocked; otherwise it is the rules as they are now. Report the deciding rule first, then what else matched, and propose the change only with the user's yes: changing the deciding rule can let an also-matched rule decide instead.

## 7. Protection: known threats, trackers and the agent guard

Kilobat's Protection (CLI 0.26) blocks connections from every app to known malware, phishing and scam hosts and addresses, from public lists the app downloads daily to the Mac (HaGeZi Threat Intelligence Feeds, HaGeZi Fake, URLhaus), and names the app, or the command and agent, that tried. Threats and the agent guard are Kilobat Pro; tracker lists are free and off by default. It needs the app's network filter.

- **Before you fetch, install from or send data to an unfamiliar host**, check it: `kilobat protect check <host> --json` (MCP `check_host_reputation`). It is read-only and local: `listed`, the listed `entry`, `lists` with `category` and `licence`, `wouldBlock` and a `note`. Not listed doesn't mean safe; the lists hold known destinations only. If it is listed, don't use it; tell the user what you found.
- **Status:** `kilobat protect --json` (MCP `get_protection_status`): `threats`, `trackers`, `agentGuard`, entries, when each list was `updated`, `error` for a list whose update failed (the last good copy is kept), `allowedSites`, `pro`, `filterActive`.
- **What was blocked:** `list_blocked_connections` rows carry `protection` with `kind` (malware, phishing, scam, threat, tracker, agentGuard), `list`, `host`, and `agent` and `process` when an agent's tree made the connection. The agent network log marks those rows Blocked.
- **Precedence:** the VPN kill switch first; then threats and the agent guard, over every allow rule and profile; only the local network, Apple's own services, Kilobat and sites the user allowed are never blocked. Trackers lose to the user's allow rules (Unrestricted, a destination rule, an allowed site).
- **The agent guard** (`protect on --agents`) also keeps agents and everything they start from paste sites, file shares, tunnels and request catchers, raw public IP addresses typed into a command, and cuts a connection sending more than 10 MB to a host no agent on the Mac used before. If your own request is refused by it, say so and let the user decide.
- **Changes** (`protect on|off`, `protect on --trackers --list hagezi-light|hagezi-normal|oisd-small|stevenblack`, `protect allow|unallow <host>`, MCP `set_protection`) need the user's yes. Never turn Protection off or allow a site so that your own blocked request gets through.
- **Limits to state honestly:** it isn't antivirus and scans no files; names come from the app, the TLS server name or the HTTP Host line, so QUIC without a name and apps with their own encrypted DNS are checked by address only; data can still leave through hosts on no list, such as a GitHub repository.

