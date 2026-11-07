# Notes for Anthropic reviewers

Kilobat is a first-party local macOS network tool. It runs on the user's Mac and uses CoreWLAN, process and network APIs and power assertions. The plugin works in Claude Code on a Mac. It can't diagnose a user's Mac from claude.ai, Cowork's sandbox or another cloud host, so the listing is for Claude Code only.

## What the plugin contains

Readable files only: Agent Skills (`skills/`), the plugin and marketplace manifests (`.claude-plugin/`) and this documentation. There are no binaries, scripts, hooks or MCP servers. The skills run the `kilobat` command, the CLI the user installs with Homebrew (`brew install seifip/kilobat/kilobat`). Its source is public under GPL-3.0 at https://github.com/seifip/kilobat-cli, and the release binaries are signed with Developer ID (team HAR874R67J). The `kilobat-setup` skill checks for the CLI and asks the user before installing it.

## Credentials

The plugin has no API-key or token configuration, no environment credential forwarding and no credential upload. The word "pass" in tool descriptions and skills means supplying a tool or CLI argument, such as a data budget. No MCP tool exposes the standalone CLI's saved-Wi-Fi-password command, and the skills never ask for it. The plugin declares no MCP server. The `kilobat-setup` skill can connect the CLI's local MCP server with `claude mcp add` only after the user agrees.

## Data and user control

Requested diagnostic results can include Wi-Fi names, process and app identifiers, connection destinations, usage totals and project or session paths. The AI client receives these results under its own policy. Local storage, network probe destinations and optional macOS permissions are described in `PRIVACY.md`. Settings changes require confirmation. App allow requests open Kilobat's approval window for the person to confirm. The plugin installs no hooks and does not activate the app's network extension.

GPL-3.0-only covers the plugin and the CLI. The separately distributed menu-bar app has its own terms.
