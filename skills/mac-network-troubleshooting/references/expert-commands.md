# Expert commands

Use these after the main workflow points at a layer. All are read-only unless marked. Run `kilobat <command> --help` for exact usage.

| Question | Command | Notes |
|---|---|---|
| Is one website slow, or everything? | `kilobat http-check https://example.com --json` | DNS, connect, TLS and first-byte timings for one URL, in milliseconds. |
| Which DNS resolver is fastest here? | `kilobat dns-benchmark` | JSON timings for the system resolver, Cloudflare (1.1.1.1), Google (8.8.8.8) and Quad9 (9.9.9.9). Compare with `dnsMilliseconds` before suggesting a change. |
| Where along the path does latency jump? | `kilobat trace <host>` | 12-hop route probe. A jump that persists to the last hop is real; a single slow hop that recovers is usually a router deprioritizing probes. |
| Does that hop stay slow, or did the route change? | `kilobat trace <host> --rounds 3 --json`, then `kilobat route-history --host <host> --json` | Traces with three probes a hop, saved and combined with earlier traces like mtr: per hop `replies`/`probes`, `bestMs`, `averageMs`, `worstMs`, `lastMs`, `changed`; `pathChanges` and a `verdict` on where lasting delay starts. MCP: `trace_route` with `rounds`, `get_route_history`. |
| Is a VPN in the path? | `kilobat ip --json` | `vpn` full, partial (connected, but only some destinations use it) or off, `vpnName`, `interface`. `--public` adds the public IP from Cloudflare (contacts a third party). `kilobat interfaces --json` has the counters. |
| Why do two speed tests differ? | `kilobat speed-compare --json` | The newest two saved tests by default: differences in network, time, server, connections, Wi-Fi band and signal, VPN, other traffic, idle latency. Differences, not proven causes. |
| Which process talks to which host? | `kilobat destinations --json [--pid PID]` | Open sockets per process. |
| What else is on the local network? | `kilobat devices --json` | Neighbor cache: address, MAC, interface. |
| Did the connection drop earlier? | `kilobat outages [--days N]` | Outages recorded by the app (three failed checks to start, two successes to end). |
| How did quality change over the day? | `kilobat health-history --hours 24 --json` | The app's periodic router, internet, jitter, loss and DNS checks. |
| Which channel is least crowded? | `kilobat channels --json` | Congestion estimate per candidate channel. |
| Has speed changed over time? | `kilobat speed-history` | Earlier speed test results. |
| Was the signal better elsewhere? | `kilobat spots`, `kilobat scan-history` | Saved Radar spots and scans. |
| Is Kilobat set up correctly? | `kilobat doctor` | Whether the app runs, the filter is on and Pro is active. |

Ask the user before running these; they change settings, touch other devices or record private data:

| Command | Effect |
|---|---|
| `kilobat dns <automatic, cloudflare, google or quad9>` | Changes system DNS with an administrator prompt. |
| `kilobat dns flush` | Flushes the DNS cache with an administrator prompt. Harmless, but still needs approval. |
| `kilobat lan-test <server> [--reverse] --json` | Five seconds of iperf3 to a local machine running `iperf3 -s`: `receivedMbps`, `retransmits`. Floods the LAN briefly; needs iperf3 the user installed (never install it). MCP: `run_lan_test`. |
| `kilobat discover <subnet>` | Scans hosts with nmap. Only on the user's own network. |
| `kilobat capture <interface> <file.pcap>` | Records packet headers for up to 15 seconds with an administrator prompt. Private data. |
| `kilobat wake <MAC>` | Sends a Wake-on-LAN packet. |
| `kilobat connection-rule …`, `blocklist …`, `domain-exception …`, `scope …` | Saved filtering policy. |
