---
title: "Intermittent ERR_CONNECTION_CLOSED: A Force-DNS NAT Rule Catching AdGuard's Own Upstream Queries"
date: 2026-10-10
featured: false
category: "Networking"
tags: ["AdGuard Home", "UniFi", "DNS", "NAT", "DoH"]
summary: "Random page-load failures and Apple TV 'no network' warnings traced to a UniFi rule that redirects all port 53 traffic to AdGuard. It was also redirecting AdGuard's own plain-DNS upstream queries back to itself. Switching the upstreams to DNS-over-HTTPS removed the loop."
---

# Intermittent ERR_CONNECTION_CLOSED: A Force-DNS NAT Rule Catching AdGuard's Own Upstream Queries

**Category:** Networking / DNS
**Environment:** UniFi Dream Machine, AdGuard Home (LXC on Node-C), multiple VLANs
**Date:** October 2026
**Status:** Resolved

---

## Summary

Clients on the network intermittently saw `ERR_CONNECTION_CLOSED` in browsers, and Apple TVs reported "no network". The fleet enforces DNS by redirecting all outbound port 53 traffic to AdGuard Home at `192.168.1.4` with a NAT rule on the UDM. AdGuard's own upstream resolvers were configured as plain DNS (1.1.1.1 and 8.8.8.8), so its outbound queries were caught by the same rule and sent back to itself. Switching the upstreams to DoH removed the loop.

---

## Impact

- Removed intermittent browser and Apple TV failures across the network.
- Kept network-wide DNS enforcement through AdGuard (about 2.5 million blocklist rules) without needing an exclusion rule.

---

## Environment

| Component | Detail |
|-----------|--------|
| Resolver | AdGuard Home, `192.168.1.4`, about 2.5 million blocklist rules |
| Enforcement | UDM NAT rule redirecting outbound port 53 to AdGuard |
| Upstreams before | Plain DNS: 1.1.1.1, 8.8.8.8 |
| Upstreams after | DoH: `https://1.1.1.1/dns-query`, `https://9.9.9.9/dns-query` |

---

## Symptom

- Browser errors with `ERR_CONNECTION_CLOSED` that came and went.
- Apple TVs periodically warning that there was no network connection.
- No consistent client, VLAN, or site was affected.

---

## Isolation

1. **Look for what is shared.** Intermittent failures across unrelated clients point at a shared dependency, and DNS was the obvious one.
2. **Think about the rule's blast radius.** A rule that redirects every port 53 packet leaving the network also applies to packets that AdGuard itself originates.
3. **Compare the upstream transport.** Plain DNS upstreams on port 53 were exactly the traffic the rule matches. DoH on port 443 is not.

---

## Root Cause

The Force DNS NAT rule had no exception for the resolver. AdGuard's plain-DNS upstream queries were redirected to AdGuard, so some lookups looped instead of reaching a real upstream.

---

## Resolution

There were two valid fixes: switch the upstreams to DoH, or exempt AdGuard's own address from the Force DNS rule. DoH was chosen because it matched the original design and needs no exception. The configuration documented for AdGuard was DoH, and the live configuration had drifted to plain DNS, which is why nothing had broken earlier.

Replaced the plain upstreams with DoH. Port 443 traffic is not caught by the port 53 redirect, so AdGuard reaches its upstreams directly while every client still has its plain DNS forced through AdGuard. No exclusion rule was needed.

Worth recording: DoH on port 443 bypasses the port 53 NAT rule by design, which is why it fixes this, and also why a Force-DNS rule alone is not a complete control against clients that use DoH themselves.

---

## Evidence

Captured from the terminal sessions at the time and trimmed. Credentials, tokens, and identifiers are removed.

**What the AdGuard query log showed:** the failing sites were being answered `NOERROR`, so AdGuard was not blocking them, and the upstream column showed plain DNS to 8.8.8.8 on port 53, which was not the intended design.

**Upstream settings before and after:**

```
Before (plain DNS, matched by the port 53 redirect):
  1.1.1.1
  8.8.8.8

After (DoH on port 443, not matched):
  Upstream:  https://1.1.1.1/dns-query
             https://9.9.9.9/dns-query
  Fallback:  https://1.0.0.1/dns-query
             https://149.112.112.112/dns-query
  Bootstrap: plain IPs, used only to resolve the DoH hostnames
```

**Result:** after the change the previously failing pages loaded immediately, and the Apple TV warning (a connectivity check that depends on a first DNS lookup succeeding) stopped.

---

## Verify It Yourself

These commands check the claims above. Run them on the relevant host.

```bash
# Resolve through AdGuard
dig @192.168.1.4 example.com +short

# Watch whether the resolver's own upstream queries are being redirected
sudo tcpdump -ni any 'port 53 and host 192.168.1.4'

# Upstream transport in AdGuard Home settings should be DoH:
#   https://1.1.1.1/dns-query
#   https://9.9.9.9/dns-query
```

---

## What I Would Change

- Alert on AdGuard query failure rate and latency, not only on whether the container is up.
- Finish the secondary AdGuard on Ghost Squadron with the same rewrites.
- Decide whether to block or redirect client-side DoH, since the port 53 rule does not cover it.

---

## Takeaways

- **An enforcement rule must account for the enforcer.** The resolver's own traffic needs an exception or a transport the rule does not match.
- **Intermittent plus shared means DNS first.**
- **DoH sidesteps port 53 NAT cleanly.** It is also the reason a Force-DNS rule is not a complete control.

---

## Related

| Repo | Description |
|------|-------------|
| [Alliance Homelab Infrastructure](https://github.com/timanlemvo/Alliance-homelab-infrastructure) | Network design, DNS, and VLAN policy |
