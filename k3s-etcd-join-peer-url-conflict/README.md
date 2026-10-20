---
title: "K3s Third Server Fails to Join: etcd 'Peer URLs already exists' and an Empty node-ip"
date: 2026-10-18
featured: false
category: "Kubernetes"
tags: ["K3s", "etcd", "Raspberry Pi", "High Availability"]
summary: "Adding the third server to a K3s embedded-etcd cluster took three attempts. The first failed because the install reused another node's --node-ip, which collided with an existing etcd peer. The second failed because the shell variable holding the address was empty. Fix was an explicit, unique --node-ip on every node."
---

# K3s Third Server Fails to Join: etcd "Peer URLs already exists" and an Empty node-ip

**Category:** Kubernetes / High availability
**Environment:** K3s v1.36.5+k3s1, three servers with embedded etcd, Raspberry Pi 4B, Ubuntu Server 26.04 LTS
**Date:** October 2026
**Status:** Resolved

---

## Summary

Phantom (cluster-init) and Gauntlet joined cleanly. The third server, Nightbrother, needed three attempts. Attempt one failed with an etcd error saying the peer URL already existed. Attempt two failed with `invalid node-ip`. Attempt three, with the correct address typed in directly, succeeded and all three nodes reported Ready.

---

## Impact

- A three-server high-availability control plane formed (it tolerates the loss of one node).
- The failure became a checklist: an explicit unique `--node-ip`, print variables before use, join one node at a time.

---

## Environment

| Node | Address | Role |
|------|---------|------|
| Phantom | 192.168.60.11 | Server, `--cluster-init` |
| Gauntlet | 192.168.60.12 | Server |
| Nightbrother | 192.168.60.13 | Server (the node that failed to join) |

---

## Symptom

1. **First attempt:** the join failed with an etcd error, "Peer URLs already exists".
2. **Second attempt:** the join failed with `invalid node-ip`.

---

## Isolation

1. **Read the etcd error literally.** "Peer URLs already exists" means the cluster already has a member advertising that peer address. Nightbrother was trying to register an address that belonged to an existing member.
2. **Compare the install commands.** The command used on Nightbrother carried Gauntlet's `--node-ip` (192.168.60.12), copied from the previous node's install.
3. **Read the second error literally.** `invalid node-ip` with an address variable in the command pointed at an empty value after expansion.

---

## Root Cause

Two separate input mistakes, both in the install command:

1. A copied `--node-ip` that belonged to another member. etcd peer URLs are derived from the node address, so a duplicate collides with an existing member.
2. An address held in a shell variable (`$NODE_IP`) that was empty when the command ran.

---

## Resolution

1. Reinstalled with Nightbrother's own address (192.168.60.13).
2. Typed the address into the command instead of relying on the variable.
3. Confirmed with `kubectl get nodes`: three nodes Ready, with control-plane and etcd roles.

---

## Evidence

Captured from the terminal sessions at the time and trimmed. Credentials, tokens, and identifiers are removed.

**Proving the second failure was an empty variable:**

```
tee@nightbrother:~$ echo "NODE_IP=[$NODE_IP]"
NODE_IP=[]
Oct 17 23:54:54 nightbrother systemd[1]: k3s.service: Main process exited, code=exited, status=1/FAILURE
Oct 17 23:54:54 nightbrother systemd[1]: k3s.service: Failed with result 'exit-code'.
Oct 17 23:54:54 nightbrother systemd[1]: Failed to start k3s.service - Lightweight Kubernetes.
```

**First server healthy before the joins, with memory headroom on a 2GB board:**

```
NAME      STATUS   ROLES                AGE   VERSION
phantom   Ready    control-plane,etcd   33s   v1.36.5+k3s1

               total        used        free      shared  buff/cache   available
Mem:           1.8Gi       856Mi        71Mi       1.6Mi       1.0Gi       976Mi

kube-system   coredns-...                 1/1   Running
kube-system   local-path-provisioner-...  1/1   Running
```

The exact etcd "Peer URLs already exists" line was not saved, so the first failure is described from the error text and the command that produced it.

---

## Verify It Yourself

These commands check the claims above. Run them on the relevant host.

```bash
kubectl get nodes -o wide
kubectl get nodes --show-labels | grep -c etcd
sudo k3s etcd-snapshot ls
```

---

## What I Would Change

- Script the join with a preflight that prints and validates the node IP before installing.
- Store the join token in Infisical and rotate it after cluster formation (still open).
- Copy etcd snapshots off the SD cards.

---

## Takeaways

- **Give every server an explicit, unique `--node-ip`.** On a multi-NIC or multi-address host, never let it be inferred, and never copy it between nodes.
- **Read etcd's error as a membership statement.** It names exactly what collided.
- **Do not trust a variable you did not echo.** Print it before the command that depends on it.
- **A related shell trap:** `sudo` does not carry across `&&`, so a value set before it may not exist after.
- **etcd needs two of three.** After the cluster is formed, reboot one node at a time.

---

## Related

| Writeup | Description |
|---------|-------------|
| [K3s Node Rejects Its Own Certificates](../k3s-node-clock-skew-certificate-rejection/) | Another failure from the same cluster build |
