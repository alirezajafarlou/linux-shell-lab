# Troubleshooting: Web Server 403 Forbidden & Least-Privilege Hardening

- **Status:** Resolved
- **Category:** Linux Security & Permissions
- **Target Environment:** Debian 13 VM (libvirt/KVM)
- **Service Account:** `www-data` (UID 33, GID 33)

---

## 1. Incident Overview

A static web document root located at `/var/www/my-site` failed to deliver content. Testing file reads as the unprivileged web daemon account (`www-data`) triggered an immediate permission denial:

```bash
$ sudo -u www-data cat /var/www/my-site/index.html
cat: /var/www/my-site/index.html: Permission denied
```

In a production web tier (Nginx/Apache), this failure manifests as an **HTTP 403 Forbidden** error to external clients.

---

## 2. Diagnostic Investigation

Checking ownership and permission mode bits on the directory and payload:

```bash
$ ls -ld /var/www/my-site
drwx------ 2 root root 4096 Oct 20 10:45 /var/www/my-site

$ ls -l /var/www/my-site/index.html
-rw------- 1 root root   26 Oct 20 10:45 /var/www/my-site/index.html
```

### Evaluation Breakdown

1. **Owner:** `root:root`
2. **Actor:** `www-data`
3. **Permission Evaluation:**
   - `www-data` is not the user `root`.
   - `www-data` does not belong to the group `root`.
   - The kernel evaluates `www-data` under the **Others** tier.
4. **Directory Block:** `/var/www/my-site` is mode `700` (`drwx------`). Others have `---` (`0`), denying the execute/traversal (`x`) bit necessary to access inodes within the directory.
5. **File Block:** `index.html` is mode `600` (`-rw-------`). Others have `---` (`0`), denying read (`r`) access.

---

## 3. Resolution Strategy: Standard vs. Hardened

### The Naive Fix (Anti-Pattern)
A common approach is opening permissions to all users:
```bash
chmod 755 /var/www/my-site
chmod 644 /var/www/my-site/index.html
```
*Why this is flawed:* It gives every user on the system read access. On shared servers or container hosts, this exposes internal assets and violates least privilege.

### The Production DevSecOps Fix (Hardened)
Bind group access specifically to `www-data` while leaving **Others** completely blocked (`0`).

```text
Target Directory:  Owner: root (7) | Group: www-data (5) | Others: None (0) -> 750
Target File:       Owner: root (6) | Group: www-data (4) | Others: None (0) -> 640
```

---

## 4. Implementation Steps

### Step 1: Assign Group Ownership
Keep owner as `root`, assign group to `www-data`:
```bash
sudo chown -R root:www-data /var/www/my-site
```

### Step 2: Set Hardened Permissions
Restrict Others to `0`, while granting the group read access (and execute traversal on directory):
```bash
sudo chmod 750 /var/www/my-site
sudo chmod 640 /var/www/my-site/index.html
```

---

## 5. Verification & Security Validation

### Test 1: Service Account Access (Success Expected)
```bash
$ sudo -u www-data cat /var/www/my-site/index.html
<h1>DevOps Lab Site</h1>
```
*Result:* Web server successfully reads payload without write access.

### Test 2: Unprivileged User Lateral Movement (Rejection Expected)
```bash
$ ls -l /var/www/my-site/index.html
ls: cannot access '/var/www/my-site/index.html': Permission denied
```
*Result:* Non-root users cannot read, list, or inspect application contents.

---

## 6. Key Takeaways
- **Defense-in-depth:** Read-only access for daemons prevents website defacement and web-shell script persistence even if an RCE exploit occurs in the web tier.
- **`chmod` does not take user names:** `chmod` operates on tiers (`u`/`g`/`o`); targeted access control requires pairing `chmod` with `chown :<group>`.
