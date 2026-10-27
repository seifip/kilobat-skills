# Kilobat agent skill

`mac-network-troubleshooting` helps an AI agent investigate slow internet, Wi-Fi drops, latency, DNS, bufferbloat, hotspots, and per-app data use on macOS. It interprets Kilobat's measurements, identifies uncertainty, and asks before changing settings.

Install the skill:

```sh
npx skills add seifip/kilobat-skills
```

The skill uses Kilobat's CLI or MCP server. Installing the skill does not install the CLI. To install the CLI on macOS 14 or newer, using Xcode 16 or newer:

```sh
brew install seifip/kilobat/kilobat
```

See the [skill on skills.sh](https://skills.sh/seifip/kilobat-skills/mac-network-troubleshooting), the [documentation](https://kilobat.app/docs.html#network-troubleshooting-skill), or read [the skill and its references](skills/mac-network-troubleshooting/SKILL.md).
