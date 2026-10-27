---
name: mac-wifi-troubleshooting
description: Fix weak, slow or dropping Wi-Fi on a Mac with Kilobat. Measures signal and noise, finds crowded channels, compares spots to find where the signal is best or where to move the router, and reads past drops and access-point roams. Use for dead zones, Wi-Fi that disconnects, mesh or extender problems and 2.4 vs 5 vs 6 GHz questions.
---

# Mac Wi-Fi troubleshooting with Kilobat

This skill answers one question: is the Wi-Fi link itself the problem, and what would improve it? It covers signal strength, interference, channel crowding, placement and drops. If the Wi-Fi turns out fine and the internet is still slow, switch to `mac-network-troubleshooting`.

## Setup

Use the Kilobat MCP tools if connected, otherwise the `kilobat` CLI (`brew install seifip/kilobat/kilobat`, macOS 14+; ask before installing). Read commands below are safe to run. Ask before anything that changes settings; the `kilobat` skill lists the rules.

Wi-Fi scans and network names need Location permission for the Kilobat app. An empty scan or a missing `ssid` means permission is missing, not that there are no networks; point the user to System Settings › Privacy & Security › Location Services.

## 1. Read the link

Run `kilobat diagnose --json` and read `measurements.radio`:

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

Suggest a quieter channel by name, for example "channel 36 has no strong neighbors, 44 has four". The change happens in the router's admin page; Kilobat cannot change it.

## 3. Find a better spot

When the signal is weak on a laptop, compare places instead of guessing.

- Interactive: the user runs `kilobat radar --watch` in Terminal, or opens the Signal guide in the Kilobat app, and walks around. The score updates four times a second and says when an earlier spot was stronger.
- Agent-driven: ask the user to move to each spot and say when they are there, then run `kilobat radar --json` once per spot. It returns `score`, `signalNoiseRatio` and `radio`. A difference of 10 points or more is meaningful; less is noise.
- `kilobat spots` lists spots the user saved in the app, with their radio readings.

Report a ranked list of spots. If every spot in the room is poor, the router placement is the problem: suggest raising it, moving it out of cabinets and away from microwaves and thick walls, or adding a wired mesh node.

## 4. Explain drops

For "it disconnects sometimes", one sample is not enough.

1. Run `kilobat outages`. Each event has `start`, `end`, `network`, `kind` and `detail`. `Unreachable` means both ping targets and an HTTPS check failed. `Wi-Fi roam` means the Mac moved to another access point. `Network changed` means it joined a different network.
2. Many `Wi-Fi roam` events in a short time on a mesh mean the Mac is bouncing between nodes of similar strength. Moving a node or lowering its power often helps more than a stronger signal.
3. `Unreachable` with a good signal at that time points past Wi-Fi, to the router or internet; hand over to `mac-network-troubleshooting`.
4. Run `kilobat diagnose --json` three times about 20 seconds apart to see whether `rssi` or router loss swings.
5. `kilobat scan-history` shows earlier scans, to see whether neighbors changed.

Outages are recorded only while the Kilobat app runs.

## 5. Recommend

Lead with the measurement, then at most three fixes, most likely first. For example: "Signal is −74 dBm with 18 dB SNR on 2.4 GHz channel 6, shared with five strong networks. 1) Move closer or use 5 GHz, 2) switch the router's 2.4 GHz to channel 1, which has one neighbor, 3) add a mesh node in the hallway." After a change, take the same measurement again and report before and after.

Things the user can try on the Mac itself: forget and rejoin the network, turn Wi-Fi off and on, and check for a VPN (`kilobat interfaces --json` lists VPN interfaces). Ethernet is the fastest way to tell a Wi-Fi problem from everything else.
