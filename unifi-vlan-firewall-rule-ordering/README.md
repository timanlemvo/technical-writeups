# UniFi Firewall Rule Ordering — Inter-VLAN Access Blocked Despite Permit Rules

**Category:** Networking / Firewall
**Environment:** UniFi Dream Machine, Proxmox VE LXC, multi-VLAN network
**Date:** February 2026
**Status:** Resolved

---

## Summary

Nginx Proxy Manager on VLAN 40 was unreachable from the Default network despite existing firewall rules that appeared to permit the traffic. Root cause: those rules had high index numbers and were being evaluated after a deny earlier in the processing order. Fix was creating explicit low-index bidirectional allow rules.

---

## Environment

| Component | Detail |
|-----------|--------|
| Firewall/Router | UniFi Dream Machine |
| NPM Container | LXC 101, `192.168.40.10`, VLAN 40 |
| Management workstation | `192.168.1.174`, Default network |
| Services VLAN | `192.168.40.0/24` |
| Default network | `192.168.1.0/24` |
| NPM admin port | `81` |

---

## Initial State

NPM installed via Proxmox community helper script. Service confirmed running locally:

```bash
ss -tlnp | grep 81
# LISTEN 0 511 0.0.0.0:81

curl -s http://localhost:81 | head -5
# <!doctype html><html lang="en">...
```

From management workstation:

```bash
curl http://192.168.40.10:81
# curl: (7) Failed to connect to 192.168.40.10 port 81: Connection refused
```

---

## Isolation

**Container network config:**

```bash
ip addr show
# inet 192.168.40.10/24 brd 192.168.40.255 scope global eth0

ip route show
# default via 192.168.40.1 dev eth0
```

IP, mask, and gateway correct.

**Same-VLAN gateway reachability:**

```bash
ping 192.168.40.1
# 64 bytes from 192.168.40.1: icmp_seq=1 ttl=64 time=0.8 ms
```

**Cross-VLAN reachability:**

```bash
ping 192.168.1.174
# Request timeout
```

**From workstation to container:**

```bash
ping 192.168.40.10
# Request timeout
```

Container reaches its gateway. Cannot cross VLANs in either direction. This is a firewall issue, not a routing issue. The UniFi gateway creates both networks and knows how to route between them. Firewall rules are dropping the traffic before it can pass.

---

## Existing Rules

UniFi Network showed two rules covering this traffic:

| Rule | Name | Source | Destination | Action |
|------|------|--------|-------------|--------|
| High index | "Allow Management to All" | Default network | Any | Accept |
| High index | "Allow Network 192.168.40.0/24 Traffic" | Services VLAN | Any | Accept |

Both looked correct. Neither was working.

---

## Root Cause

UniFi evaluates firewall rules top-to-bottom by rule index number. Lower index = evaluated first. When a packet matches a rule, evaluation stops.

The existing allow rules had high index numbers, placing them late in the processing order. Something earlier in the chain — a default deny or a more specific block — was matching this traffic first.

After creating new rules with low index numbers (2001, 2002) covering the same traffic, connectivity was immediately established. The original rules were never reaching the matching step.

---

## Fix

Created two explicit rules in UniFi Network under Firewall and Security:

**Rule 1 — Default network to Services VLAN:**

```
Index:       2001
Action:      Accept
Protocol:    All
Source:      Default network (192.168.1.0/24)
Destination: Services VLAN (192.168.40.0/24)
```

**Rule 2 — Services VLAN to Default network:**

```
Index:       2002
Action:      Accept
Protocol:    All
Source:      Services VLAN (192.168.40.0/24)
Destination: Default network (192.168.1.0/24)
```

After applying:

```bash
# From LXC 101
ping 192.168.1.174
# 64 bytes from 192.168.1.174: icmp_seq=0 ttl=64 time=1.2 ms

# From workstation
curl http://192.168.40.10:81
# <!doctype html>... (NPM admin panel)
```

---

## Why Bidirectional Rules

UniFi tracks TCP connection state, but only for sessions where the initial SYN packet was permitted. If outbound is allowed but return traffic is blocked, the TCP handshake appears to complete from the client's side but response packets get dropped. The connection hangs or times out rather than refusing cleanly.

For management access across VLANs, both directions need to be explicitly permitted.

---

## Services Proxied Through NPM Post-Fix

| Service | Backend |
|---------|---------|
| Grafana | `192.168.20.40:3000` |
| Authentik | `192.168.20.10:9000` |
| InfluxDB | `192.168.20.41:8086` |
| Wazuh Dashboard | `192.168.20.30:443` |

---

## Lessons Learned

Confirm the traffic path before debugging the service. NPM was working correctly from the start. A simple ping across VLANs immediately confirmed the right failure layer and avoided time spent debugging the application.

In UniFi, rule index numbers are the actual processing order. When a rule looks correct but isn't working, check what has a lower index number and could be matching the same traffic first.

Test the specific traffic pair in both directions. Pinging the gateway only confirms same-VLAN routing. It does not reveal inter-VLAN firewall behavior.

---

## Related

- [Alliance Homelab Infrastructure](https://github.com/timanlemvo/Alliance-homelab-infrastructure) — Full VLAN layout and network segmentation design
