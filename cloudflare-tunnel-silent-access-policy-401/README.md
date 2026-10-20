---
title: "Cloudflare Tunnel Returning Empty 401s: A Zero Trust Access Policy the Wizard Attached Silently"
date: 2026-10-16
featured: false
category: "Networking"
tags: ["Cloudflare", "Tunnel", "Zero Trust", "NPM", "TLS", "SNI"]
summary: "External requests to a self-hosted analytics endpoint returned empty 401s while internal requests worked. The Cloudflare 'Published application' wizard had attached a Zero Trust Access policy that did not appear in the Access Applications list. Recreating the route without Access, and fixing the origin URL and SNI, resolved it."
---

# Cloudflare Tunnel Returning Empty 401s

**Category:** Networking / Edge
**Environment:** Cloudflare Tunnel (`cloudflared` on Phoenix-Nest), Nginx Proxy Manager, Umami analytics, tima.dev portfolio
**Date:** October 2026
**Status:** Resolved

---

## Summary

The portfolio's tracking script and event endpoint (`/script.js` and `/api/send` on `analytics.tima.dev`) returned empty-body 401 responses to external clients. Requests from inside the network to the same service worked. The cause was a Zero Trust Access policy that the tunnel's "Published application" setup wizard had silently attached to the hostname, and it did not show up in the Access Applications list. A second problem, wrong origin URL and SNI, was found while fixing the first.

---

## Impact

- External pageview tracking for the portfolio and blog works again, confirmed from an external device.
- Found and removed an access policy that the Cloudflare Access Applications list did not show.
- Corrected the tunnel origin (the reverse proxy instead of the DNS host) and the TLS and SNI settings.

---

## Environment

| Component | Detail |
|-----------|--------|
| Tunnel | `alliance-fleet`, runtime is a container on Phoenix-Nest |
| Origin | Nginx Proxy Manager at `192.168.1.101` (wildcard certificate for `*.tima.dev`) |
| Service | Umami on Phoenix-Nest, host port 3500 |
| Client | tima.dev (Astro) loading the tracking script |

---

## Symptom

- External requests to `/script.js` and `/api/send` returned HTTP 401 with an empty body.
- A request made from inside the network, bypassing Cloudflare, worked.
- No WAF rule or bot rule explained it.

---

## Isolation

1. **Split the path.** Internal success plus external failure puts the fault in Cloudflare or the tunnel, not the application.
2. **Read the Cloudflare Security Events.** The entry for the request showed `originResponseStatus=401` with `securityAction=unknown`. A 401 that reached the log with an unknown action is not a WAF or bot decision.
3. **Check Access.** The Access Applications list showed nothing for this hostname, yet requests behaved as if an Access policy applied.
4. **Inspect how the route was created.** It had been created through the tunnel's "Published application" wizard, which can attach an Access policy as a side effect.

---

## Root Cause

The wizard attached a Zero Trust Access policy to `analytics.tima.dev` that was not visible in the Access Applications list, so every unauthenticated external request was rejected with a bare 401.

---

## Resolution

1. Deleted the published route (which also removed its DNS record).
2. Re-added it as a plain Published application, with no Access policy.
3. Fixed the service URL. It pointed at `http://192.168.1.4`, which is the AdGuard host, not the reverse proxy. The correct origin is `https://192.168.1.101` (NPM).
4. Turned on "Disable TLS certificate verification" for the origin, and "Match SNI to host". NPM had rejected the TLS handshake with "unrecognized name" when the SNI was the raw IP address.
5. Confirmed end to end by loading the portfolio from an external device and seeing pageviews arrive. The Astro layout was changed to load the script directly from `https://analytics.tima.dev/script.js`.

---

## Evidence

Captured from the terminal sessions at the time and trimmed. Credentials, tokens, and identifiers are removed.

**Where the 401 was NOT coming from.** With logs open on the reverse proxy and the analytics server while reproducing from outside, every line in the proxy host's access log came from the internal client, none from the external test:

```
[15/Oct/2026:13:16:23 -0700] - 401 401 - POST https analytics.tima.dev "/api/auth/verify" [Client 192.168.1.12] [Sent-to 192.168.20.70]
[15/Oct/2026:13:16:24 -0700] - 401 401 - POST https analytics.tima.dev "/api/auth/verify" [Client 192.168.1.12] [Sent-to 192.168.20.70]
```

Eight minutes after the Cloudflare 401 event, no external request had reached the proxy host at all. That moved the fault from the origin to Cloudflare's side of the tunnel.

**The tunnel's route table after the first fix, showing the second problem:**

```
Published applications are matched from top to bottom. The first route that matches handles the request.
1  analytics.tima.dev  Published application  http://192.168.1.4  Using defaults
```

`192.168.1.4` is AdGuard, not the reverse proxy. Internal tests had passed because internal DNS pointed straight at the proxy.

**A temporary workaround used during diagnosis:** the tracking script was briefly self-hosted from the site (`/umami.js`) to keep pageviews working while the tunnel was broken. Once the Access policy was removed, the layout was changed back to the direct script URL and the self-hosted copy was deleted.

**Why internal tests kept passing:** a DNS rewrite sends internal clients straight to the proxy, so they never exercised the tunnel or Cloudflare Access.

---

## Verify It Yourself

These commands check the claims above. Run them on the relevant host.

```bash
# External path (run from outside the network)
curl -sI https://analytics.tima.dev/script.js
curl -s -o /dev/null -w '%{http_code}\n' https://analytics.tima.dev/script.js

# Origin path, bypassing Cloudflare (run from inside the network)
curl -sk --resolve analytics.tima.dev:443:192.168.1.101 -o /dev/null -w '%{http_code}\n' https://analytics.tima.dev/script.js

# Does the origin accept the SNI and serve the right certificate?
openssl s_client -connect 192.168.1.101:443 -servername analytics.tima.dev </dev/null 2>/dev/null | openssl x509 -noout -subject -dates
```

---

## What I Would Change

- Add an external uptime check for the public analytics endpoint, so a path-only failure is detected.
- Manage the tunnel configuration as code instead of through dashboard wizards.
- Review every other published application for the same hidden-policy side effect.

---

## Takeaways

- **If Cloudflare logs a 401 with an unknown security action, suspect the tunnel, not the WAF.**
- **Wizards leave side effects.** Check what a setup flow attached, not just what you asked for.
- **An origin that is an IP needs an SNI override.** Otherwise a name-based proxy rejects the handshake.
- **Point the tunnel at the proxy, not at a neighboring host on the same subnet.**
- **Always test from outside the network.** Internal success proves the application, not the path.

---

## Related

| Repo | Description |
|------|-------------|
| [UniFi Firewall Rule Ordering](../unifi-vlan-firewall-rule-ordering/) | Another case of working internal paths hiding an upstream policy |
