---
name: mac-dns-troubleshooting
description: Diagnose and fix DNS on a Mac with Kilobat. Measures lookup time, benchmarks the system resolver against Cloudflare, Google and Quad9, switches or flushes DNS with the user's approval, and builds an encrypted DNS (DNS-over-HTTPS) profile. Use when pages are slow to start, sites fail with "server not found", a hotel or café login page won't appear, or the user asks which DNS is fastest.
---

# Mac DNS troubleshooting with Kilobat

DNS turns names into addresses before anything loads, so a slow or broken resolver makes every site feel slow to start even when the connection is fast. This skill measures DNS, compares resolvers and changes DNS only with the user's approval.

## Setup

Use the Kilobat MCP tools if connected, otherwise the `kilobat` CLI (`brew install seifip/kilobat/kilobat`, macOS 14+; ask before installing). Measuring is safe. Changing or flushing DNS shows a macOS administrator prompt and needs a clear yes first. The `kilobat` skill lists the rules.

## 1. Rule out a login page first

Run `kilobat diagnose --json`. If `measurements.captivePortal` is `true`, the network wants a login (hotel, café, plane, airport). Ask the user to open any http site in a browser to bring up the page. A custom DNS server can stop the page from appearing; if it never shows, offer `kilobat dns automatic` (with approval), log in, then switch back.

## 2. Measure the current resolver

From the same `diagnose` output:

- `measurements.dnsMilliseconds`: one lookup of example.com through the system resolver. Above 150 ms Kilobat rates DNS as degraded. Near 0 means a cached answer; run diagnose again or rely on the benchmark.
- `measurements.dnsServers`: the resolvers in use. Write them down before any change so they can be restored.
- `measurements.networkService`: the macOS service (for example `Wi-Fi`) that a DNS change applies to.

`kilobat dns` with no argument prints the current resolvers on their own.

## 3. Compare resolvers

Run `kilobat dns-benchmark`. It prints JSON (no `--json` flag): one entry per resolver with `resolver` (`System`, `1.1.1.1` for Cloudflare, `8.8.8.8` for Google, `9.9.9.9` for Quad9), `milliseconds` (null if none answered) and `replies`, out of three test domains.

- Run it twice. Treat a resolver as faster only if it wins both runs by a clear margin, roughly 20 ms or more. Single-digit differences are noise.
- Fewer `replies` than the others means lost queries, which hurts more than a few milliseconds.
- If the system resolver is the router (a private address such as 192.168.x.x), a slow result may be the router itself.

## 4. Separate DNS from the site

For "one site is slow", run `kilobat http-check https://example.com` with the site's URL. It prints cumulative timings: `DNS`, `Connect`, `TLS`, `First byte`, `Total`. A large `DNS` value points here; a large gap between `TLS` and `First byte` is the server, not DNS. Compare against a second site before blaming DNS.

## 5. Change DNS, with approval

Name the change, the reason with numbers, and how to undo it, then wait for a yes.

- `kilobat dns cloudflare` or `kilobat dns google` sets that provider on the active network service. `kilobat dns automatic` returns to the network's own DNS.
- `kilobat dns flush` clears the macOS DNS cache. It helps after a site moved or a stale "server not found"; it does not make DNS faster.
- Quad9 and other providers cannot be set by the CLI. The user sets them in System Settings › Network › the service › Details › DNS.
- Encrypted DNS: `kilobat dns-profile https://dns.example/dns-query > Kilobat-DNS.mobileconfig` writes a DNS-over-HTTPS configuration profile for the given HTTPS endpoint, for example `https://cloudflare-dns.com/dns-query` or `https://dns.quad9.net/dns-query`. The user reviews and installs it in System Settings; removing the profile restores the previous behavior.

Things to warn about before changing:

- Corporate networks and VPNs often need their own DNS for internal names. `kilobat interfaces --json` shows whether a VPN is up; if one is, do not change DNS without the user checking with their admin.
- A custom resolver or encrypted DNS profile can hide captive portal pages on hotel and café Wi-Fi.
- Parental controls or ad blocking on the router stop working when the Mac bypasses the router's DNS.

## 6. Verify

Run `kilobat diagnose --json` and `kilobat dns-benchmark` again and report before and after, including `dnsServers`. If nothing improved, offer to restore the original setting (`kilobat dns automatic`, or the servers recorded in step 2) and look elsewhere with `mac-network-troubleshooting`.
