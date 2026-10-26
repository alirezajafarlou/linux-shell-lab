# Linux Storage Architecture, Filesystems & Inodes

A comprehensive technical reference for block devices, filesystem mounting, UUID persistence, disk usage diagnostics, and inode table architecture on Debian 13.

---

## 1. Storage Abstraction Hierarchy

Linux organizes non-volatile physical storage through layered abstractions:

```text
Physical Hardware Drive (NVMe, SSD, VirtIO virtual disk)
       │
       ▼
Block Device         (/dev/vda, /dev/sda)
       │
       ▼
Partition Slice      (/dev/vda1, /dev/vda2)
       │
       ▼
Filesystem Format    (ext4, XFS, Btrfs - structures data & metadata blocks)
       │
       ▼
Mount Point Hook     (/, /var, /mnt/storage - binds filesystem to VFS tree)
```

### Essential Block Inspection Utilities
- **`lsblk`**: Hierarchical visualization of block devices, partitions, sizes, and active mountpoints.
- **`blkid`**: Queries block device metadata and displays unique **UUIDs** (Universally Unique Identifiers).
  > **DevOps Standard:** Always reference partitions in `/etc/fstab` via `UUID=...` rather than dynamic device names (`/dev/sda1`). Device letters shift when drives are reordered or hot-plugged; UUIDs remain immutable.

---

## 2. Inodes (Index Nodes) vs. Data Blocks

A filesystem (such as `ext4`) divides available disk space into two pools:
1. **Data Blocks:** Storage sectors containing the raw bytes/content of files.
2. **Inode Table:** Fixed array of metadata structures allocated at filesystem formatting (`mkfs`).

### Inode Metadata Structure
An inode stores all file attributes except the file name:
- File ownership (UID, GID)
- Permission bitmask (Mode: `rwxr-xr-x`)
- File byte size & timestamps (atime, mtime, ctime)
- Pointers to underlying data blocks on disk

> **The Directory Abstraction:** In Linux, a directory is a special table containing simple key-value mappings: `filename ──► Inode Number`.

---

## 3. Storage Forensics Toolkit

```bash
# Check filesystem block capacity (Human-readable)
df -h /

# Check filesystem inode table capacity (File count ceiling)
df -i /

# Summarize top-level directory usage inside a path
sudo du -sh /var/* 2>/dev/null | sort -h

# Find the top 10 largest individual files on a filesystem
sudo find /var/log -type f -exec du -h {} + | sort -rh | head -n 10
```
