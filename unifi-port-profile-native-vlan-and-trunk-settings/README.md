---
title: "UniFi Port Profiles: A Native Network of 'None' and a Trunk Set to Block All"
date: 2026-10-18
featured: false
category: "Networking"
tags: ["UniFi", "VLAN", "Trunking", "NetGear", "ARP"]
summary: "Raspberry Pis on a new VLAN got DHCP leases but could not reach the gateway, and a Pi behind a second managed switch was unreachable. The causes were a switch port with its native network set to None and an uplink port with Tagged VLAN Management set to Block All."
---

# UniFi Port Profiles: A Native Network of "None" and a Trunk Set to Block All

**Category:** Networking / Switching
**Environment:** UniFi Dream Machine, UniFi US-8-150W, NetGear GS108Tv3 managed switch, VLAN 60
**Date:** October 2026
**Status:** Resolved

---

## Summary

During the Ghost Squadron network build, two port-profile settings caused failures that looked like host problems. A Pi that obtained a lease could not ping its gateway because its port's native network was None. A second Pi behind the NetGear switch was unreachable because the UniFi uplink port's Tagged VLAN Management was set to Block All.

---

## Impact

- Three Pis came online on an isolated VLAN, one of them behind a second managed switch, with firewall behavior tested from every node.
- Left behind a port plan and a gotcha list that make the next VLAN faster to build.

---

## Environment

| Component | Detail |
|-----------|--------|
| Gateway | UniFi Dream Machine, VLAN 60 "Endor" (192.168.60.0/24) |
| Switch 1 | US-8-150W, Pi on a port with the Endor profile |
| Switch 2 | NetGear GS108Tv3, uplinked to US-8 port 7, carries VLAN 60 to two Pis |

---

## Failure 1: Lease but no gateway

**Symptom:** the Pi received an address but pings to the gateway failed, with ARP failures visible.

**Cause:** the port's native network was set to None, so untagged frames from the Pi had no network to belong to.

**Fix:** set the native network to Default.

---

## Failure 2: Pi unreachable behind the second switch

**Symptom:** a Pi on the GS108Tv3 could not be reached, while a Pi on the UniFi switch could.

**Cause:** the US-8 uplink port to the GS108Tv3 (port 7) had Tagged VLAN Management set to Block All, so tagged traffic for VLAN 60 was dropped at the trunk.

**Fix:** native network Default and Tagged VLAN Management changed to Allow All. The GS108Tv3 side carries VLAN 60 tagged on its uplink (g8) and untagged with PVID 60 on the Pi ports.

---

## Related Gotchas From the Same Build

| Gotcha | Detail |
|--------|--------|
| A port change does not force a new DHCP lease | The Pi kept its old lease until the cable was unplugged for about five seconds |
| A drop rule with Destination left as Any | Blocks everything, including internet. Set the destination to the internal-networks list and check the Destination column after saving |
| The GS108Tv3 only keeps changes if the configuration is saved | Unsaved VLAN edits are lost on reboot |
| mDNS names do not cross VLANs | Use IP addresses for hosts on other segments |

---

## Verify It Yourself

These commands check the claims above. Run them on the relevant host.

```bash
# From the Pi
ip -4 addr show eth0
ip route
ip neigh show
arping -I eth0 -c 3 192.168.60.1
sudo tcpdump -ni eth0 arp
```

---

## What I Would Change

- Export the UniFi and GS108Tv3 configuration to Git after each change.
- Replace the interim GS108Tv3 with a PoE UniFi switch so the trunk is managed in one place.
- Add an uptime check per node address, so a bad port profile is noticed immediately.

---

## Takeaways

- **Check the port profile before blaming the host.** Native network and tagged-VLAN policy decide what a port can carry.
- **"Lease but no gateway" is a layer 2 symptom.** Look at ARP and the port's native VLAN.
- **Trunks have two ends.** Verify both, and verify the tagged policy, not just the native one.
- **Save the second switch's configuration.**

---

## Related

| Writeup | Description |
|---------|-------------|
| [UniFi Firewall Rule Ordering](../unifi-vlan-firewall-rule-ordering/) | Inter-VLAN policy troubleshooting on the same platform |
