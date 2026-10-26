# Troubleshooting: Diagnosing Hanging Network Connections & Layer 4 Packet Drops

- **Status:** Resolved & Documented
- **Category:** Linux Networking & Netfilter / UFW Firewall
- **Target Host:** Debian 13 VM (`192.168.122.142`)
- **Client Host:** EndeavourOS Workstation (`192.168.122.1`)

---

## 1. Problem Statement

A custom TCP service was initialized on the remote Debian server using Netcat to listen on port `9090`:

```bash
# On Debian Server:
nc -lvnp 9090
```

Attempting to connect to the socket from the workstation host resulted in a stalled terminal session:

```bash
# On EndeavourOS Workstation:
$ nc 192.168.122.142 9090
# Session hangs indefinitely with zero output or termination
```

---

## 2. Root Cause Analysis (RCA)

### Diagnostic Step 1: Verify Socket Binding State
Querying socket statistics on the target Debian VM:

```bash
$ ss -tlpn | grep 9090
LISTEN 0   1   0.0.0.0:9090   0.0.0.0:*   users:(("nc",pid=2853,fd=3))
```
*Finding:* The process was actively listening in state `LISTEN` on all IPv4 adapters (`0.0.0.0`).

### Diagnostic Step 2: Differentiating Connection Refusal vs. Packet Drop
- If a target port is closed or unassigned, the operating system kernel immediately generates and transmits a **TCP RST (Reset)** packet, causing the client to return `Connection refused` instantaneously.
- If the connection **hangs indefinitely**, the client's TCP `SYN` packets are being accepted by the wire but **silently discarded** by an intermediary or host firewall rule without emitting an ICMP unreachable or TCP RST response.

### Diagnostic Step 3: Inspecting Host Firewall Policies
Auditing Debian's Uncomplicated Firewall (UFW):

```bash
$ sudo ufw status
Status: active

To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW       Anywhere
OpenSSH (v6)               ALLOW       Anywhere (v6)
```
*Root Cause:* UFW enforced a default incoming policy of **DROP/DENY**. Only port 22 (SSH) possessed an explicit ingress allowance. Inbound SYN packets destined for port 9090 on `enp1s0` were silently dropped by Netfilter kernel chains.

---

## 3. Remediation

### Step 1: Apply Explicit Ingress Firewall Rule
Permit inbound TCP traffic targeting port 9090:
```bash
sudo ufw allow 9090/tcp
```

### Step 2: Connection Verification
Re-executing the connection from the EndeavourOS workstation:
```bash
$ nc 192.168.122.142 9090
```

Debian listener output:
```text
Connection received on 192.168.122.1 39556
```

### Step 3: Socket State Transition
Auditing the active socket state on Debian:
```bash
$ ss -tp | grep 9090
ESTAB 0 0 192.168.122.142:9090 192.168.122.1:39556 users:(("nc",pid=2853,fd=3))
```
The socket successfully completed the 3-Way Handshake and transitioned from `LISTEN` to `ESTAB`.

---

## 4. Security Hardening / Cleanup
After test validation, the temporary debug rule was pruned to maintain minimal attack surface:
```bash
sudo ufw delete allow 9090/tcp
```
