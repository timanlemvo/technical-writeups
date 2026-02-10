# Technical Writeups

> In-depth engineering documentation — incident forensics, infrastructure hardening, and systems troubleshooting from a production-grade homelab environment.

---

These writeups document real problems encountered, investigated, and resolved on live infrastructure. Each includes full methodology, CLI evidence, root cause analysis, and lessons learned. They are written to the standard expected in production postmortems and engineering retrospectives.

## Writeups

| Writeup | Summary | Key Skills |
|---------|---------|------------|
| **[Proxmox VFIO Lockup Forensics](proxmox-vfio-lockup-forensics/)** | Diagnosed a silent hard lockup on a Proxmox node with zero local crash artifacts. Used external Telegraf/InfluxDB telemetry to reconstruct a second-by-second timeline and traced the root cause to a PCIe bus stall from an NVIDIA GPU under VFIO passthrough. | Linux kernel troubleshooting, incident response, time-series forensics, hardware fault isolation |

*More writeups will be added as new incidents, hardening projects, and infrastructure challenges are documented.*

---

## Related

| Repo | Description |
|------|-------------|
| [homelab-infrastructure](https://github.com/timanlemvo/Alliance-homelab-infrastructure) | Full architecture, network design, security stack, and project portfolio for the 3-node Proxmox cluster these writeups are based on |

---

**Contact:** [LinkedIn](https://linkedin.com/in/tima-nlemvo) · [Portfolio](https://tima.dev) · timanlemvo@gmail.com
