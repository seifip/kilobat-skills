---
name: mac-agent-runs
description: Keep a Mac awake for long AI agent runs, builds and downloads with Kilobat, with the lid closed when the user allows it, check whether the connection suits a big transfer, and find out when a run finished, is waiting for the user, stalled or hit a network change. Covers kilobat awake, keep_awake, get_conditions, kilobat sessions, kilobat why and connecting Claude Code, Codex or Gemini CLI hooks, or Cline's plugin, to kilobat agent-event. Use for "don't let my Mac sleep during this", "is this a good time to download", "did the agent finish", "tell me when Claude needs me" or "why can't the agent reach this API".
---

# Long agent runs on a Mac with Kilobat

A Mac that idle-sleeps halfway through a build, a model download or an agent's run loses the work. Kilobat keeps it awake on purpose, ends the hold when the work goes quiet, and records each run with the data it used and the network changes it went through. This skill plans a long run, keeps it alive and reads what happened.

## Setup

Use the Kilobat MCP tools if connected, otherwise the `kilobat` CLI (`brew install seifip/kilobat/kilobat`, macOS 14+; ask before installing). Commands here need CLI 0.4 or later (`kilobat --version`); if one is unknown, suggest `brew upgrade kilobat`. Reading is safe. Keeping the Mac awake changes no settings and is low risk; say what you are doing and for how long, and ask first on battery. The `kilobat` skill lists the rules.

Kilobat stops idle sleep. Closing the lid still sleeps the Mac unless the user turned on "Allow running with the lid closed" in Kilobat Settings (section 2b). Never suggest `pmset` or other tricks to override sleep.

## 1. Check conditions before a big transfer or a long run

Call `get_conditions` (CLI: `kilobat conditions --json`, about 2 seconds). Read:

- `metered` (hotspot or a network marked expensive) and `lowDataMode`: ask before downloading or uploading more than a few hundred MB, and prefer smaller artifacts or a later time.
- `vpn` and `vpnInterface`: some hosts are blocked or slower through a VPN.
- `internetLatencyMs` and `internetLossPercent`: high loss means retries and timeouts; `null` means the ping went unanswered, not that the internet is down.
- `power`: `onAC`, `batteryPercent`, `thermal` (`nominal` to `critical`) and `lowPowerMode`. On battery below about 25%, suggest plugging in before a long run.
- `keptAwake` and `awakeReasons`: whether Kilobat (the app, the CLI or this MCP server) already keeps the Mac awake.
- `lidClosedAllowed` and `lidHold` (`until`, `reason`, `batteryFloor`, `onlyOnAC`): whether the user allowed running with the lid closed, and whether a hold is on now.
- `advice`: one line that sums it up. Quote it.

## 2. Keep the Mac awake

Pick the narrowest form:

- One command: `kilobat awake -- <command> [args]` keeps the Mac awake while it runs and exits with its status. A summary with the bytes its processes moved goes to stderr (`--json` for JSON). Prefer this for builds, test suites and downloads started from a shell.
- A set time: MCP `keep_awake` with `minutes` (1–240) and a short `reason`; call it again to extend, `release_awake` when done. The hold also ends when the MCP server stops. CLI: `kilobat awake --for 90m`, which runs in the foreground.
- Until transfers finish: `kilobat awake --until-quiet` ends once the Mac stays below 16 KB/s for 3 minutes (`--quiet 1…60` minutes). `--app <id>` watches one app, using the `id` from `kilobat status --json`.
- `--display` keeps the screen on too; only when the user wants to watch.
- For the user's own reading or presenting, not an agent's work: `kilobat awake --while-frontmost <app> [--display] [--for 2h]` keeps the Mac awake only while that app (its name, such as Keynote, or bundle ID) is in front, lets it sleep when the user switches away, and ends when the app quits or at 20% battery. It follows macOS's app-switch events and never fakes input. Suggest it for the user's Terminal; the app's teacup has it as While an App Is in Front. There is no MCP tool for it, and it never ends an agent run or another awake session.

Plain `kilobat awake` runs until Ctrl-C, so suggest it for the user's own Terminal. From an agent, prefer `awake -- <command>`, `--for` or `keep_awake`, which end on their own.

The Kilobat app can also keep the Mac awake while it sees an AI agent (Claude Code, Codex, Gemini CLI, Aider and others) at work, and notify when a run finishes, waits for the user or loses the network. That is the user's setting in the app. With the agent's hooks connected (section 5), it knows which; without them it goes by traffic and CPU, and a quiet agent may be done or waiting.

## 2b. With the lid closed

Only when the user asks to close the lid or carry the Mac during a run. It needs the Kilobat app 1.3 or later with "Allow running with the lid closed" on in Settings › Awake & agents and its helper approved in System Settings › General › Login Items; only the user can do both. `get_conditions` says so in `lidClosedAllowed`.

- Risks to state before asking: a closed Mac in a bag can overheat, and the battery drains with no screen to warn anyone. Suggest plugging in.
- Get the user's explicit confirmation every time, with how long. Then MCP `keep_awake` with `lid_closed: true`, or `kilobat awake --lid-closed --for 2h` (or `-- <command>`, which ends the hold with the command). 8 hours at most.
- Kilobat's helper lets the Mac sleep anyway at the user's battery floor (20% by default), when it runs hot, when unplugged if the user chose power only, at the time limit, and a minute after the app quits. `lidFloor` in the result is the floor (100 means power only). Don't promise the run survives any of those.
- If it fails, quote the error: the setting is off, the helper is waiting for approval, or the battery or heat already rule it out. Never work around it.
- With the setting on, the app also holds the lid for agent runs it sees working on its own, and for an agent waiting for the user only with "Keep awake while an agent waits" on, 30 minutes at most.

## 3. Read what happened

`list_sessions` (CLI: `kilobat sessions --json --days N`, 7 days by default) returns the runs the app recorded, newest first:

- `kind`: `awake` or `agent`; `title`: the mode or the agent's name.
- `start`, `end` and `ended`: `Finished` (the agent's hook said so, after any background tasks it started), `Failed: <error>`, `Went quiet` (no hooks; its traffic and CPU stopped, so it finished or is waiting for input; `end` is when it last worked), `Exited`, `Timer ended` or `Turned off`. Only `Finished` means done.
- With hooks: `session`, `cwd`, `project` (the folder's name) and `state`: `working`, `waitingForYou`, `stalled` (no hook event, traffic or CPU work for the quiet minutes while the agent still runs), `networkFailed`, `done` or `failed`; for an ended run, the last state seen. `note` adds a rate limit ("Rate-limited until 3pm") or background tasks still running.
- `lastEvent` (last hook), `lastActive` (last traffic) and `lastCPU` (last CPU work) are kept apart; quiet on one of them alone is not done.
- `executable`, `origin` (Homebrew, npm (node 22.3.0), Claude app, ~/.local…), `version` and `otherInstalls`: which install of the agent ran, read from its path. Another install of the same agent is a common cause of "my settings don't apply".
- `sharedTraffic`: runs of one agent that share a process can't be told apart in traffic, so these bytes may include another run.
- `received`, `sent` and `apps` (bytes per process, including helpers an agent started, such as git, npm or python).
- `destinations`: hosts an agent connected to, sampled every 30 seconds.
- `events`: network changes and drops during the run. An agent that stalled right after one may need a restart.
- `batteryStart`, `batteryEnd` and `peakThermal`.

Sessions are recorded only while the Kilobat app runs. An empty list does not prove nothing ran.

## 4. When the agent can't reach something

Run `diagnose_connection` with the `target` host or URL (CLI: `kilobat why <url|host> --json`; it exits 1 when a layer failed). It checks the Mac's network, VPN and proxy, DNS, Kilobat's own rules, the TCP connection, TLS and the HTTP answer, and stops at the first failure. Report `verdict` and `next`, and the failing step's `detail`. Pass `app` (bundle ID or `process:<name>`) when one app is affected, to include its Kilobat rules, environment proxy and open connections, and `proxy` when it uses a proxy address from its own settings. Give the exact failing URL; query values are redacted in the output. A `Transfer` warning means a download already under way went quiet, which isn't the same as finished: say it went quiet, and suggest checking it in the app.

If every layer passes, the problem is inside the agent's environment: a sandbox, container, its own proxy setting or a remote machine. Say so; the check ran on this Mac only. When a hosted service makes the request from its own servers (Warp's custom inference endpoint, say), pass `context` provider: a private target such as `localhost` or a LAN or Tailscale address works from the Mac but not from there, and `next` gives the options. For broader slowness, switch to `mac-network-troubleshooting`.

## 5. Connect agents

Hooks tell Kilobat what traffic can only guess: which project a run belongs to, when it waits for the user and when it is done. Each hook runs `kilobat agent-event`, which prints nothing, returns at once and always exits 0, so it can't break the agent. If the app isn't running, events wait in a small file it reads on launch. It needs a CLI that lists `agent-event` in `kilobat --help`. Show the user the snippet and ask before editing their settings; merge with hooks they already have rather than replacing them. `kilobat agent-event start --agent claude --verbose` explains what it did, for a test by hand.

**Claude Code**, in `~/.claude/settings.json`. Claude Code passes each hook a JSON object on stdin with `session_id`, `cwd` and `hook_event_name`, which `--stdin-json` reads: SessionStart starts the run, UserPromptSubmit marks it working, Notification (a permission or input prompt) waiting, Stop done (or still working while background tasks run), StopFailure failed, or waiting when it is a rate limit, and SessionEnd done.

```json
{
  "hooks": {
    "SessionStart": [{ "hooks": [{ "type": "command", "command": "kilobat agent-event --agent claude --stdin-json" }] }],
    "UserPromptSubmit": [{ "hooks": [{ "type": "command", "command": "kilobat agent-event --agent claude --stdin-json" }] }],
    "Notification": [{ "hooks": [{ "type": "command", "command": "kilobat agent-event --agent claude --stdin-json" }] }],
    "Stop": [{ "hooks": [{ "type": "command", "command": "kilobat agent-event --agent claude --stdin-json" }] }],
    "StopFailure": [{ "hooks": [{ "type": "command", "command": "kilobat agent-event --agent claude --stdin-json" }] }],
    "SessionEnd": [{ "hooks": [{ "type": "command", "command": "kilobat agent-event --agent claude --stdin-json" }] }]
  }
}
```

**Codex**, simplest: `notify` in `~/.codex/config.toml`. Codex runs it after each turn with a JSON argument (`thread-id`, `cwd`), which `--json-arg` reads. It only says done, and Codex allows one `notify` program, so check for an existing one first.

```toml
notify = ["kilobat", "agent-event", "done", "--agent", "codex", "--json-arg"]
```

For waiting too, use Codex hooks in `~/.codex/hooks.json` instead; they get the same stdin JSON as Claude Code's, and PermissionRequest means waiting. Codex asks the user to trust new hooks with `/hooks` before they run.

```json
{
  "hooks": {
    "SessionStart": [{ "hooks": [{ "type": "command", "command": "kilobat agent-event --agent codex --stdin-json" }] }],
    "UserPromptSubmit": [{ "hooks": [{ "type": "command", "command": "kilobat agent-event --agent codex --stdin-json" }] }],
    "PermissionRequest": [{ "hooks": [{ "type": "command", "command": "kilobat agent-event --agent codex --stdin-json" }] }],
    "Stop": [{ "hooks": [{ "type": "command", "command": "kilobat agent-event --agent codex --stdin-json" }] }]
  }
}
```

**Gemini CLI**, in `~/.gemini/settings.json`. Its hooks also pass JSON on stdin; BeforeAgent marks a turn working, AfterAgent done, and a ToolPermission notification waiting. Gemini CLI runs as `node`, so Kilobat can't tie its traffic or CPU to the run; the state still comes through.

```json
{
  "hooks": {
    "SessionStart": [{ "hooks": [{ "type": "command", "command": "kilobat agent-event --agent gemini --stdin-json" }] }],
    "BeforeAgent": [{ "hooks": [{ "type": "command", "command": "kilobat agent-event --agent gemini --stdin-json" }] }],
    "Notification": [{ "hooks": [{ "type": "command", "command": "kilobat agent-event --agent gemini --stdin-json" }] }],
    "AfterAgent": [{ "hooks": [{ "type": "command", "command": "kilobat agent-event --agent gemini --stdin-json" }] }]
  }
}
```

**Cline** (CLI, SDK and Kanban; plugins don't run in its IDE extensions): Kilobat's plugin, one file that `cline plugin install` adds to Cline's plugin store without touching other settings. It loads in sessions started afterwards; `cline plugin uninstall kilobat` removes it.

```sh
cline plugin install https://kilobat.app/agents/cline/kilobat.ts
```

It maps Cline's plugin hooks to `agent-event` with the session id and folder: a run starting is working; completing is done once the subagents and teammates it started finish; failing while Cline stays open is failed, with a fixed message such as "Lost the connection to the model provider", or waiting when it is a rate limit that outlasted Cline's retries; cancelling is waiting ("Cancelled"); `ask_question` is waiting until answered; closing the session ends an open run as done. Plugins can't see Cline's approval prompts, so traffic and CPU detection covers that time. The plugin finds `kilobat` on PATH, in `/opt/homebrew/bin` or `/usr/local/bin`, or at `KILOBAT_CLI`.

Other agents: call `kilobat agent-event start|working|waiting|done|failed --agent <name> --session <id> --cwd <folder>` from whatever hook they offer. Without hooks, traffic and CPU detection still cover them. Kilobat reads ids, folders and short status text only, never prompts or replies.

## MCP tools

| MCP tool | CLI equivalent | Changes anything |
|---|---|---|
| `get_conditions` | `kilobat conditions --json` | No |
| `keep_awake` (`minutes`, `reason`, `lid_closed`) | `kilobat awake --for <duration> [--lid-closed]` | Holds off idle sleep; tell the user. `lid_closed` needs their confirmation |
| `release_awake` | Ctrl-C on `kilobat awake` | Ends the hold |
| `list_sessions` (`days`) | `kilobat sessions --json --days N` | No |
| `diagnose_connection` (`target`, `app`, `proxy`, `context`, `failed_at`) | `kilobat why <target> --json [--app <id>] [--proxy <url>] [--context container\|vm\|remote\|provider] [--failed-at <time>]` | No |
