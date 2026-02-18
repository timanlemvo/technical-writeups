# Grafana OAuth SSO Failure — redirect_uri_mismatch from Unconfigured root_url

**Category:** Identity / SSO
**Environment:** Grafana on Alpine Linux, Authentik
**Date:** February 2026
**Status:** Resolved

---

## Summary

Grafana SSO with Authentik failed with `redirect_uri_mismatch` despite the redirect URI appearing correct in both the Grafana OAuth config and the Authentik provider settings. Root cause: Grafana constructs its redirect URI dynamically from `root_url` in the `[server]` section of `grafana.ini`. That value was commented out and defaulting to `localhost`. Authentik was receiving `http://localhost:3000/login/generic_oauth` instead of the registered `http://192.168.20.40:3000/login/generic_oauth`. Fix was uncommenting and setting `domain` and `root_url` in `[server]`.

---

## Environment

| Component | Detail |
|-----------|--------|
| Grafana | LXC 112, Alpine Linux, `192.168.20.40:3000` |
| Authentik | `192.168.20.10:9000` |
| VLAN | 20 (Management) |
| Auth protocol | OAuth2 / OpenID Connect |
| Alpine service manager | `rc-service` |

---

## Authentik Provider Configuration

In the Authentik admin panel under Applications, Providers, create OAuth2/OpenID Provider:

```
Type:          OAuth2/OpenID Provider
Name:          Grafana OAuth Provider
Client Type:   Confidential
Client ID:     [generated]
Client Secret: [generated]
Redirect URI:  http://192.168.20.40:3000/login/generic_oauth
```

Create an Application binding the provider:

```
Name:     Grafana
Slug:     grafana
Provider: Grafana OAuth Provider
```

---

## Grafana OAuth Configuration

Added to `/etc/grafana/grafana.ini`:

```ini
[auth.generic_oauth]
enabled = true
name = Authentik
allow_sign_up = true
client_id = [client id]
client_secret = [client secret]
scopes = openid profile email
auth_url = http://192.168.20.10:9000/application/o/authorize/
token_url = http://192.168.20.10:9000/application/o/token/
api_url = http://192.168.20.10:9000/application/o/userinfo/
```

```bash
rc-service grafana restart
```

---

## Failure

Clicking "Sign in with Authentik" on the Grafana login page redirected to Authentik correctly. Authentication and MFA completed successfully. Authentik redirected back to Grafana.

Error:

```
redirect_uri_mismatch

The redirect URI in the request did not match a registered redirect URI.
Expected: http://192.168.20.40:3000/login/generic_oauth
Received: http://localhost:3000/login/generic_oauth
```

---

## Investigation

**Step 1 — Check both sides of the redirect URI.**

Authentik registered URI: `http://192.168.20.40:3000/login/generic_oauth`

Grafana's `[auth.generic_oauth]` section has no explicit `redirect_uri` field. Grafana builds this value at runtime. There was nothing to check in the OAuth config itself.

**Step 2 — Capture what Grafana is actually sending.**

Browser DevTools, Network tab, clear, click "Sign in with Authentik", find the authorization redirect request to Authentik's `/application/o/authorize/` endpoint, inspect query parameters:

```
redirect_uri=http%3A%2F%2Flocalhost%3A3000%2Flogin%2Fgeneric_oauth
```

Grafana was sending `localhost:3000` regardless of the registered URI. The problem was not in the OAuth config.

**Step 3 — Find where Grafana constructs the redirect URI.**

Grafana builds its redirect URI as:

```
{root_url} + /login/generic_oauth
```

`root_url` comes from the `[server]` section of `grafana.ini`. Checking that section:

```ini
[server]
;protocol = http
;http_addr =
;http_port = 3000
;root_url = %(protocol)s://%(domain)s:%(http_port)s/
;serve_from_sub_path = false
```

Every line prefixed with `;` is a comment. `root_url` was commented out. When unset, Grafana defaults to `localhost`. That default is correct from Grafana's own perspective but wrong for any external redirect callback.

---

## Fix

Uncomment and set `domain` and `root_url` in the `[server]` section:

```ini
[server]
domain = 192.168.20.40
root_url = http://192.168.20.40:3000/
```

```bash
rc-service grafana restart
```

Re-ran the OAuth flow. DevTools now showed:

```
redirect_uri=http%3A%2F%2F192.168.20.40%3A3000%2Flogin%2Fgeneric_oauth
```

Login completed. Grafana created the SSO user account. Verified under Administration, Users.

---

## Complete Working Configuration

```ini
[server]
domain = 192.168.20.40
root_url = http://192.168.20.40:3000/

[auth.generic_oauth]
enabled = true
name = Authentik
allow_sign_up = true
client_id = YOUR_CLIENT_ID
client_secret = YOUR_CLIENT_SECRET
scopes = openid profile email
auth_url = http://192.168.20.10:9000/application/o/authorize/
token_url = http://192.168.20.10:9000/application/o/token/
api_url = http://192.168.20.10:9000/application/o/userinfo/
```

All other `grafana.ini` values remain at defaults.

---

## Why This Is Hard to Find

The error points to the redirect URI. The natural place to look is the OAuth config section and the Authentik provider. Both appear correct. There is no `redirect_uri` field in `[auth.generic_oauth]` to check. Most SSO setup guides only document that section.

The `[server]` section is documented separately under Grafana's general configuration reference and is not cross-referenced in SSO integration guides. Without knowing that Grafana builds the redirect URI from `root_url`, there is no obvious next step.

Capturing the actual network request in DevTools is the reliable shortcut. The `redirect_uri` query parameter shows exactly what Grafana is sending, regardless of what any config file suggests it should be sending. That's the starting point for any `redirect_uri_mismatch` where the configured values look correct.

---

## Notes on Alpine Linux

Alpine uses `rc-service` instead of `systemctl`. Commands that will silently fail:

```bash
# Wrong on Alpine
systemctl restart grafana
systemctl status grafana

# Correct
rc-service grafana restart
rc-service grafana status
```

---

## Lessons Learned

When a redirect URI mismatch error occurs and the configured values look correct, capture the actual authorization request in DevTools and read the `redirect_uri` parameter directly. That shows what the application is actually sending.

Grafana's `[server]` section contains commented-out defaults that are not the same as configured values. A file full of commented options is not a configured file. Check `[server]` explicitly for anything that involves Grafana referring to itself externally.

---

## Related

- [proxmox-vfio-lockup-forensics](../proxmox-vfio-lockup-forensics/) — Grafana was the visualization layer used during the VFIO forensic investigation
- [Alliance Homelab Infrastructure](https://github.com/timanlemvo/Alliance-homelab-infrastructure)
