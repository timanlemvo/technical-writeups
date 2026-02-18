# Diagnosing a Silent Hard Lockup on a Proxmox VFIO Node Using Only External Telemetry

**Category:** Incident Forensics
**Environment:** Proxmox VE, 3-node cluster, Debian, NVIDIA RTX 4000 Ada VFIO passthrough
**Date:** February 9, 2026
**Status:** Resolved

---

## Summary

Node-A (FCM2250) hard-locked with zero local crash artifacts. No kernel panic, no pstore dump, no journal entries. `log2ram` was buffering logs in RAM and the hard lockup prevented the final sync to disk. The investigation pivoted to external Telegraf metrics in InfluxDB, which reconstructed a precise second-by-second timeline, ruled out every software cause, and traced the failure to a PCIe bus stall from an NVIDIA GPU under VFIO passthrough.

---

## Environment

| Property | Value |
|----------|-------|
| Affected node | FCM2250 / Node-A / "Millennium Falcon" |
| OS | Proxmox VE, kernel 6.17.9-1-pve |
| GPU | NVIDIA RTX 4000 Ada, VFIO passthrough |
| Telemetry | Telegraf → InfluxDB 2.x (Node-B), 10-second intervals |
| Logging | log2ram (identified as the critical blind spot) |

---

## The Problem

At approximately 07:55 AM PST on February 9, 2026, FCM2250 became completely unresponsive. Red question mark in the Proxmox UI, zero network activity on the UniFi switch port, machine powered on but unreachable. Required a physical hard reset.

`journalctl -b -1` returned logs only up to January 31, nine days before the crash. The node was running `log2ram`, which holds system logs in a tmpfs and periodically syncs to disk. Because the failure was instantaneous, no final sync occurred and all logs from February 1-9 were irrecoverably lost.

With no local crash data, the investigation had to rely entirely on external telemetry.

---

## Investigation

### Phase 1: Establish the Timeline from a Cluster Peer

Corosync logs from the surviving peer node (QCM1255, Node-B):

```bash
journalctl -u corosync --since "yesterday"
```

```
Feb 09 07:55:07 QCM1255 corosync[1374]: [KNET] host: 1 has no active links
```

A corosync event from Feb 6 was cross-referenced and confirmed as an intentional reboot during GPU passthrough setup, not a prior instance of the same failure.

**Established:** FCM2250 crashed at approximately 07:55 AM PST.

---

### Phase 2: Confirm the Log Gap

```bash
journalctl -b -1 --no-pager | tail -100
```

Previous boot logs ended on January 31. The `log2ram.service` entry confirmed all runtime logs lived in RAM and were destroyed by the lockup before syncing to disk.

`log2ram` is designed to reduce SD card wear on SBCs. On a production hypervisor with NVMe storage, the tradeoff of losing all crash forensics on hard lockup is not acceptable.

---

### Phase 3: Pivot to External Telemetry

Confirmed FCM2250 data existed in InfluxDB on Node-B:

```bash
influx query 'from(bucket: "telegraf")
  |> range(start: -24h)
  |> keep(columns: ["host"])
  |> distinct(column: "host")'
```

FCM2250 was among 9 hosts actively reporting. Telegraf had been collecting `cpu`, `mem`, `disk`, `diskio`, `net`, `netstat`, `kernel`, `processes`, `swap`, and `system` at 10-second intervals. The external pipeline was intact.

---

### Phase 4: Determine the Exact Crash Timestamp

Used the `uptime` metric to find the precise moment data stopped:

```bash
influx query 'from(bucket: "telegraf")
  |> range(start: 2026-02-09T12:00:00Z, stop: 2026-02-09T19:12:00Z)
  |> filter(fn: (r) => r.host == "FCM2250")
  |> filter(fn: (r) => r._measurement == "system")
  |> filter(fn: (r) => r._field == "uptime")
  |> filter(fn: (r) => r._value > 100000)
  |> sort(columns: ["_time"])
  |> tail(n: 10)'
```

Last pre-crash datapoint: **15:54:50 UTC (7:54:50 AM PST)**. Uptime was incrementing normally up to that exact moment, then stopped.

This aligned with the corosync alert at 07:55:07 PST. The 17-second delta is corosync's token timeout for detecting a failed member.

A second query confirmed the post-reboot uptime reset appeared at **19:12 UTC**, marking when the hard reset occurred.

| Time (PST) | Event |
|---|---|
| 07:54:50 | Last Telegraf metric from FCM2250 |
| 07:55:07 | Corosync declares host unreachable (17s detection latency) |
| 11:12:00 | Hard reset performed, node reboots |

---

### Phase 5: Analyze System State at Failure

Final 5 minutes of telemetry pulled across all measurements:

```bash
influx query 'from(bucket: "telegraf")
  |> range(start: 2026-02-09T15:50:00Z, stop: 2026-02-09T15:55:00Z)
  |> filter(fn: (r) => r.host == "FCM2250")
  |> filter(fn: (r) => r._measurement == "cpu"
    and r._field == "usage_idle"
    and r.cpu == "cpu-total")'
```

| Metric | Pre-Crash Value | Anomaly |
|--------|----------------|---------|
| CPU idle | 99.78–99.85% | None |
| Memory used | 7.4% | None |
| Network traffic | ~120 KB/s | None |
| Network errors | 0 across all counters | None |

The system was completely idle at the moment of failure. Not a resource exhaustion event.

---

### Phase 6: Check for Kernel Crash Artifacts

```bash
ls /sys/fs/pstore/
# empty

dmesg | grep -iE "mce|machine.check|panic|error"
# only benign firmware-load warnings (iwlwifi, regulatory.db)
# no MCE, no panic, no hardware errors
```

No kernel panic. No Machine Check Exception. The CPU did not detect a fault, meaning the failure happened below the level the kernel could observe.

---

### Phase 7: Examine the VFIO Configuration

GPU passthrough had been configured 3 days prior to the crash:

```bash
dmesg | grep -i vfio
# vfio-pci claimed 10de:27b0 (RTX 4000 Ada) and 10de:22bc (audio function)

cat /etc/modprobe.d/*
# nouveau/nvidia blacklisted, softdeps in place, vfio-pci ids set
```

Configuration was correct. NVIDIA GPUs under VFIO passthrough are a known cause of PCIe bus stalls: the GPU hangs, the PCIe bus locks, and the entire host freezes without triggering any kernel panic or MCE because the CPU cannot execute the crash handler while the bus is stalled.

**Root cause: PCIe bus stall from the NVIDIA GPU under VFIO passthrough.**

Evidence:
- GPU passthrough configured 3 days prior
- System completely idle at failure
- No kernel panic, MCE, or pstore artifact
- Instantaneous total lockup
- Known failure mode for NVIDIA consumer GPUs under VFIO

---

## The Fix

### Kernel Parameters

```bash
# /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt pcie_aspm=off pci=noaer"

update-grub && proxmox-boot-tool refresh
```

| Parameter | Purpose |
|-----------|---------|
| `pcie_aspm=off` | Disables PCIe Active State Power Management, prevents GPU power state transitions that trigger bus stalls |
| `pci=noaer` | Disables Advanced Error Reporting, prevents AER recovery attempts from stalling the bus further |

Verify after reboot:

```bash
cat /proc/cmdline
# BOOT_IMAGE=/boot/vmlinuz-6.17.9-1-pve ... pcie_aspm=off pci=noaer
```

### GRUB Syntax Gotcha

New parameters were pasted inside the existing value rather than replacing it, producing a mangled command line only caught by the post-reboot `cat /proc/cmdline` check. Fixed with:

```bash
sed -i 's/^GRUB_CMDLINE_LINUX_DEFAULT=.*/GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt pcie_aspm=off pci=noaer"/' /etc/default/grub
```

Always verify with `cat /proc/cmdline` after reboot. Build it into the procedure.

### Disable log2ram

```bash
systemctl disable log2ram
```

---

## Lessons Learned

External telemetry saved this investigation. With local logs destroyed, the only reason root cause analysis was possible is because Telegraf was shipping metrics to an external InfluxDB at 10-second granularity. Without it, the conclusion would have been "it crashed, we don't know why."

`log2ram` is a liability on servers. It makes sense on SBCs preserving SD card life. On a Proxmox node with NVMe storage, it destroys crash evidence in exchange for negligible write reduction.

NVIDIA VFIO passthrough can cause undetectable host lockups. A GPU PCIe bus stall freezes the entire host without producing any kernel panic, MCE, or pstore artifact. The host appears powered on but is completely unresponsive. `pcie_aspm=off` is the standard mitigation.

The 17-second corosync detection latency is measurable and useful for calibrating monitoring alert thresholds.

Always verify boot parameter changes. A copy-paste error in `/etc/default/grub` went undetected until `cat /proc/cmdline` caught it. Build the verification step into the procedure.

---

## Related

| Repo | Description |
|------|-------------|
| [Alliance Homelab Infrastructure](https://github.com/timanlemvo/Alliance-homelab-infrastructure) | Full cluster architecture, Node-A hardware specs, and observability stack design |
| [technical-writeups](https://github.com/timanlemvo/technical-writeups) | Index of all incident documentation |

---

*Investigation conducted February 9, 2026. Node has remained stable since applying the PCIe mitigations.*
