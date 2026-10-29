---
name: mac-wifi-troubleshooting
description: Fix weak, slow, stuttering or dropping Wi-Fi on a Mac with Kilobat. Measures signal and noise, finds crowded channels, compares spots to find where the signal is best or where to move the router, finds periodic latency spikes, checks AirDrop/AWDL and Bluetooth sharing the radio, and reads past drops and access-point roams. Use for dead zones, Wi-Fi that disconnects or lags in bursts, mesh or extender problems and 2.4 vs 5 vs 6 GHz questions.
---

# Mac Wi-Fi troubleshooting with Kilobat

This skill answers one question: is the Wi-Fi link itself the problem, and what would improve it? It covers signal strength, interference, channel crowding, placement and drops. If the Wi-Fi turns out fine and the internet is still slow, switch to `mac-network-troubleshooting`.

## Setup

Use the Kilobat MCP tools if connected, otherwise the `kilobat` CLI (`brew install seifip/kilobat/kilobat`, macOS 14+; ask before installing). Commands here need CLI 0.2 or later (`kilobat --version`), and `stutter`, `airdrop` and `bluetooth` a newer one; if one is unknown, suggest `brew upgrade kilobat`. Read commands below are safe to run. Ask before anything that changes settings; the `kilobat` skill lists the rules.

macOS shows Wi-Fi names only to apps with Location permission. The CLI gets them from the running Kilobat app; if the app is closed or lacks permission, `ssid` is null and scans list "Hidden network". Signal, channel and band still work. Point the user to `kilobat open`, or System Settings › Privacy & Security › Location Services for Kilobat.

## 1. Read the link

For the radio alone, `kilobat wifi status --json` is quickest: `ssid`, `rssi`, `noise`, `signalNoiseRatio`, `channel`, `band`, `transmitMbps` and `score`. `kilobat diagnose --json` adds router latency and loss; its `measurements.radio` has the same fields:

| Field | Good | Degraded | Poor |
|---|---|---|---|
| `rssi` | −65 dBm or stronger | −65 to −75 | weaker than −75 |
| `signalNoiseRatio` (`rssi − noise`) | 25 dB or more | 15–25 | under 15 |

The weaker of the two decides. `signalScore` is `(rssi + 90) × 2`, clamped to 0–100. Also note:

- `band` (2.4, 5 or 6 GHz) and `channel`. 2.4 GHz reaches further but is slower and more crowded.
- `transmitMbps`, the negotiated link rate. Low with a good RSSI points to interference, an old router or a narrow channel.
- `routerLatencyMs` above 20 ms or router `lossPercent` above 0 with a good signal points to congestion or a busy router rather than distance.

If the Mac is on Ethernet (`radio` fields are null), this skill does not apply.

## 2. Check for crowding

Run `kilobat scan --json`. Each entry has `ssid`, `rssi`, `noise`, `channel`, `band` and `widthMHz`.

- Count networks on the user's channel with `rssi` stronger than about −75 dBm. Three or more strong neighbors on the same channel means contention.
- On 2.4 GHz only channels 1, 6 and 11 don't overlap. A network on 3 or 9 interferes with two of them.
- 80 or 160 MHz channels are faster but collide with more neighbors. In a crowded building 40 MHz on 5 GHz is often more stable.
- The same `ssid` appearing on several channels means a mesh or extenders. Check that the Mac is on the strongest one.

Run `kilobat channels --json` for Kilobat's estimate: `band`, `current`, `recommended` and a `congestion` score per candidate channel (lower is better). Suggest a quieter channel by name, for example "channel 36 scores 56, your channel 44 scores 73". The change happens in the router's admin page; Kilobat cannot change it.

## 3. Find a better spot

When the signal is weak on a laptop, compare places instead of guessing.

- Interactive: the user runs `kilobat radar --watch` in Terminal, or opens the Signal guide in the Kilobat app, and walks around. The score updates four times a second and says when an earlier spot was stronger.
- Agent-driven: ask the user to move to each spot and say when they are there, then run `kilobat radar --json` once per spot. It returns `score`, `signalNoiseRatio` and `radio`. A difference of 10 points or more is meaningful; less is noise.
- `kilobat spots save "<room>"` stores the current reading under a name (with approval); `kilobat spots` lists saved spots with their radio readings, so the user can compare rooms later or in the app.

Report a ranked list of spots. If every spot in the room is poor, the router placement is the problem: suggest raising it, moving it out of cabinets and away from microwaves and thick walls, or adding a wired mesh node.

## 4. Explain drops

For "it disconnects sometimes", one sample is not enough.

1. Run `kilobat outages`. Each event has `start`, `end`, `network`, `kind` and `detail`. `Unreachable` means both ping targets and an HTTPS check failed. `Wi-Fi roam` means the Mac moved to another access point. `Network changed` means it joined a different network.
2. Many `Wi-Fi roam` events in a short time on a mesh mean the Mac is bouncing between nodes of similar strength. Moving a node or lowering its power often helps more than a stronger signal.
3. `Unreachable` with a good signal at that time points past Wi-Fi, to the router or internet; hand over to `mac-network-troubleshooting`.
4. `kilobat health-history --hours 24 --json` shows the app's periodic checks (`rssi`, `routerMs`, `internetMs`, `jitterMs`, `lossPercent`, `dnsMs`, `health`) to see when quality dropped. For a live view, run `kilobat diagnose --json` three times about 20 seconds apart.
5. `kilobat scan-history` shows earlier scans, to see whether neighbors changed.

Outages are recorded only while the Kilobat app runs.

## 5. Bursts of lag: spikes, AirDrop and Bluetooth

When the signal is fine but games, calls or remote desktops lag in bursts, look for short spikes and for the other radios sharing the Wi-Fi chip and band.

- `kilobat stutter --json` (MCP: `run_stutter_test`) pings the router every 50 ms for 20 seconds. Spikes repeating at a regular interval (`pattern: periodic`, `periodSeconds`) often come from AWDL, the link AirDrop, Universal Control and Sidecar use, which hops channels on a schedule; irregular spikes point to interference or a busy network. If `pingRateLimited` is true, the router answers only some fast pings, and the loss is not real.
- `kilobat airdrop --json` (MCP: `get_nearby_radios`) shows whether the AirDrop link is up and active and which features use it, and the AirDrop visibility. Only System Settings can change visibility; `kilobat airdrop off` opens the page (ask first). Visibility doesn't stop AWDL from scanning; the low-latency pause does.
- To test AWDL as the cause, the user runs `kilobat low-latency on --for 15m` in their own Terminal (administrator prompt; 4 hours at most with `--for`, or `--until-quit <app>`), then `kilobat stutter` again. Fewer spikes with the pause means AWDL contributes; the same spikes mean it doesn't. `kilobat low-latency off` restores it.
- `kilobat bluetooth --json` lists Bluetooth devices. Bluetooth uses 2.4 GHz; when `wifiBand` is 2.4 GHz and headphones, a keyboard or a mouse are connected (`sharesBand`), suggest a 5 GHz network. The Kilobat app's Wi-Fi menu shows the same tag and a shortcut to a known 5 GHz network.

## 6. Recommend

Lead with the measurement, then at most three fixes, most likely first. For example: "Signal is −74 dBm with 18 dB SNR on 2.4 GHz channel 6, shared with five strong networks. 1) Move closer or use 5 GHz, 2) switch the router's 2.4 GHz to channel 1, which has one neighbor, 3) add a mesh node in the hallway." After a change, take the same measurement again and report before and after.

Things to try on the Mac itself, with approval: `kilobat wifi restart` (Wi-Fi off and on), rejoining with `kilobat wifi join "<name>"`, and checking for a VPN (`kilobat interfaces --json` lists VPN interfaces). `kilobat wifi known` lists saved networks in order of preference; a stale network high on the list can make the Mac join the wrong one. Restarting or rejoining drops the connection for a few seconds, which can cut off a remote session. Ethernet is the fastest way to tell a Wi-Fi problem from everything else.
