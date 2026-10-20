---
title: "Proxmox Backups Silently Filled the Root Disk When an NFS Mount Dropped"
date: 2026-09-12
featured: false
category: "Storage"
tags: ["Proxmox", "vzdump", "NFS", "Backups", "Storage"]
summary: "On two different nodes, an NFS backup mount failed silently and vzdump kept 'succeeding' by writing into the mount point on the root filesystem. Months of backups landed on pve-root. Fix was is_mountpoint on the storage definition, plus retention that matches the disk."
---

# Proxmox Backups Silently Filled the Root Disk When an NFS Mount Dropped

**Category:** Storage / Backups
**Environment:** Proxmox VE 9.x, NFS export from Node-B (USB SSD), vzdump jobs on Node-A and Node-C
**Date:** August and September 2026
**Status:** Resolved on both nodes

---

## Summary

The fleet's local backup target is a directory storage backed by an NFS export from Node-B. When that mount failed, nothing alerted, and vzdump kept running because the mount point is an ordinary directory on the root filesystem. Backups were written there instead of to the drive. The same failure happened on two nodes, months apart:

| Node | Mount failed | Effect | Recovered |
|------|--------------|--------|-----------|
| Node-A | 14 April: the mount never succeeded after setup, and `nofail` hid it | Root disk reached 85% | 66GB reclaimed, root fell to 11% |
| Node-C | Around 24 July | About 46GB of vzdump data written into pve-root | Root fell from 65% to 14% |

---

## Impact

- Node-A: 66GB reclaimed, root filesystem from 85% to 11%. Node-C: about 46GB of stray backups removed, root from 65% to 14%.
- The failure class is closed: with `is_mountpoint` set, a missing mount fails the backup job instead of filling the root disk.
- Local retention is bounded (`keep-last=5` on the storage, `keep-last=3` on the Node-A job) while Azure keeps the full history.

---

## Symptom

The Proxmox root volume (`pve-root`) was filling on a node that should have had nearly nothing on it. Backup jobs reported success. The backup drive itself looked healthy and had free space.

---

## Isolation

1. **Find what is using the root filesystem.** Large vzdump archives were present under the backup storage's directory path, on the root volume rather than on the NFS drive.
2. **Check whether the path is a mount.** On the affected node the NFS mount was not active. The directory existed, so Proxmox treated it as valid storage.
3. **Understand why the data seemed to vanish.** After the mount was restored, the files written during the outage were hidden underneath it, because a mount covers whatever the underlying directory contains. They were still consuming root disk space, but were not visible at the mount path.

---

## Root Cause

On Node-A the NFS mount had been failing since it was created on 14 April, and the `nofail` mount option kept it from raising any error at boot or afterward. A directory storage definition has no idea whether its path is a mount unless it is told. Without `is_mountpoint`, a missing NFS mount degrades silently: the path still exists, writes still succeed, and they go to the wrong disk.

---

## Resolution

1. **Reach the hidden data.** The underlying root filesystem copy was exposed at a separate path (`/mnt/backup-real`) so it could be examined and salvaged, with `rsync` for anything worth keeping.
2. **Delete the stray copies** from `pve-root` and remove an 81GB `ddrescue` image that was no longer needed.
3. **Set `is_mountpoint yes`** on the storage in `storage.cfg`. If the mount is absent, vzdump now refuses to write instead of filling the root disk.
4. **Fix retention.** `keep-last=5` on the storage and `keep-last=3` on the Node-A job. Time-based retention is not needed locally because the offsite copy in Azure holds the full history (the upload runs with `--delete-destination false`), so the local drive is only a rolling working set.

---

## Evidence

Captured from the terminal sessions at the time and trimmed. Credentials, tokens, and identifiers are removed.

**Node-A, after the fix:**

```
Filesystem            Size  Used Avail Use% Mounted on
/dev/mapper/pve-root   94G  9.6G   80G  11% /
```

**Node-C, with the real mount absent (the hidden copy still on root):**

```
Filesystem            Size  Used Avail Use% Mounted on
/dev/mapper/pve-root   94G   58G   32G  65% /
```

**Node-C, the NFS export once mounted:**

```
192.168.1.11:/mnt/backup-ext on /mnt/backup-ext type nfs4 (rw,relatime,vers=4.2,...,hard,proto=tcp,...)
Filesystem                    Size  Used Avail Use% Mounted on
192.168.1.11:/mnt/backup-ext  938G  630G  261G  71% /mnt/backup-ext
```

**The control that closes it:**

```
# pvesm set ext-backup --is_mountpoint yes
# grep is_mountpoint /etc/pve/storage.cfg
        is_mountpoint yes
```

**Node-C, after removing the hidden copy (root went from 58G used to 12G, about 46G):**

```
Filesystem            Size  Used Avail Use% Mounted on
/dev/mapper/pve-root   94G   12G   78G  14% /
192.168.1.11:/mnt/backup-ext  938G  676G  215G  76% /mnt/backup-ext
```

---

## Verify It Yourself

These commands check the claims above. Run them on the relevant host.

```bash
# Is the backup target actually mounted? (path depends on the storage definition)
findmnt /mnt/pve/ext-backup

# Where is root space going? -x stays on one filesystem, so it sees data hidden under mounts
df -h /
du -xh --max-depth=2 /mnt/pve | sort -h | tail

# Storage definition and status
grep -A8 'ext-backup' /etc/pve/storage.cfg
pvesm status

# Require a real mount before writing
pvesm set ext-backup --is_mountpoint yes
```

---

## What I Would Change

- Alert on the absence of the mount (an Uptime Kuma or Telegraf check), so the failure shows up on the first night and not months later.
- Alert on root filesystem usage on every node.
- Finish the offsite verification and heartbeat for the Azure upload, which is still open.

---

## Takeaways

- **A directory is not a mount.** Tell Proxmox that the path must be a mountpoint.
- **`nofail` is silent by design.** It prevents a boot failure and also removes the only signal that the mount is missing, so it needs a monitoring check to go with it.
- **Silent success is the worst failure mode.** A backup job that cannot fail when its target is gone is not a backup job.
- **Mounts hide data.** When a disk is full and the numbers do not add up, look underneath mount points.
- **The same bug twice means the control is missing.** The second occurrence is what justified fixing the storage definition rather than just cleaning up.
- **Match retention to capacity and keep history offsite.** A small local drive plus a full cloud archive is a sound split.

---

## Related

| Repo | Description |
|------|-------------|
| [Alliance Homelab Infrastructure](https://github.com/timanlemvo/Alliance-homelab-infrastructure) | Backup architecture: vzdump to NFS to Azure Blob |
