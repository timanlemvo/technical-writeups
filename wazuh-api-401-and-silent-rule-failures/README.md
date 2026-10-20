---
title: "Wazuh Integration Failures That Gave No Useful Error: API 401 and Silent Rule Mismatches"
date: 2026-09-01
featured: false
category: "Security Operations"
tags: ["Wazuh", "API", "Authentication", "Detection Engineering"]
summary: "A Wazuh API integration returned 401 for a long time because authentication needed HTTP Basic credentials, not a JSON body. Separately, the first real deployment of the custom ruleset found that pcre2-typed field elements failed silently on Wazuh 4.14.2, and that the earlier ruleset had only ever existed in documentation."
---

# Wazuh Integration Failures That Gave No Useful Error

**Category:** Security operations
**Environment:** Wazuh 4.14.2 manager (LXC on Node-C), n8n automation
**Date:** July and August 2026
**Status:** Resolved

---

## Summary

Two integration problems on the Wazuh manager, both silent or misleading:

1. **API 401.** Automation calling the Wazuh API got 401 responses through a long investigation. The authentication endpoint requires HTTP Basic credentials, and the client was sending them in a JSON body.
2. **Rules that did not match.** When the custom ruleset was deployed for real, an earlier "v1.0" ruleset turned out to have existed only in documentation. Version 2.0 (41 rules) was the first live deployment, and it exposed that `type="pcre2"` on field elements fails silently in 4.14.2: the rule loads, nothing errors, and it never matches.

---

## Impact

- Established that the earlier ruleset was documentation only, and deployed the 41-rule version for real.
- Documented two failure modes that produce no useful error: a mismatched authentication scheme and a rule that loads but never matches.

---

## Failure 1: API returns 401

**Symptom:** authentication calls to the Wazuh API (port 55000) returned 401 even though the username and password were correct.

**Isolation:** with the credentials known to be correct, the request format was the remaining variable. The server expected the credentials in the `Authorization` header as HTTP Basic, not as a JSON payload.

**Why it took so long:** a Wazuh upgrade a couple of nights earlier had also auto-rotated the indexer passwords, so a wrong password was a believable explanation for the 401 and sent the investigation the wrong way first. A JSON body of valid credentials and a wrong password return the same "Invalid credentials" message.

**Fix:** send the credentials using Basic authentication, receive the token, and use the token on later calls.

---

## Failure 2: Rules loaded but never fired

**Symptom:** custom detections existed on paper, but the first live deployment did not behave as documented.

**Findings:**

- The v1.0 ruleset had never been deployed. It existed in documentation only.
- After the first real deployment (v2.0, 41 rules), a rule using `type="pcre2"` on a field element loaded without an error and silently never matched on Wazuh 4.14.2.

**Fix:** remove `type="pcre2"` from the `<field>` element and let it use the default OS_Regex matching. The cause was isolated by comparing a working rule with a dead one on the same event stream until only that attribute differed. Rule 100303, written without the attribute, fired; the pcre2 rules did not. After removing the attribute, the suppression for the Entra Connect sync account worked. This is what was observed on 4.14.2; it was not tested on other releases.

**Other silent causes found in the same deployment:** `dstuser` and `srcuser` are decoded fields, not rule tags (the correct static tag is `<user>`), and `timeframe` must be an attribute on the opening `<rule>` tag, not a child element.

**Why it is hard to see:** nothing in the manager log marks the rule as inactive, so the only evidence is the absence of an alert.

---

## Evidence

Captured from the terminal sessions at the time and trimmed. Credentials, tokens, and identifiers are removed.

**Why a detection ruleset mattered (24-hour sample before the new rules):**

```
Total alerts:            16,090
From alliance-dc01:      13,947  (89%)
Two rules responsible:   60106 (Windows logon success), 60137 (Windows logoff), both level 3
```

**The ruleset reality check:** the earlier "25+ custom rules" claim was documentation of a plan. The rules had been drafted as a runbook in March and never written to `local_rules.xml`. Version 2.0 (41 rules across SSH, Windows and AD, privilege escalation, file integrity, self-monitoring, and suppression) was the first real deployment.

**The attribute that silently disables a rule (shape of the fix):**

```xml
<!-- loads without error, never matches on 4.14.2 -->
<field name="win.eventdata.targetUserName" type="pcre2">...</field>

<!-- matches -->
<field name="win.eventdata.targetUserName">...</field>
```

**API authentication that works:**

```bash
curl -sk -u '<user>:<password>' -X POST 'https://<manager>:55000/security/user/authenticate?raw=true'
```

---

## Verify It Yourself

These commands check the claims above. Run them on the relevant host.

```bash
# API authentication uses HTTP Basic
curl -sk -u '<user>:<password>' -X POST 'https://<manager>:55000/security/user/authenticate?raw=true'

# Test a rule against a sample log line
sudo /var/ossec/bin/wazuh-logtest

# Count loaded custom rules
grep -R 'rule id=' /var/ossec/etc/rules/ | wc -l
```

---

## What I Would Change

- Keep one sample event per rule and run it through `wazuh-logtest` on every ruleset change.
- Store the ruleset in Git and deploy from it, so documentation and the manager cannot diverge.
- Re-test every rule that uses `pcre2` on a field element against 4.14.2 and rewrite any that fail.

---

## Takeaways

- **A 401 is not always bad credentials.** Check the authentication scheme the endpoint expects before rotating anything.
- **Documented is not deployed.** Verify a ruleset on the manager, not in the runbook.
- **A rule that never fires looks identical to a quiet network.** Test each rule with a known matching event, for example with the `wazuh-logtest` tool.
- **Write detection tests, not just detections.** Silent parse behavior is the failure mode to design against.

---

## Related

| Writeup | Description |
|---------|-------------|
| [Wazuh SIEM Deployment on Debian 13](../wazuh-deployment-on-debian13/) | Initial deployment of the same manager |
