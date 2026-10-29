---
name: mac-agent-runs
description: Keep a Mac awake for long AI agent runs, builds and downloads with Kilobat, check whether the connection suits a big transfer, and find out when a run finished, stalled or hit a network change. Covers kilobat awake, keep_awake, get_conditions, kilobat sessions and kilobat why. Use for "don't let my Mac sleep during this", "is this a good time to download", "did the agent finish" or "why can't the agent reach this API".
---

# Long agent runs on a Mac with Kilobat

A Mac that idle-sleeps halfway through a build, a model download or an agent's run loses the work. Kilobat keeps it awake on purpose, ends the hold when the work goes quiet, and records each run with the data it used and the network changes it went through. This skill plans a long run, keeps it alive and reads what happened.

## Setup

Use the Kilobat MCP tools if connected, otherwise the `kilobat` CLI (`brew install seifip/kilobat/kilobat`, macOS 14+; ask before installing). Commands here need CLI 0.4 or later (`kilobat --version`); if one is unknown, suggest `brew upgrade kilobat`. Reading is safe. Keeping the Mac awake changes no settings and is low risk; say what you are doing and for how long, and ask first on battery. The `kilobat` skill lists the rules.

Kilobat stops idle sleep only. Closing the lid still sleeps the Mac; never suggest `pmset` or other tricks to override that.

## 1. Check conditions before a big transfer or a long run

Call `get_conditions` (CLI: `kilobat conditions --json`, about 2 seconds). Read:

- `metered` (hotspot or a network marked expensive) and `lowDataMode`: ask before downloading or uploading more than a few hundred MB, and prefer smaller artifacts or a later time.
- `vpn` and `vpnInterface`: some hosts are blocked or slower through a VPN.
- `internetLatencyMs` and `internetLossPercent`: high loss means retries and timeouts; `null` means the ping went unanswered, not that the internet is down.
- `power`: `onAC`, `batteryPercent`, `thermal` (`nominal` to `critical`) and `lowPowerMode`. On battery below about 25%, suggest plugging in before a long run.
- `keptAwake` and `awakeReasons`: whether Kilobat (the app, the CLI or this MCP server) already keeps the Mac awake.
- `advice`: one line that sums it up. Quote it.

## 2. Keep the Mac awake

Pick the narrowest form:

- One command: `kilobat awake -- <command> [args]` keeps the Mac awake while it runs and exits with its status. A summary with the bytes its processes moved goes to stderr (`--json` for JSON). Prefer this for builds, test suites and downloads started from a shell.
- A set time: MCP `keep_awake` with `minutes` (1–240) and a short `reason`; call it again to extend, `release_awake` when done. The hold also ends when the MCP server stops. CLI: `kilobat awake --for 90m`, which runs in the foreground.
- Until transfers finish: `kilobat awake --until-quiet` ends once the Mac stays below 16 KB/s for 3 minutes (`--quiet 1…60` minutes). `--app <id>` watches one app, using the `id` from `kilobat status --json`.
- `--display` keeps the screen on too; only when the user wants to watch.

Plain `kilobat awake` runs until Ctrl-C, so suggest it for the user's own Terminal. From an agent, prefer `awake -- <command>`, `--for` or `keep_awake`, which end on their own.

The Kilobat app can also keep the Mac awake while it sees an AI agent (Claude Code, Codex, Gemini CLI, Aider and others) moving data, and notify when the agent goes quiet. That is the user's setting in the app.

## 3. Read what happened

`list_sessions` (CLI: `kilobat sessions --json --days N`, 7 days by default) returns the runs the app recorded, newest first:

- `kind`: `awake` or `agent`; `title`: the mode or the agent's name.
- `start`, `end` and `ended`: `Went quiet` (the agent finished or is waiting for input; `end` is when it last moved data), `Exited`, `Timer ended` or `Turned off`.
- `received`, `sent` and `apps` (bytes per process, including helpers an agent started, such as git, npm or python).
- `destinations`: hosts an agent connected to, sampled every 30 seconds.
- `events`: network changes and drops during the run. An agent that stalled right after one may need a restart.
- `batteryStart`, `batteryEnd` and `peakThermal`.

Sessions are recorded only while the Kilobat app runs. An empty list does not prove nothing ran.

## 4. When the agent can't reach something

Run `diagnose_connection` with the `target` host or URL (CLI: `kilobat why <url|host> --json`; it exits 1 when a layer failed). It checks the Mac's network, VPN and proxy, DNS, Kilobat's own rules, the TCP connection, TLS and the HTTP answer, and stops at the first failure. Report `verdict` and `next`, and the failing step's `detail`. Pass `app` (bundle ID or `process:<name>`) when one app is affected, to include its Kilobat rules.

If every layer passes, the problem is inside the agent's environment: a sandbox, container, its own proxy setting or a remote machine. Say so; the check ran on this Mac only. For broader slowness, switch to `mac-network-troubleshooting`.

## MCP tools

| MCP tool | CLI equivalent | Changes anything |
|---|---|---|
| `get_conditions` | `kilobat conditions --json` | No |
| `keep_awake` (`minutes`, `reason`) | `kilobat awake --for <duration>` | Holds off idle sleep; tell the user |
| `release_awake` | Ctrl-C on `kilobat awake` | Ends the hold |
| `list_sessions` (`days`) | `kilobat sessions --json --days N` | No |
| `diagnose_connection` (`target`, `app`) | `kilobat why <target> --json [--app <id>]` | No |
