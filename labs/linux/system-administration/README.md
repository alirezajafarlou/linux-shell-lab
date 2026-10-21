# Linux Service Management, systemd & Centralized Logging

A comprehensive architectural reference and operational guide for managing system services, targets, unit lifecycles, and centralized logs using `systemd` and `journalctl` on Debian 13.

---

## 1. The systemd Architecture

In modern Linux distributions, `systemd` acts as the initialization system (PID 1) and service manager. It controls all background services, network interfaces, storage mounts, and hardware events.

### Unit File Types
Everything managed by `systemd` is represented as a **Unit**:
- **`.service`**: Daemons and background processes (e.g., Nginx, SSH, custom worker scripts).
- **`.target`**: Grouping mechanisms that synchronize system boot states (replaces legacy SysVinit runlevels).
- **`.timer`**: Monotonic or calendar-based job schedulers (modern replacement for `cron`).
- **`.socket`**: IPC or network sockets that activate associated services on incoming traffic (socket activation).

---

## 2. Core Service Operations: `start` vs. `enable`

A common operational anti-pattern is confusing runtime execution with boot persistence:

```text
systemctl start <service>     !=     systemctl enable <service>
```

| Command | Action | Persistence Across Reboot |
| :--- | :--- | :---: |
| `systemctl start <name>` | Immediately spawns the process into RAM. | ❌ No |
| `systemctl stop <name>` | Sends `SIGTERM` (followed by `SIGKILL`) to terminate. | ❌ No |
| `systemctl enable <name>` | Creates symlinks in `/etc/systemd/system/*.wants/` to ensure startup on boot. | ✅ Yes (Does **not** start now) |
| `systemctl disable <name>` | Removes boot symlinks. | ✅ Yes (Does **not** stop running process) |
| `systemctl enable --now` | **DevOps Standard:** Atomically enables for boot AND starts immediately. | ✅ Yes |

---

## 3. System Targets (Boot Profiles)

Targets define the state of the operating system:
- **`multi-user.target`**: Standard non-graphical multi-user server environment with networking (headless server standard).
- **`graphical.target`**: Full GUI desktop environment (includes display manager, X11/Wayland).

```bash
# Query current default target
systemctl get-default

# Optimize headless server (disable GUI overhead)
sudo systemctl set-default multi-user.target
```

---

## 4. Centralized Logging via `journalctl`

`systemd-journald` captures `stdout` and `stderr` streams directly from the process cgroups without requiring file logging libraries:

```bash
# Stream live logs for a specific service (like tail -f)
journalctl -u <service> -f

# Filter to the last 50 entries
journalctl -u <service> -n 50

# Logs from current boot only
journalctl -b

# Filter across the entire OS for errors only (Priority 3)
journalctl -p err
```

### The Invocation ID
Every time systemd starts a service, it generates a unique 128-bit UUID called the **Invocation ID**. This allows precise log isolation for a specific execution run, even if the service restarts frequently.

---

## 5. Unit Anatomy: Writing a Custom Service

Custom administrator unit files reside in `/etc/systemd/system/`:

```ini
[Unit]
Description=Application Service Name
After=network.target

[Service]
Type=simple
User=deployer
Group=developers
ExecStart=/path/to/binary_or_script
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

### Critical Sections Explained:
- **`After=network.target`**: Ensures dependencies (like network sockets) are available before executing.
- **`User=` / `Group=`**: Adheres to least-privilege principles by avoiding root execution.
- **`Restart=always`**: Configures systemd as an automatic supervisor that restarts the process upon failure.
- **`WantedBy=multi-user.target`**: Links the service into the standard boot sequence.

---

## 6. Operational Workflow

Whenever modifying or creating unit files:
```bash
# 1. Inform systemd daemon of disk changes
sudo systemctl daemon-reload

# 2. Control lifecycle
sudo systemctl enable --now <service>
sudo systemctl restart <service>

# 3. Inspect health
systemctl status <service>
```
