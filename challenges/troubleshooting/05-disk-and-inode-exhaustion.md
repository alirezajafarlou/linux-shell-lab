# Troubleshooting: Diagnosing "No Space Left on Device" via Inode Exhaustion

- **Status:** Resolved & Documented
- **Category:** Linux Filesystem & Storage Forensics
- **Target Environment:** Debian 13 VM (libvirt/KVM)

---

## 1. Incident Overview

An administrative operation to create a file on a mounted volume (`/mnt/minidisk`) failed with a critical input/output error:

```bash
$ touch /mnt/minidisk/my-important-file.txt
touch: cannot touch '/mnt/minidisk/my-important-file.txt': No space left on device
```

---

## 2. Diagnostic Investigation & The Paradox

A standard check of filesystem disk space reported abundant available capacity:

```bash
$ df -h /mnt/minidisk
Filesystem      Size  Used Avail Use% Mounted on
/dev/loop0      9.7M  128K  8.9M   2% /mnt/minidisk
```

### Analysis:
- Total space: 9.7 MB.
- Free space: 8.9 MB (**98% Free**).
- **The Paradox:** Despite nearly the entire volume being unallocated, write operations return `ENOSPC` (No space left on device).

---

## 3. Root Cause Analysis (RCA)

Inspecting the inode table using `df -i` exposed the true bottleneck:

```bash
$ df -i /mnt/minidisk
Filesystem     Inodes IUsed IFree IUse% Mounted on
/dev/loop0        496   496     0  100% /mnt/minidisk
```

### Root Cause:
1. When formatted, the filesystem allocated a fixed inode capacity (496 total inodes).
2. A leaking process or unpruned application generated hundreds of 0-byte or micro-files.
3. While these tiny files consumed negligible data blocks (128 KB total), **each file claimed an entire inode metadata slot**.
4. The inode table reached **100% utilization (`IFree = 0`)**.
5. Linux kernels require a free inode to register any new directory entry; without an available inode, file creation throws `No space left on device` regardless of available data gigabytes.

---

## 4. Remediation & Recovery

### Step 1: Identify and Purge Runaway Micro-files
Using `find` to isolate and batch-delete the orphaned files:
```bash
find /mnt/minidisk -name "file_*.txt" -delete
```

### Step 2: Validate Inode Table Recovery
```bash
$ df -i /mnt/minidisk
Filesystem     Inodes IUsed IFree IUse% Mounted on
/dev/loop0        496    11   485    3% /mnt/minidisk
```
Inodes dropped from 100% to 3% utilization.

### Step 3: Verify Write Restoration
```bash
$ touch /mnt/minidisk/my-important-file.txt
$ ls -l /mnt/minidisk/my-important-file.txt
-rw-r--r-- 1 mr_alj mr_alj 0 Oct 26 09:50 /mnt/minidisk/my-important-file.txt
```
File created successfully.

---

## 5. Production DevOps Lessons
- **Monitoring Alerts:** Prometheus/Grafana and Datadog alerts must track both `node_filesystem_free_bytes` **AND** `node_filesystem_files_free` (inodes).
- **Runaway Logging / Cache Accumulation:** Common sources of inode exhaustion in production include unpruned PHP session files (`/var/lib/php/sessions`), excessive Docker build caches, and micro-log files written without rotation policies (`logrotate`).
