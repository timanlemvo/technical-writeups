# Wazuh SIEM Deployment on Debian 13

**Category:** Infrastructure / Deployment
**Environment:** Proxmox VE LXC, Debian 13 (Trixie)
**Date:** February 2026
**Status:** Resolved

---

## Summary

Manual installation of the Wazuh stack on Debian 13 failed across four sequential issues: intermittent DNS resolution, filebeat crashing on start, certificate path mismatches, and a dashboard stuck loading. After each fix introduced more unknown state, the container was destroyed and redeployed cleanly using the official automation script. Agents were then rolled out to all hosts using a reusable provisioning script.

---

## Environment

| Property | Value |
|----------|-------|
| Container ID | LXC 110 |
| OS | Debian 13 (Trixie) |
| IP | `192.168.20.30` |
| VLAN | 20 (Management) |
| Node | Node-C / Gozanti Cruiser |
| Wazuh version | 4.x |

---

## Failure Timeline

### Failure 1 — DNS Resolution During Package Install

`apt-get` operations against `packages.wazuh.com` failed intermittently. Other containers on VLAN 20 resolved the same domain without issues.

```bash
apt-get install wazuh-indexer
# Err:1 https://packages.wazuh.com/4.x/apt stable/main amd64 wazuh-indexer amd64 4.x.x
# Could not resolve 'packages.wazuh.com'
```

Checked `/etc/resolv.conf` — correct nameserver entries. Tested direct resolution:

```bash
dig packages.wazuh.com @192.168.20.x
# intermittently failing
```

Root cause not identified. Likely a race condition in LXC DNS initialization for this specific container.

Workaround: added the Wazuh repo IP directly to `/etc/hosts` to bypass DNS for the install phase, removed afterward.

---

### Failure 2 — Filebeat Crashing on Start

After the indexer and manager came up, filebeat crashed immediately on every start.

```bash
systemctl start filebeat
systemctl status filebeat
# Active: failed (Result: exit-code)

journalctl -u filebeat -n 50 --no-pager
# [Filebeat] Connection to indexer timed out
# [Filebeat] Failed to connect to backoff(elasticsearch(https://127.0.0.1:9200))
```

Verified the indexer was actually responding:

```bash
curl -k -u admin:admin https://127.0.0.1:9200
# {"name":"node-1","cluster_name":"wazuh-cluster"...}
```

Indexer fine. Reviewed `/etc/filebeat/filebeat.yml` and found two certificate path references that didn't match what was actually on disk. Fixed both. Filebeat still crashed. Adjusted timeout values. No change.

Temporary workaround: wrote a Python script to read alert output and POST directly to the indexer API so the dashboard could start populating while investigation continued.

```python
import requests, json, time

INDEXER_URL = "https://127.0.0.1:9200"
ALERTS_LOG = "/var/ossec/logs/alerts/alerts.json"

def ship_events():
    with open(ALERTS_LOG, 'r') as f:
        for line in f:
            try:
                event = json.loads(line.strip())
                requests.post(
                    f"{INDEXER_URL}/wazuh-alerts-*/_doc",
                    json=event,
                    verify=False,
                    auth=("admin", "admin")
                )
            except Exception as e:
                print(f"Error: {e}")
            time.sleep(0.1)

ship_events()
```

This ran in a tmux session. No error handling, no retry logic, not a production solution.

---

### Failure 3 — Certificate Path Mismatches

The dashboard was logging TLS verification failures against the indexer.

```bash
ls -la /etc/wazuh-indexer/certs/
# admin.pem, admin-key.pem, root-ca.pem, node.pem, node-key.pem

grep ssl /etc/wazuh-dashboard/opensearch_dashboards.yml
# opensearch.ssl.certificateAuthorities: ["/etc/wazuh-dashboard/certs/root-ca.pem"]
```

The dashboard was pointing at its own cert directory. The indexer was using a different path. Each component had been configured against a different section of the documentation and was trusting a different CA cert.

Attempted fix: symlinked all cert directories to `/etc/wazuh-certs/`. Some services accepted the change. Filebeat did not.

---

### Failure 4 — Dashboard Stuck Loading

Dashboard reachable at `https://192.168.20.30` but stuck on an infinite loading spinner. Downstream effect of the filebeat failure — no events reaching the indexer means nothing for the dashboard to display.

---

## Decision to Rebuild

At this point the container had accumulated:

- A DNS workaround of unknown lasting effect
- A Python script in tmux replacing a core component
- Symlinked cert directories with unclear scope
- Multiple service restarts in various orders across three failure modes
- No clean record of what had changed from defaults

Continuing meant debugging an unknown starting state. The right call was to start clean.

```bash
pct stop 110
pct destroy 110
```

A new LXC 110 was provisioned from a clean Debian 13 template.

---

## Successful Deployment

```bash
bash <(curl -s https://packages.wazuh.com/4.x/wazuh-install.sh)
```

All three components came up in correct order with certs generated and placed automatically. Dashboard accessible at `https://192.168.20.30` within 15 minutes of starting the script. Generated admin credentials saved immediately.

---

## Agent Deployment

```bash
#!/bin/bash
MANAGER_IP="192.168.20.30"

curl -s https://packages.wazuh.com/4.x/apt/conf/wazuh-archive-keyring.gpg \
  | gpg --dearmor -o /usr/share/keyrings/wazuh-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/wazuh-archive-keyring.gpg] \
  https://packages.wazuh.com/4.x/apt/ stable main" \
  | tee /etc/apt/sources.list.d/wazuh.list

apt-get update -q
WAZUH_MANAGER="${MANAGER_IP}" apt-get install -y wazuh-agent
systemctl enable wazuh-agent && systemctl start wazuh-agent

echo "Agent installed. Verify at https://${MANAGER_IP}"
```

Run as root on any Debian/Ubuntu host.

**Enrolled agents:**

| Host | Role |
|------|------|
| QCM1255 | Proxmox hypervisor |
| FCM2250 | Node-A (Millennium Falcon) |
| OptiPlex7050 | Node-B workloads |
| AdGuard container | DNS layer |

---

## Lessons Learned

Use the script when the goal is a working system, not an educational exercise. The manual path exists to understand the components — it's not the right tool for getting Wazuh operational on Debian 13.

Know when the system state is no longer trustworthy. The signal is when you can no longer accurately describe what the system's configuration actually is. Three overlapping partial fixes crossed that line here.

In a stack where every component verifies TLS against a shared CA, get certificates correct and verified end-to-end before touching anything else. If certs aren't clean, nothing downstream will work cleanly and you'll chase symptoms instead of causes.

Destroying and rebuilding LXC 110 took five minutes. The prior two hours produced a partially working system of unknown configuration.

---

## Current Status

| Component | Status |
|-----------|--------|
| Wazuh Manager | Operational |
| Wazuh Indexer | Operational |
| Wazuh Dashboard | Operational — `https://192.168.20.30` |
| Agent collection | All enrolled hosts reporting |
| File integrity monitoring | Active |
| Active response (automated blocking) | Pending — UniFi API integration |

---

## Related

- [proxmox-vfio-lockup-forensics](../proxmox-vfio-lockup-forensics/) — FCM2250 agent was enrolled before the VFIO lockup incident
- [Alliance Homelab Infrastructure](https://github.com/timanlemvo/Alliance-homelab-infrastructure)
