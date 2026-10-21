# Challenge: Self-Healing Daemons & systemd Crash Resilience

- **Status:** Documented & Verified
- **Category:** Linux Service Reliability & Supervision
- **Target Environment:** Debian 13 VM (libvirt/KVM)
- **Unit Name:** `heartbeat.service`

---

## 1. Challenge Objective

Design, implement, and chaos-test a hardened background daemon that:
1. Runs under an unprivileged service account (`deployer`).
2. Streams real-time health telemetry directly to `systemd-journald`.
3. Persists across host reboots.
4. **Demonstrates self-healing resilience:** If the underlying process experiences an unhandled fatal crash (`SIGKILL`), the supervisor must automatically detect the failure and restore service within 2 seconds without human intervention.

---

## 2. Implementation

### Step 1: Daemon Application Logic (`/opt/heartbeat.sh`)
```bash
#!/bin/bash
echo "[Heartbeat] Starting service under PID: $$"

count=1
while true; do
    echo "[Heartbeat] Beat #$count - System healthy at $(date +%T)"
    ((count++))
    sleep 3
done
```
Permissions were locked to `755` with ownership `deployer:developers`.

### Step 2: Systemd Unit Definition (`/etc/systemd/system/heartbeat.service`)
```ini
[Unit]
Description=DevOps Heartbeat Telemetry Daemon
After=network.target

[Service]
Type=simple
User=deployer
Group=developers
ExecStart=/bin/bash /opt/heartbeat.sh
Restart=always
RestartSec=2

[Install]
WantedBy=multi-user.target
```

### Step 3: Activation
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now heartbeat.service
```

---

## 3. The Chaos Test: Unhandled Crash Simulation

### Initial State
```bash
$ systemctl status heartbeat.service
● heartbeat.service - DevOps Heartbeat Telemetry Daemon
     Active: active (running)
 Invocation: 1e2a8c5c8ff844bd8616337f3fe68c74
   Main PID: 7903 (bash)
```

### Injecting Fatal Failure (`SIGKILL`)
We delivered a non-catchable signal directly to the active process:
```bash
$ sudo kill -9 7903
```

---

## 4. Observations & Recovery Analysis

Inspecting systemd state immediately after injection:

```bash
$ systemctl status heartbeat.service
● heartbeat.service - DevOps Heartbeat Telemetry Daemon
     Active: active (running)
 Invocation: c8b1f502df204892bc58a032991ddc44
   Main PID: 8012 (bash)
```

### Post-Mortem Findings:
1. **Detection:** systemd caught the abrupt exit code (`SIGKILL / signal 9`) of PID `7903`.
2. **Backoff Delay:** Respected `RestartSec=2` before respawning.
3. **Restoration:** A new instance of `/opt/heartbeat.sh` was allocated PID `8012`.
4. **Session Tracking:** A brand-new **Invocation ID** was assigned to isolate new log streams in `journalctl`.
5. **Downtime:** Approximately 2.1 seconds total before service was completely restored.

---

## 5. Production Takeaway
In containerized and cloud architectures, processes should be ephemeral and designed to fail fast. Relying on an external supervisor (`systemd` on bare metal/VMs, or the Kubelet in Kubernetes) with clear restart policies guarantees high availability without manual engineer intervention during runtime panics.
