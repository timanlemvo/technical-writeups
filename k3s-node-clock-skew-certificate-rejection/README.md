---
title: "K3s Node Rejects Its Own Certificates: Clock Skew on a Raspberry Pi With No RTC"
date: 2026-10-18
featured: false
category: "Kubernetes"
tags: ["K3s", "Raspberry Pi", "chrony", "Certificates", "Ubuntu"]
summary: "One node of a new three-node K3s cluster reported 'certificate not yet valid' on its own certificates. Root cause was a stale system clock, not a certificate problem. Fix was correcting the time and letting chrony take over, not regenerating certificates."
---

# K3s Node Rejects Its Own Certificates: Clock Skew on a Raspberry Pi With No RTC

**Category:** Kubernetes / Time synchronization
**Environment:** K3s v1.36.5+k3s1 (3 servers, embedded etcd), Raspberry Pi 4B, Ubuntu Server 26.04 LTS, chrony
**Date:** October 2026
**Status:** Mitigated. Reboot persistence not yet verified (see Remaining Work)

---

## Summary

During the build of Ghost Squadron, a three-node K3s cluster, `kubectl` on one node (Gauntlet) failed with a "certificate not yet valid" error against K3s's own certificates. The certificates were fine. The node's system clock was set to 27 July while the certificates had been issued in October. Setting the date and letting chrony synchronize it cleared the error. The certificates were never touched.

---

## Impact

- The cluster reached three Ready nodes with no certificates regenerated.
- Produced a repeatable first check for any Pi reboot: look at `date` before suspecting K3s.

---

## Environment

| Component | Detail |
|-----------|--------|
| Cluster | 3 K3s servers, embedded etcd, installed 17 to 18 Oct 2026 |
| Nodes | Phantom (cluster-init), Gauntlet, Nightbrother |
| Hardware | Raspberry Pi 4B, 2GB, microSD boot |
| OS | Ubuntu Server 26.04.1 LTS |
| Time sync | chrony |
| Affected node | Gauntlet |

---

## Symptom

On Gauntlet, `kubectl` failed with a TLS validation error stating the certificate was not yet valid. The node had booted with a stale date (27 July) rather than the current one.

---

## Isolation

1. **Certificate validity windows.** A "not yet valid" error means the certificate's `notBefore` date is later than the clock of the machine validating it. It is a different failure from "expired", and it points at the validator's clock before it points at the certificate.
2. **Compare the clocks.** The cluster's certificates were generated at install time in October. The system date on Gauntlet read 27 July, months earlier.
3. **Why a Pi.** A Raspberry Pi has no battery-backed real-time clock. After a power cycle it depends on whatever time source comes up first, so a stale time at boot is possible until NTP corrects it.

---

## Root Cause

Gauntlet booted with a clock set to 27 July. Its local K3s certificates (issued in October) therefore appeared to start in the future, and the node rejected them. Nothing was wrong with the PKI.

---

## Resolution

1. Set the system date by hand to the correct time.
2. Let chrony step the clock to its selected NTP source.
3. Confirmed the `kubectl` error was gone.

Deliberately **not** done: deleting or regenerating K3s certificates. That would have "fixed" the symptom by issuing new certificates with a later start date, hidden the real fault (a bad clock), and risked destabilizing etcd peer trust across the three servers.

A second node (Nightbrother) was also set about two minutes fast by hand during the same cleanup, then stepped back by chrony, which reported synchronized afterward.

---

## Remaining Work

This is mitigated, not closed. These items were still open when this was written:

| Item | Why it matters |
|------|----------------|
| Reboot each Pi, one at a time, checking `date` first and the other nodes' Ready state second | Proves the fix survives a power cycle. Do not reboot a second Pi until the first is Ready, because etcd needs two of three members |
| Confirm `systemctl list-dependencies time-sync.target` includes chrony | Verifies the time-sync target actually waits for chrony |
| If a reboot still produces a stale clock, add a `k3s` service drop-in with `After=chrony.service time-sync.target` and `Wants=time-sync.target` | Orders K3s startup after the clock is correct |
| Check chrony status on Phantom and re-check Gauntlet's synchronized flag | Neither was confirmed at the last check |

---

## Evidence

Captured from the terminal sessions at the time and trimmed. Credentials, tokens, and identifiers are removed.

**The error on Gauntlet:**

```
Unable to connect to the server: tls: failed to verify certificate: x509: certificate has expired or is not yet valid: current time 2026-07-27T16:16:52-07:00 is before 2026-10-18T05:57:30Z
```

**The clock, with no hardware RTC and no sync:**

```
$ timedatectl
               Local time: Mon 2026-07-27 16:17:28 PDT
           Universal time: Mon 2026-07-27 23:17:28 UTC
                 RTC time: n/a
                Time zone: America/Los_Angeles (PDT, -0700)
System clock synchronized: no
              NTP service: active
```

**The first fix attempt did not apply to this image:**

```
$ sudo systemctl restart systemd-timesyncd
Failed to restart systemd-timesyncd.service: Unit systemd-timesyncd.service not found.
```

The image uses chrony for time, so chrony was the service to check. Internet reachability from the node was confirmed first (about 11 ms to 1.1.1.1), which ruled out a network cause for the failed sync.

---

## Verify It Yourself

These commands check the claims above. Run them on the relevant host.

```bash
timedatectl
chronyc tracking
systemctl list-dependencies time-sync.target

# Certificate validity window versus the system clock
sudo openssl x509 -in /var/lib/rancher/k3s/server/tls/client-ca.crt -noout -dates
```

---

## What I Would Change

- Run the one-node-at-a-time reboot test and then close this writeup.
- Add the `After=chrony.service time-sync.target` drop-in if a reboot shows a stale clock.
- Consider an RTC module, or a boot-time check that holds K3s until the clock is plausible.

---

## Takeaways

- **"Not yet valid" is a clock problem first.** Compare the validator's date with the certificate's `notBefore` before touching any certificates.
- **Check `date` before suspecting K3s.** On hardware without an RTC, make it the first step after any reboot.
- **Do not regenerate certificates to solve a time problem.** It masks the cause and can break trust between etcd members.
- **A fix is not verified until it survives a reboot.** This writeup stays marked as mitigated until the reboot test passes.

---

## Related

| Repo | Description |
|------|-------------|
| [Ghost Squadron](https://github.com/timanlemvo/Alliance-homelab-infrastructure/tree/main/projects/ghost-squadron) | The K3s cluster this occurred in |
