---
title: "NVIDIA Driver Missing After a Kernel Update: DKMS Without Matching Headers"
date: 2026-10-18
featured: false
category: "AI Infrastructure"
tags: ["NVIDIA", "DKMS", "Debian 13", "Ollama", "VFIO"]
summary: "After a kernel update, the NVIDIA driver in a GPU-passthrough VM failed to load at boot. The kernel had moved ahead of the available linux-headers package, so DKMS could not build the module. Installing the headers and rebuilding fixes it each time. It has recurred, so the permanent fix (a headers meta-package or a pinned kernel) is the open item."
---

# NVIDIA Driver Missing After a Kernel Update: DKMS Without Matching Headers

**Category:** AI infrastructure / Linux
**Environment:** Debian 13 VM (Tantive-III) with an RTX 4000 SFF Ada passed through via VFIO, NVIDIA driver 550.163.01, Docker, Ollama
**Date:** October 2026
**Status:** Resolved each time. A permanent fix (headers following the kernel automatically) is still open

---

## Summary

A recurring problem on the GPU VM (at least three separate occurrences: April, July across three kernels, and October): when a kernel update lands before the matching `linux-headers` package is available, DKMS cannot build the `nvidia-current` module for the new running kernel. After a reboot into kernel 6.12.113+deb13, the driver was not loaded and the GPU workloads could not start. Installing the headers and re-running the DKMS build restored it. The follow-up is to install the `linux-headers-amd64` meta-package so headers follow future kernels, or to pin the kernel on this VM.

---

## Impact

- GPU workloads came back each time within minutes of finding the cause.
- Three recurrences established the pattern, which is what moved the fix from "rebuild the module" to "make headers follow the kernel".

---

## Symptom

- The NVIDIA driver failed to load at boot after the kernel update.
- GPU workloads on the VM could not start without the driver.

---

## Isolation

1. **Separate host from guest.** The host-side VFIO binding on Node-A was checked first: the GPU functions were still bound to `vfio-pci`. The fault was inside the VM.
2. **Check the module for the running kernel.** The `nvidia-current` DKMS module had not been built for the new kernel.
3. **Find why the build did not happen.** The headers for that kernel were not installed.

---

## Root Cause

DKMS builds out-of-tree modules against the kernel headers. With the new kernel installed but its headers missing, the automatic build had nothing to compile against, so the driver was absent for that kernel.

---

## Resolution

1. Installed the headers for the running kernel.
2. Ran `dkms autoinstall`, which rebuilt `nvidia-current` 550.163.01.
3. Follow-up to prevent a fourth occurrence: install the `linux-headers-amd64` meta-package (or pin the kernel) so headers arrive with each kernel. This was identified after the second recurrence and is the open item.
4. Confirmed the driver loads and the GPU is visible to the containers.

---

## Evidence

Captured from the terminal sessions at the time and trimmed. Credentials, tokens, and identifiers are removed.

**Occurrences:**

| When | Running kernel | Missing |
|------|----------------|---------|
| April | Not recorded | Module not built |
| July (6th) | 6.12.90+deb13.1 | Module not built |
| July (mid) | 6.12.95+deb13 | Headers |
| July (26th) | 6.12.96+deb13 | Headers |
| October | 6.12.113+deb13 (from 6.12.111) | Headers |

**July, kernel 6.12.90: driver present, module missing**

```
$ nvidia-smi
NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver.
$ lspci | grep -i nvidia
01:00.0 VGA compatible controller: NVIDIA Corporation AD104GL [RTX 4000 SFF Ada Generation] (rev a1)
$ modprobe nvidia
modprobe: FATAL: Module nvidia-current not found in directory /lib/modules/6.12.90+deb13.1-amd64
```

**July, kernel 6.12.95: the module existed for three older kernels, not the running one**

```
$ dkms status
nvidia-current/550.163.01, 6.12.73+deb13-amd64, x86_64: installed
nvidia-current/550.163.01, 6.12.74+deb13+1-amd64, x86_64: installed
nvidia-current/550.163.01, 6.12.90+deb13.1-amd64, x86_64: installed
$ dkms install nvidia-current/550.163.01 -k 6.12.95+deb13-amd64
Error! Your kernel headers for kernel 6.12.95+deb13-amd64 cannot be found at /lib/modules/6.12.95+deb13-amd64/build or /lib/modules/6.12.95+deb13-amd64/source.
Please install the linux-headers-6.12.95+deb13-amd64 package ...
```

**After installing the headers, DKMS builds and signs the module on its own:**

```
$ apt install linux-headers-6.12.95+deb13-amd64
...
Autoinstall of module nvidia-current/550.163.01 for kernel 6.12.95+deb13-amd64 (x86_64)
Building module(s)......... done.
Signing module /var/lib/dkms/nvidia-current/550.163.01/build/nvidia.ko
Installing /lib/modules/6.12.95+deb13-amd64/updates/dkms/nvidia-current.ko.xz
Running depmod.... done.
```

The October occurrence repeated the same pattern for kernel 6.12.113 (driver 550.163.01), and the host's `vfio-pci` binding was never affected.

---

## Verify It Yourself

These commands check the claims above. Run them on the relevant host.

```bash
uname -r
dkms status
dpkg -l | grep -E 'linux-headers|nvidia'
sudo apt install linux-headers-amd64
sudo dkms autoinstall
nvidia-smi

# On the Proxmox host: confirm the GPU is still bound to vfio-pci
lspci -nnk | grep -A3 -i nvidia
```

---

## What I Would Change

- Alert when `nvidia-smi` fails on the VM, so a missing module is noticed before the workloads fail.
- Hold the kernel update until headers are available, or pin the kernel on the GPU VM.
- Add the post-update checklist to the Alliance Fleet Codex.

---

## Takeaways

- **Out-of-tree drivers depend on headers.** Treat headers as part of the driver's package set.
- **Use the meta-package.** It removes the manual step that causes the recurrence, and it should have been installed after the first occurrence.
- **Check the host before the guest.** A passthrough GPU has two places to fail, and VFIO binding is the first thing to rule out.

---

## Related

| Writeup | Description |
|---------|-------------|
| [Proxmox VFIO Lockup Forensics](../proxmox-vfio-lockup-forensics/) | The host side of the same GPU passthrough setup |
