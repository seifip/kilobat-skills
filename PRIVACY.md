# Privacy and network behavior

This plugin is provided by Philipp Seyfi, the author of Kilobat, through the `seifip` GitHub account. Contact: ping@kilobat.app.

The MCP server runs locally over stdio. There is no Kilobat cloud account, API key, hosted relay, or telemetry endpoint in this plugin. It does not read shell credential stores, saved Wi-Fi passwords, SSH keys, or `.env` files.

Requested MCP results can contain app/process names and paths, bundle IDs, Wi-Fi names, IP addresses, DNS resolvers, connection destinations, usage totals, local rules, agent session identifiers and project paths, and power/network status. The MCP client receives these results; the AI provider's data-handling policies apply. Kilobat's app stores network and session history locally. The app retains history according to its local retention and cleanup settings; the user can clear it. The plugin introduces no separate history store.

## Network endpoints

- The local router and system-configured DNS resolvers: reachability, DNS, gateway latency and local-device diagnostics.
- `1.1.1.1` (Cloudflare): internet latency and optional direct DNS comparisons.
- `8.8.8.8` (Google) and `9.9.9.9` (Quad9): optional DNS benchmarks. Benchmark queries include test domains, not full browsing history.
- `http://captive.apple.com/hotspot-detect.html`: captive-portal detection.
- `https://speed.cloudflare.com/__down` and `/__up`: explicitly requested, data-budgeted speed tests. Default maximum is 80 MB; on hotspots ask first and use 10 MB or less.
- `https://www.cloudflare.com/cdn-cgi/trace`: public-IP lookup, only when requested. The service receives the connection's public IP.
- A host or URL supplied for connection diagnosis, HTTP timing, route tracing or an approved LAN test: requests go to that host. Avoid URLs containing credentials or private query values; diagnostics redact query values in displayed output.
- `https://kilobat.app/guides/index.json` and linked guides: documentation lookup by the AI client when needed.

Read-only diagnosis does not change settings. DNS changes use the macOS administrator prompt. App rules, profiles, network-cost overrides, budgets, app approvals, and related controls require explicit confirmation. Enforcement requires the separately installed Kilobat app, Pro, and its approved filter. Local Network, Location, and system-extension permissions are optional and requested only by the feature that needs them.

The plugin is a general-purpose network utility and is not directed to children. Support: ping@kilobat.app. Published policy: https://kilobat.app/privacy.html.

## Applicable third-party policies

The connected AI client processes requested tool results under its own policy: [Anthropic](https://www.anthropic.com/legal/privacy), [OpenAI](https://openai.com/policies/privacy-policy/) or [xAI](https://x.ai/legal/privacy-policy), according to the client you choose. Network probes are subject to the contacted service's policy: [Apple](https://www.apple.com/legal/privacy/), [Cloudflare](https://www.cloudflare.com/privacypolicy/), [Google](https://policies.google.com/privacy) and [Quad9](https://quad9.net/privacy/policy/). A user-supplied diagnostic target applies its own policy. The plugin sends no conversation, credentials or stored traffic history to these probe endpoints.
