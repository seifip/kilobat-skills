---
name: kilobat-setup
description: Set up Kilobat on a Mac for an agent, install or update its CLI with Homebrew, check the MCP connection, explain permissions and local data handling, and run the first network check. Use after installing the Kilobat plugin or skills, or when Kilobat's tools or the kilobat command are missing.
---

# Set up Kilobat

1. Check the execution environment. Kilobat measures the Mac it runs on and supports macOS 14 or later on Intel and Apple silicon. A cloud or Linux session cannot measure the user's Mac. Say so if the current session is remote or sandboxed.
2. Check for the CLI with `command -v kilobat` and `kilobat --version`. If it is missing, ask the user, then install it with `brew install seifip/kilobat/kilobat`. If it is old, `brew upgrade kilobat`. Don't install Homebrew itself without asking. The menu-bar app is optional: `brew install --cask seifip/kilobat/kilobat`.
3. The plugin starts the MCP server with `kilobat mcp`. If its tools are missing after the install, ask the user to restart or reconnect the client. Don't add a duplicate global MCP entry or replace another install.
4. When the tools connect, call `get_conditions` (or run `kilobat conditions --json`). Explain whether the connection is metered, whether Low Data Mode is on, and whether the Mac is on battery. Missing values mean unknown.
5. Explain that Kilobat keeps network history on the Mac, and that results requested through the agent are shared with the AI client under its policies. Probes contact the router, the configured DNS resolvers, Apple or Cloudflare, or a host the user asks to check. Don't look up the public IP or share history unless the task needs it.
6. For a first check, call `diagnose_network` (or `kilobat diagnose --json`). During setup don't run a speed test, change DNS, change an app rule, turn on filtering or keep the Mac awake. Show the measured findings and anything that couldn't be measured.
7. Optional app features: per-app rules need the Kilobat app, Kilobat Pro and the network filter, which the person approves in macOS. History needs the app running. Nearby Wi-Fi names can need Location permission. The CLI diagnoses a network without the app, so missing history or filtering is not a failed install.
8. Load `../kilobat/SKILL.md` for the task router and the right focused skill. Report whether the tools connected and which checks ran.
