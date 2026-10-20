---
title: "A Zero-Byte Script Exits 0: Why Rotated Secrets Never Reached the Service"
date: 2026-10-13
featured: false
category: "Secrets Management"
tags: ["Infisical", "systemd", "Bash", "Secrets", "CLI"]
summary: "Rotated API keys were not appearing in an agent's environment. The secrets-bridge script had been emptied to zero bytes, and an empty script exits successfully, so the .env file went stale with no error. The earlier hang in the same script came from a silenced CLI and a missing domain flag."
---

# A Zero-Byte Script Exits 0: Why Rotated Secrets Never Reached the Service

**Category:** Secrets management / Reliability
**Environment:** Infisical CLI and self-hosted Infisical, a systemd `ExecStartPre` hook, Hermes agent on Coruscant
**Date:** October 2026
**Status:** Resolved

---

## Summary

During a secret rotation, new values stored in Infisical never appeared in the agent's `.env` file. The script that bridges Infisical secrets into that file (`jocasta-secrets.sh`) had been emptied to zero bytes earlier in the session. A zero-byte shell script runs and exits with status 0, so the service started normally with stale credentials and nothing logged an error. The same script had previously hung for a different reason.

---

## Impact

- Rotated secrets now reach the service, and a failure to fetch them stops the service instead of starting it with stale values.
- The old credentials were revoked after the new ones were confirmed in the process environment.

---

## Environment

| Component | Detail |
|-----------|--------|
| Secrets store | Infisical (`secrets.tima.dev`), machine identity with universal auth |
| Bridge | `jocasta-secrets.sh`, run before the service starts and writing `.env` |
| Consumer | Hermes agent: Exa API key and dashboard secret were the values being rotated |

---

## Symptom

- New secrets existed in Infisical but did not appear in `.env`.
- The service started cleanly with the old values.
- No error appeared anywhere in the unit's logs.

---

## Isolation

1. **Check the producer end.** The values were correct in Infisical, so the fault was in the bridge.
2. **Look at the script itself.** It was 0 bytes.
3. **Understand why that was invisible.** An empty file executes as a no-op and returns 0, so the `ExecStartPre` step succeeded.

---

## Root Cause

Two separate problems in the same script:

1. **The silent no-op.** The empty script exited 0, and nothing validated that `.env` had actually been refreshed.
2. **The earlier hang.** The original script used `--silent` together with a stdout redirect, which blocked the CLI. The `export` step also ran without `--domain` or `INFISICAL_TOKEN`, so it fell into an interactive login prompt that a service can never answer.

---

## Resolution

The script was rewritten and hardened:

- `set -euo pipefail`, so any failing command stops the script with a non-zero status
- Token captured with `--plain`
- `INFISICAL_TOKEN` passed through to the `export` command
- `--domain` set on both `login` and `export`
- Fail-closed exit: if the secrets cannot be fetched, the unit fails instead of starting with stale values

Two secrets (the Exa API key and the dashboard secret) were then rotated again, moved into Infisical, and confirmed to arrive in the environment; the old key was revoked.

---

## Evidence

Captured from the terminal sessions at the time and trimmed. Credentials, tokens, and identifiers are removed.

**The stale environment file:** the agent's `.env` still held 26 lines and none of the new variables, while the new values existed in Infisical. The unit started cleanly.

**An earlier version of the script hanging on login, then failing:**

```
INF No valid login session found, triggering login flow
error: ^C
Unable to parse domain url
Failed to automatically trigger login flow. Please run [infisical login] manually to login.
ERROR: Infisical export failed
FAILED
```

**The credentials were fine, the script was not:**

```
>>>> Successfully authenticated with universal auth!
```

**Cause of the silent no-op:** the script file had been truncated to 0 bytes earlier in the session. The Exa key was regenerated and the old one revoked, the dashboard secret was regenerated, and both now reach the process through the bridge script.

---

## Verify It Yourself

These commands check the claims above. Run them on the relevant host.

```bash
# A zero-byte script is a no-op that exits 0
: > /tmp/empty.sh; bash /tmp/empty.sh; echo "exit=$?"

# Guards worth putting in the script
set -euo pipefail
[ -s "$ENV_FILE" ] || { echo "env file empty" >&2; exit 1; }

# Size and modification time of the real script
stat -c '%s bytes, modified %y' /path/to/jocasta-secrets.sh
```

---

## What I Would Change

- Add a startup check that fails if the script is empty or if the generated `.env` is older than the fetch.
- Track the script in Git so an accidental truncation shows up as a diff.
- Finish migrating the remaining services' secrets into Infisical.

---

## Takeaways

- **An empty script is a success.** Add a guard that checks the output file was written.
- **Fail closed.** A secrets hook that can fail open will eventually serve stale credentials.
- **Do not silence a CLI you are depending on.** Quiet flags and redirects can change its behavior.
- **A service cannot answer an interactive prompt.** Every non-interactive path needs its token and domain explicit.
- **Confirm rotation end to end.** The value arriving in the process matters, not the value sitting in the vault.

---

## Related

| Repo | Description |
|------|-------------|
| [Alliance Homelab Infrastructure](https://github.com/timanlemvo/Alliance-homelab-infrastructure) | Infisical rollout and credential migrations |
