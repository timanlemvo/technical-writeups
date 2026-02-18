# Technical Writeups

> In-depth engineering documentation — incident forensics, infrastructure hardening, and systems troubleshooting from a production-grade homelab environment.

---

These writeups document real problems encountered, investigated, and resolved on live infrastructure. Each includes full methodology, CLI evidence, root cause analysis, and lessons learned. They are written to the standard expected in production postmortems and engineering retrospectives.

## Writeups

| Writeup | Summary | Key Skills |
| --- | --- | --- |
| **[Proxmox VFIO Lockup Forensics](./proxmox-vfio-lockup-forensics/)** | Diagnosed a silent hard lockup on a Proxmox node with zero local crash artifacts. Used external Telegraf/InfluxDB telemetry to reconstruct a second-by-second timeline and traced the root cause to a PCIe bus stall from an NVIDIA GPU under VFIO passthrough. | Linux kernel troubleshooting, incident response, time-series forensics, hardware fault isolation |
| **[Wazuh SIEM Deployment on Debian 13](./wazuh-deployment-on-debian13/)** | Manual installation of the Wazuh SIEM stack failed across four sequential failure modes on Debian 13. Documented each failure, partial mitigations, and the operational decision to destroy and rebuild using the automation script. Includes reusable agent deployment script. | Linux systems troubleshooting, PKI debugging, service dependency resolution, operational decision-making, infrastructure automation |
| **[UniFi Firewall Rule Ordering — Inter-VLAN Access](./unifi-vlan-firewall-rule-ordering/)** | Nginx Proxy Manager on VLAN 40 was unreachable from the Default network despite existing permit rules. Root cause traced to high rule index numbers being evaluated after an earlier deny. Resolution required explicit low-index bidirectional allow rules. | Network troubleshooting methodology, firewall rule evaluation logic, inter-VLAN routing, UniFi Network configuration |
| **[Grafana OAuth SSO — redirect_uri_mismatch from Unconfigured root_url](./grafana-authentik-sso-oauth-misconfiguration/)** | Grafana SSO integration with Authentik failed with redirect_uri_mismatch despite matching OAuth configuration. Root cause: Grafana constructs redirect URIs dynamically from `root_url` in the `[server]` section, which was commented out and defaulting to localhost. Identified using browser DevTools network capture. | OAuth2/OIDC flow analysis, browser DevTools debugging, multi-component configuration troubleshooting, identity provider integration |

*More writeups will be added as new incidents, hardening projects, and infrastructure challenges are documented.*

---

## Related

| Repo | Description |
| --- | --- |
| [Alliance Homelab Infrastructure](https://github.com/timanlemvo/Alliance-homelab-infrastructure) | Full architecture, network design, security stack, and project portfolio for the 3-node Proxmox cluster these writeups are based on |

---

**Contact:** [LinkedIn](https://linkedin.com/in/timanlemvo) · [Portfolio](https://tima.dev) · [timanlemvo@gmail.com](mailto:timanlemvo@gmail.com)
