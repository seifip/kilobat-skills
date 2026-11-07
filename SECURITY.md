# Security

Report security issues privately to [ping@kilobat.app](mailto:ping@kilobat.app), with the plugin version and a minimal redacted reproduction. Do not put credentials, private network history or Wi-Fi passwords in public issues.

The plugin contains no executable code: skills, manifests and an MCP entry that runs `kilobat mcp`. The `kilobat` CLI comes from Homebrew (`brew install seifip/kilobat/kilobat`), built from the public, GPL-3.0 source at github.com/seifip/kilobat-cli and signed with Developer ID. The plugin registers no login item or lifecycle hook and does not activate a network filter. MCP input schemas and tool annotations distinguish diagnostic reads, network probes, sleep assertions and settings changes. Settings changes require the person's confirmation. Network destinations and data categories are documented in [PRIVACY.md](PRIVACY.md).

Validate it with `claude plugin validate .` before publishing.
