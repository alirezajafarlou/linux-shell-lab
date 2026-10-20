# Linux Identity & Permissions Deep-Dive

Notes, architectural models, and command workflows for user identity, group management, and file permissions on Debian 13.

---

## 1. The Linux Identity Model

Linux identifies all users and groups using numeric identifiers:
- **UID `0`**: Superuser (`root`). Full system authority.
- **UID `1–999`**: System and service accounts (`www-data`, `systemd`, `docker`).
- **UID `1000+`**: Human users and interactive login accounts.

### Key System Configuration Files

| File | Standard Mode | Purpose | Format |
| :--- | :---: | :--- | :--- |
| `/etc/passwd` | `644` | Public user registry (world-readable so utilities like `ls` can resolve names) | `username:x:UID:GID:Comment:HomeDir:Shell` |
| `/etc/shadow` | `640` | Secure password hashes and aging info (restricted to `root`) | `username:hash:last_changed:min:max:warn:...` |
| `/etc/group` | `644` | Group definitions and secondary memberships | `groupname:x:GID:member1,member2,...` |

---

## 2. The Permission Model: Binary Bitmask

Linux permissions operate as 3 binary flags ($2^2, 2^1, 2^0$):

| Permission | Symbol | Binary Bit | Numeric Value |
| :--- | :---: | :---: | :---: |
| **Read** | `r` | `100` | **4** |
| **Write** | `w` | `010` | **2** |
| **Execute** | `x` | `001` | **1** |

Because powers of two produce unique mathematical sums, any numeric permission maps unambiguously to a single permission set:

| Mode | Symbolic | Calculation | Meaning |
| :---: | :---: | :---: | :--- |
| **`7`** | `rwx` | $4 + 2 + 1$ | Full access (Read, Write, Execute) |
| **`6`** | `rw-` | $4 + 2 + 0$ | Read and Write |
| **`5`** | `r-x` | $4 + 0 + 1$ | Read and Execute / Traverse |
| **`4`** | `r--` | $4 + 0 + 0$ | Read-only |
| **`0`** | `---` | $0 + 0 + 0$ | No access |

### The Three Scope Tiers

```text
 -   rwx   r-x   r--
 ▲    ▲     ▲     ▲
 │    │     │     └── 3. Others (World: Everyone else on the host)
 │    │     └──────── 2. Group  (Members of assigned GID)
 │    └────────────── 1. User   (File Owner UID)
 Type (- = file, d = directory)
```

---

## 3. Production Permission Profiles

In daily operations, permissions follow standard security profiles:

- **`600` (`-rw-------`)**: Private SSH keys (`id_ed25519`), TLS private keys, secret tokens.
- **`640` (`-rw-r-----`)**: Sensitive application configs readable only by the service group.
- **`644` (`-rw-r--r--`)**: Standard publicly readable files, logs, documentation, system configs.
- **`700` (`drwx------`)**: Private directories (`~/.ssh`).
- **`750` (`drwxr-x---`)**: Service directories restricted to the service group and owner.
- **`755` (`drwxr-xr-x`)**: Standard binaries, executables, and world-navigable directories.
- **`777` (`rwxrwxrwx`)**: Anti-pattern. Never use in production environments.

---

## 4. The Directory Traversal Rule (`x` Bit)

Permissions behave differently on directories versus files:

| Permission | On a File | On a Directory |
| :---: | :--- | :--- |
| **`r`** | Read file contents (`cat`, `less`) | List directory entries (`ls`) |
| **`w`** | Modify file contents | Create, delete, or rename files inside |
| **`x`** | Execute as a binary or script | **Enter / traverse directory (`cd`, inode lookups)** |

> **Crucial Rule:** To open or read any file, a process must possess execute (`x`) permissions on **every single parent directory** leading up to that file. If an ancestor directory lacks `x`, access returns `Permission denied` regardless of file permissions.

---

## 5. Group Workflows & User Private Groups (UPG)

- **UPG Scheme:** Every user receives an eponymous private group on creation. This ensures files created with standard `umask` are not automatically shared with other system users.
- **SetGID on Directories (`chmod g+s`):** When set on a shared workspace, newly created files automatically inherit the group of the parent directory rather than the creating user's primary group.

---

## 6. Command Reference

```bash
# Manage Users and Groups
sudo groupadd developers
sudo useradd -m -s /bin/bash -g developers deployer
sudo usermod -aG developers mr_alj

# Inspect Identity
id deployer
groups mr_alj

# Ownership Management
sudo chown deployer:developers /opt/app-deploy
sudo chown -R deployer:developers /opt/app-deploy

# Permission Assignment
sudo chmod 750 /opt/app-deploy
sudo chmod 640 /opt/app-deploy/.env

# Enable SGID on Directory
sudo chmod g+s /opt/app-deploy
```
