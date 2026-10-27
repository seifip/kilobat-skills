# Expert commands

Use these after the main workflow points at a layer. All are read-only unless marked. Run `kilobat <command> --help` for exact usage.

| Question | Command | Notes |
|---|---|---|
| Is one website slow, or everything? | `kilobat http-check https://example.com` | DNS, connect, TLS and first-byte timings for one URL. |
| Which DNS resolver is fastest here? | `kilobat dns-benchmark` | JSON timings for the system resolver, Cloudflare (1.1.1.1), Google (8.8.8.8) and Quad9 (9.9.9.9). Compare with `dnsMilliseconds` before suggesting a change. |
| Where along the path does latency jump? | `kilobat trace <host>` | 12-hop route probe. A jump that persists to the last hop is real; a single slow hop that recovers is usually a router deprioritizing probes. |
| Is a VPN in the path? | `kilobat interfaces --json` | Physical and VPN interface counters, reported separately. |
| Which process talks to which host? | `kilobat destinations --json [--pid PID]` | Open sockets per process. |
| What else is on the local network? | `kilobat devices --json` | Neighbor cache: address, MAC, interface. |
| Did the connection drop earlier? | `kilobat outages` | Outages recorded by the app (three failed checks to start, two successes to end). |
| Has speed changed over time? | `kilobat speed-history` | Earlier speed test results. |
| Was the signal better elsewhere? | `kilobat spots`, `kilobat scan-history` | Saved Radar spots and scans. |
| Is Kilobat set up correctly? | `kilobat doctor` | Monitoring and filter readiness. |

Ask the user before running these; they change settings, touch other devices or record private data:

| Command | Effect |
|---|---|
| `kilobat dns <automatic, cloudflare or google>` | Changes system DNS with an administrator prompt. |
| `kilobat dns flush` | Flushes the DNS cache with an administrator prompt. Harmless, but still needs approval. |
| `kilobat lan-test <server>` | iperf3 test to another local machine running an iperf3 server. |
| `kilobat discover <subnet>` | Scans hosts with nmap. Only on the user's own network. |
| `kilobat capture <interface> <file.pcap>` | Records packet headers for up to 15 seconds with an administrator prompt. Private data. |
| `kilobat wake <MAC>` | Sends a Wake-on-LAN packet. |
| `kilobat connection-rule …`, `blocklist …`, `domain-exception …`, `scope …` | Saved filtering policy. |
