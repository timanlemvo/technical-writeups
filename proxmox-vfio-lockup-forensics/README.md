# Diagnosing a Silent Hard Lockup on a Proxmox VFIO Passthrough Node Using Only External Telemetry

> A Proxmox node hard-locked with zero local crash artifacts. No kernel panic, no pstore, no journal entries. External telemetry was the only path to root cause.

---

**Skills Demonstrated:** `Linux Kernel Troubleshooting` · `Incident Response Methodology` · `Time-Series Analysis (InfluxDB/Flux)` · `Hardware Fault Isolation` · `PCIe/IOMMU Architecture` · `Observability Pipeline Design` · `Cluster Administration (Corosync)` · `Post-Mortem Documentation`

**Relevant Roles:** SRE · Platform Engineer · Cloud Security Engineer · Systems Administrator · DevOps Engineer

---

## Executive Summary

A Proxmox VE hypervisor node suffered an unresponsive hard lockup with zero local crash artifacts — no kernel panic, no pstore dump, no journal entries — due to an in-memory logging configuration that discarded 9 days of logs on failure. By pivoting to externally-stored Telegraf metrics in InfluxDB, I reconstructed a precise, second-by-second timeline of the incident, ruled out every software cause, and traced the failure to a PCIe bus stall induced by an NVIDIA GPU configured for VFIO passthrough.

---

## Environment Context

This node is part of a 3-node Proxmox cluster running 25+ services. The diagram below shows where the affected node and the observability pipeline that saved this investigation sit within the broader infrastructure. Full cluster documentation: [homelab-infrastructure](https://github.com/yourname/homelab-infrastructure).

```
┌──────────────────────────────────────────────────────────────────────┐
│                     PROXMOX CLUSTER (3 Nodes)                        │
├──────────────────────┬──────────────────────┬────────────────────────┤
│  NODE-A ◄── CRASHED  │  NODE-B              │  NODE-C                │
│  "Millennium Falcon" │  "CR90 Corvette"     │  "Gozanti Cruiser"     │
│  (FCM2250)           │  (QCM1255)           │  (OptiPlex 7050)       │
├──────────────────────┼──────────────────────┼────────────────────────┤
│  Core Ultra 9        │  Ryzen 7 PRO         │  i7-7700               │
│  64GB DDR5           │  64GB DDR5 ECC       │  32GB DDR4             │
│  RTX 4000 Ada 20GB   │                      │                        │
│  VFIO GPU Passthru   │                      │                        │
├──────────────────────┼──────────────────────┼────────────────────────┤
│  AI/ML Workloads     │  Data & Monitoring   │  Network & Security    │
│  • Ollama, ComfyUI   │  • InfluxDB ◄─ USED │  • Wazuh SIEM          │
│  • OpenWebUI         │  • Grafana  ◄─ USED │  • AdGuard DNS         │
│  • AnythingLLM       │  • Telegraf ◄─ USED │  • Nginx Proxy Mgr     │
│                      │  • Wazuh             │                        │
└──────────────────────┴──────────────────────┴────────────────────────┘

Observability pipeline that enabled this investigation:

  Telegraf (agents on all nodes) ──push──► InfluxDB 2.x ──query──► Grafana
           10-second intervals               Flux queries            Dashboards
```

### Tech Stack (This Investigation)

| Component | Role |
|-----------|------|
| **Proxmox VE** | Hypervisor (3-node corosync cluster, kernel 6.17.9-1-pve) |
| **Telegraf** | Metric collection agents on all nodes (10s interval) |
| **InfluxDB 2.x** | Time-series database (Flux query language) |
| **Grafana** | Dashboard visualization (fresh install at time of incident) |
| **UniFi Switch** | Port-level network activity monitoring |
| **log2ram** | RAM-based journald — identified as the critical blind spot |
| **VFIO/vfio-pci** | GPU passthrough (NVIDIA RTX 4000 Ada, Intel VT-d/IOMMU) |

---

## The Problem

At approximately 07:55 AM PST on February 9, 2026, Proxmox cluster node `FCM2250` (Node-A, "Millennium Falcon") became unresponsive.

**Symptoms:**
- Red question mark in the Proxmox web UI — lost cluster member
- Zero network activity on the node's UniFi switch port
- Machine still powered on (fans spinning, LEDs lit) but completely unresponsive to network or console input
- Required a physical hard reset

**The critical roadblock:** `journalctl -b -1` returned logs only up to **January 31** — nine days before the crash. The node was running `log2ram`, a service that holds system logs in a tmpfs and periodically syncs to disk. Because the failure was instantaneous, no final sync occurred, and all logs from February 1–9 were irrecoverably lost.

With no local crash data — no journal, no panic, no pstore, no dmesg — the investigation had to rely entirely on external telemetry.

---

## Troubleshooting Process

### Phase 1: Establish the Timeline from Cluster Peers

The first step was determining *when* the node dropped out of the cluster, using corosync logs from a surviving peer node (`QCM1255`, Node-B):

```bash
journalctl -u corosync --since "yesterday"
```

**Result:**
```
Feb 09 07:55:07 QCM1255 corosync[1374]: [KNET] host: host: 1 has no active links
```

A similar corosync event on Feb 6 was cross-referenced and confirmed as an intentional reboot during GPU passthrough setup — not a prior incident of the same failure.

**Hypothesis established:** FCM2250 crashed at approximately 07:55 AM PST.

---

### Phase 2: Discover the Log Gap (log2ram)

After the hard reset and reboot, the previous boot's journal was examined for crash artifacts:

```bash
journalctl -b -1 --no-pager | tail -100
```

**Finding:** The previous boot's logs ended on January 31. The `log2ram.service` entry confirmed why — all runtime logs lived in RAM and were destroyed by the hard lockup before they could sync to disk.

**Takeaway noted:** log2ram is a liability on any system where post-mortem analysis matters. It was designed to reduce SD card wear on SBCs, not to run on production hypervisors with NVMe storage.

---

### Phase 3: Pivot to External Telemetry (InfluxDB)

With local logs gone, the investigation shifted to InfluxDB on Node-B, which was receiving Telegraf metrics from all cluster nodes. First, confirm data existed for the crashed host:

```bash
influx query 'from(bucket: "telegraf")
  |> range(start: -24h)
  |> keep(columns: ["host"])
  |> distinct(column: "host")'
```

**Result:** FCM2250 was among 9 hosts actively reporting. Telegraf had been collecting `cpu`, `mem`, `disk`, `diskio`, `net`, `netstat`, `kernel`, `processes`, `swap`, and `system` measurements at **10-second intervals**.

The external telemetry pipeline was intact. The investigation could proceed.

---

### Phase 4: Determine the Exact Crash Timestamp

The system uptime metric was used to binary-search for the precise moment data stopped:

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

**Result:** Last pre-crash data point was at **15:54:50 UTC (7:54:50 AM PST)**. The uptime counter (202,913 seconds) was incrementing normally up to this exact moment, then stopped.

This aligned precisely with the corosync alert at 07:55:07 PST. The ~17-second delta represents corosync's token timeout for detecting a failed member — useful for calibrating monitoring alert thresholds.

A separate query confirmed the post-reboot uptime reset (value: 1,089) appeared at **19:12 UTC (11:12 AM PST)**, marking when the hard reset occurred.

**Timeline established:**

| Time (PST) | Event |
|---|---|
| 07:54:50 | Last Telegraf metric received from FCM2250 |
| 07:55:07 | Corosync declares host unreachable (17s detection latency) |
| 11:12:00 | Hard reset performed, node reboots |

---

### Phase 5: Analyze System State at the Moment of Failure

With the exact crash time established, the final 5 minutes of telemetry were examined across all available metrics to identify any anomaly.

**CPU — completely idle:**
```bash
influx query 'from(bucket: "telegraf")
  |> range(start: 2026-02-09T15:50:00Z, stop: 2026-02-09T15:55:00Z)
  |> filter(fn: (r) => r.host == "FCM2250")
  |> filter(fn: (r) => r._measurement == "cpu"
    and r._field == "usage_idle"
    and r.cpu == "cpu-total")'
```
**Result:** CPU idle hovered at **99.78–99.85%** across all 24 cores for the entire 5-minute window. No spike, no gradual climb, no anomaly.

**Memory — minimal usage:**
**Result:** Memory steady at **7.4% used**. No leak, no pressure.

**Network — normal, zero errors:**
**Result:** Steady ~120 KB/s traffic on `vmbr0`, zero `err_in`, zero `err_out`, zero `drop_in` or `drop_out` counters. No anomaly.

**Summary of findings:**

| Metric | Pre-Crash Value | Anomaly? |
|---|---|---|
| CPU idle | 99.78–99.85% | ❌ No |
| Memory used | 7.4% | ❌ No |
| Network traffic | ~120 KB/s | ❌ No |
| Network errors | 0 across all counters | ❌ No |
| Disk I/O | Normal | ❌ No |

**Conclusion:** The system was functionally idle at the moment of failure. This is not a resource exhaustion event.

---

### Phase 6: Check for Kernel Crash Artifacts Post-Reboot

On the rebooted FCM2250:

```bash
ls /sys/fs/pstore/
# Result: empty — no kernel panic was captured

dmesg | grep -iE "mce|machine.check|panic|error"
# Result: only benign firmware-load warnings (iwlwifi, regulatory.db)
# No MCE (Machine Check Exception), no panic, no hardware errors
```

**Finding:** No kernel panic occurred. No Machine Check Exception was raised. The CPU did not detect a fault — which means the failure happened *below* the level the kernel could observe.

---

### Phase 7: Examine the VFIO Passthrough Configuration

Given that GPU passthrough was configured 3 days prior to the crash, the VFIO setup was inspected:

```bash
dmesg | grep -i vfio
# Confirmed: vfio-pci claimed 10de:27b0 (RTX 4000 Ada) and 10de:22bc (audio function)

cat /etc/modprobe.d/*
# Confirmed: nouveau/nvidia blacklisted, softdeps in place, vfio-pci ids set
```

The configuration was textbook-correct. However, NVIDIA GPUs under VFIO passthrough are known to cause **PCIe bus stalls** — the GPU hangs, the PCIe bus locks, and the entire host freezes without triggering any kernel panic or MCE because the CPU itself cannot execute the crash handler while the bus is stalled.

**Root cause identified:** PCIe bus stall induced by the NVIDIA GPU under VFIO passthrough. The evidence chain:

1. GPU passthrough configured 3 days prior (change correlation)
2. System completely idle at time of failure (not load-related)
3. No kernel panic, MCE, or pstore artifact (failure below kernel observation)
4. Instantaneous, total lockup (consistent with bus stall, not software crash)
5. Known failure mode for NVIDIA consumer GPUs under VFIO passthrough

---

## The Solution

### Immediate Mitigation — Kernel Parameters

Added kernel parameters to `/etc/default/grub` to reduce PCIe-related hang risk:

```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt pcie_aspm=off pci=noaer"
```

```bash
update-grub && proxmox-boot-tool refresh
```

| Parameter | Purpose |
|---|---|
| `pcie_aspm=off` | Disables PCIe Active State Power Management — prevents GPU power state transitions that can trigger bus stalls |
| `pci=noaer` | Disables Advanced Error Reporting handling — prevents AER recovery attempts from stalling the bus further |

**Verified post-reboot:**

```bash
cat /proc/cmdline
# BOOT_IMAGE=/boot/vmlinuz-6.17.9-1-pve root=/dev/mapper/pve-root ro quiet intel_iommu=on iommu=pt pcie_aspm=off pci=noaer
```

### Logging Fix — Disable log2ram

```bash
systemctl disable log2ram
```

Ensures future crashes preserve journal data on disk. The write-reduction benefit of log2ram is negligible on NVMe storage compared to the risk of losing all crash forensics.

### Grub Syntax Recovery

During the initial edit, a malformed `GRUB_CMDLINE_LINUX_DEFAULT` line was introduced — the new value was pasted *inside* the existing value rather than replacing it, resulting in a mangled kernel command line on first reboot. Caught during the `cat /proc/cmdline` verification step. Corrected with:

```bash
sed -i 's/^GRUB_CMDLINE_LINUX_DEFAULT=.*/GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt pcie_aspm=off pci=noaer"/' /etc/default/grub
```

**Lesson:** `sed` one-liners are more reliable than manual editing for single-line config replacements. Always verify with `cat /proc/cmdline` after reboot.

---

## Key Takeaways

### 1. External telemetry saved this investigation.
With local logs destroyed, the only reason a root cause analysis was possible at all is because Telegraf was shipping metrics to an external InfluxDB instance at 10-second granularity. Without that, the conclusion would have been "it crashed, we don't know why." This is why observability pipelines are non-negotiable infrastructure — they're not dashboards, they're forensic tools.

### 2. log2ram is a footgun on servers.
It's designed to reduce SD card / eMMC wear on SBCs, but on a production hypervisor with NVMe storage, the write-reduction benefit is negligible compared to the risk of losing all crash forensics. If write endurance is a concern, `journald`'s rate limiting and log rotation are safer alternatives.

### 3. NVIDIA VFIO passthrough can cause undetectable host lockups.
A GPU PCIe bus stall can freeze the entire host without producing any kernel panic, MCE, or pstore artifact. The host appears powered on but is completely unresponsive. This is a known failure mode with NVIDIA consumer GPUs under passthrough, and `pcie_aspm=off` is a standard mitigation.

### 4. Corosync detection latency is measurable and useful.
The 17-second gap between the last Telegraf data point (15:54:50 UTC) and the corosync alert (15:55:07 UTC) exactly reflects corosync's token timeout for detecting a failed member. This is useful for calibrating monitoring alert thresholds — if your metric pipeline has 10-second resolution and corosync detects in 17 seconds, you can set alerting expectations accordingly.

### 5. Always verify boot parameter changes.
A subtle copy-paste error in `/etc/default/grub` went undetected until the post-reboot `cat /proc/cmdline` verification step caught a mangled command line. Build the verification step into the procedure, not as an afterthought.

---

## Repository Structure

```
proxmox-vfio-lockup-forensics/
├── README.md                          ← You are here
├── evidence/
│   ├── corosync-alert.log             ← Peer node corosync output
│   ├── telegraf-cpu-pre-crash.csv     ← Exported CPU metrics (final 5 min)
│   ├── telegraf-uptime-timeline.csv   ← Uptime values showing exact cutoff
│   └── dmesg-post-reboot.log         ← dmesg output after recovery
├── configs/
│   ├── grub-default                   ← /etc/default/grub (with mitigations)
│   ├── vfio-modprobe.conf             ← vfio-pci driver binding config
│   └── telegraf.conf                  ← Telegraf agent config (what saved us)
├── queries/
│   └── flux-queries.md                ← All InfluxDB Flux queries used
└── diagrams/
    └── crash-timeline.md              ← Visual timeline of the incident
```

---

## Related

| Repo | Description |
|------|-------------|
| [TheAlliance-homelab-infrastructure](https://github.com/timanlemvo/Alliance-homelab-infrastructure) | Full cluster architecture, network design, security stack, and project portfolio |
| [technical-writeups](https://github.com/timanlemvo/technical-writeups) | Index of all technical writeups and incident documentation |

---

*Investigation conducted February 9, 2026. Node has remained stable since applying the PCIe mitigations.*
