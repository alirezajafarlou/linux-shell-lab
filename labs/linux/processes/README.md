# Linux Process Management, Lifecycle & Signals

A comprehensive reference for process architecture, signal handling, process states, and interactive system inspection on Debian 13.

---

## 1. Process Architecture & The Process Tree

A **program** is dormant executable machine code residing on non-volatile storage. A **process** is an active instance of that program loaded into virtual memory with dedicated system resources:
- **PID (Process ID):** Unique identifier assigned by the kernel.
- **PPID (Parent Process ID):** The PID of the process that spawned this process via `fork()` / `exec()`.
- **File Descriptors:** Pointers to standard streams (`stdin 0`, `stdout 1`, `stderr 2`), files, and network sockets.

### The Init Process: PID 1
- When the Linux kernel initializes, it spawns **PID 1** (`systemd` / `/sbin/init`).
- PID 1 is the root ancestor of all user-space processes.
- **Critical Responsibility:** PID 1 adopts orphaned child processes and reaps their exit statuses so they do not permanently leak kernel memory.
- In Docker containers, the application specified in `ENTRYPOINT` or `CMD` runs as PID 1 within the container PID namespace.

---

## 2. Essential POSIX Signals

Signals are asynchronous software interrupts delivered to a process by the kernel or userspace utilities.

| Signal Name | Number | Standard Action | DevOps & Production Context |
| :--- | :---: | :--- | :--- |
| **`SIGHUP`** | **`1`** | Hangup / Terminate | Sent when a controlling terminal disconnects. Modern daemons (Nginx, Prometheus) intercept `SIGHUP` to **reload configuration files without dropping traffic**. |
| **`SIGINT`** | **`2`** | Interactive Interrupt | Sent via `Ctrl + C`. Polite request to abort execution immediately from the terminal. |
| **`SIGKILL`** | **`9`** | Immediate Termination | The kernel forcibly purges the process from RAM. **Cannot be caught, blocked, or ignored by the process.** No cleanup occurs. |
| **`SIGTERM`** | **`15`** | Graceful Termination | **Default signal of `kill`.** Asks the process to clean up resources, flush caches, close database connections, and exit gracefully. |

> **Graceful Shutdown Workflow:** Orchestrators like Kubernetes issue `SIGTERM` when deleting a Pod, wait for a `terminationGracePeriodSeconds` window (default 30s), and only escalate to `SIGKILL` if the process fails to exit.

---

## 3. Process Execution States

Inspected via `ps` or `top`/`htop`:

| State Flag | Name | Meaning |
| :---: | :--- | :--- |
| **`R`** | Running / Runnable | Actively executing instructions or waiting in the CPU run queue. |
| **`S`** | Interruptible Sleep | Blocked waiting for an event (I/O, timer, input). Can be awakened by signals. |
| **`D`** | Uninterruptible Sleep | Blocked waiting on hardware I/O (disk, hung network mount). **Cannot be killed, even with `SIGKILL`**, until hardware responds. |
| **`T`** | Stopped / Traced | Execution suspended via `Ctrl + Z` (`SIGTSTP`) or a debugger. |
| **`Z`** | Zombie / Defunct | Process has terminated execution, but its exit code has not yet been collected by its parent via `wait()`. |

---

## 4. Job Control & Terminal Detachment

```bash
# Suspend active foreground process
Ctrl + Z         # Sends SIGTSTP (state becomes Stopped)

# Inspect background jobs in current shell
jobs -l

# Resume job in background
bg %1

# Bring job back to interactive foreground
fg %1

# Launch detached from controlling terminal (immunized against SIGHUP)
nohup /path/to/script.sh > /var/log/app.log 2>&1 &
```

---

## 5. Process Discovery & Inspection Toolkit

```bash
# Process Tree inspection
ps -ef --forest
ps -fp <PID>

# Query PIDs by executable name
pgrep -a python3
pgrep -u deployer

# Send signals by name
pkill -f "python3 ~/zombie_maker.py"

# htop Power-User Controls
# t / F5  : Toggle hierarchical tree view
# \ / F4  : Filter processes by string
# P / M   : Sort table by CPU% or Memory%
# k / F9  : Send signal directly to highlighted PID
# q       : Exit
```
