# Kilobat agent skills

Agent Skills that teach AI agents to troubleshoot a Mac's network with [Kilobat](https://kilobat.app). They interpret Kilobat's measurements, say what is uncertain, and ask before changing settings.

Install all of them:

```sh
npx skills add seifip/kilobat-skills
```

| Skill | Use it for |
|---|---|
| [`kilobat`](skills/kilobat/SKILL.md) | Start here. Setup, safety rules and which skill fits the question. |
| [`mac-network-troubleshooting`](skills/mac-network-troubleshooting/SKILL.md) | Slow or unreliable internet with an unclear cause: finds the failing layer. |
| [`mac-wifi-troubleshooting`](skills/mac-wifi-troubleshooting/SKILL.md) | Weak or dropping Wi-Fi, crowded channels, the best spot for the Mac or router, mesh roaming. |
| [`mac-hotspot-data-saver`](skills/mac-hotspot-data-saver/SKILL.md) | Phone hotspots, roaming and data caps: what uses data, Travel profile, budgets. |
| [`mac-dns-troubleshooting`](skills/mac-dns-troubleshooting/SKILL.md) | Slow lookups, resolver benchmarks, switching or flushing DNS, encrypted DNS, captive portals. |
| [`video-call-network-check`](skills/video-call-network-check/SKILL.md) | Zoom, Meet, Teams and FaceTime quality: jitter, loss, bufferbloat, background uploads. |
| [`mac-bandwidth-monitor`](skills/mac-bandwidth-monitor/SKILL.md) | Which apps use the connection, usage history and where each process connects. |

The skills use Kilobat's CLI or its MCP server. Installing them does not install the CLI. To install the CLI on macOS 14 or newer, using Xcode 16 or newer:

```sh
brew install seifip/kilobat/kilobat
```

Or connect the MCP server: `claude mcp add kilobat -- kilobat mcp`.

See the [documentation](https://kilobat.app/docs.html#network-troubleshooting-skill) or the [skills on skills.sh](https://skills.sh/seifip/kilobat-skills).