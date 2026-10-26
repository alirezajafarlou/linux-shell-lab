# SSH Cryptography, Port Forwarding & Shell Environment Mechanics

An operational reference for asymmetric SSH handshakes, local port tunneling, `$PATH` binary resolution, and process environment inheritance on Debian 13.

---

## 1. SSH Cryptography & The Two-Way Handshake

SSH uses asymmetric cryptography to establish mutually authenticated, encrypted communication:

```text
Host (Client: EndeavourOS)                       Server (Debian 13 VM)
        │                                                   │
        ├───────── 1. Verify Server Authenticity ───────────► (Matches ~/.ssh/known_hosts)
        │          (Guards against Man-In-The-Middle)       │
        │                                                   │
        ◄───────── 2. Cryptographic Proof Challenge ────────┤ (Matches ~/.ssh/authorized_keys)
                   (Proves ownership of Private Key)        │
```

- **`~/.ssh/known_hosts` (Client-side):** Stores public host keys of remote servers. If a server's fingerprint changes unexpectedly, SSH aborts connection to protect against DNS hijacking or MITM attacks.
- **`~/.ssh/authorized_keys` (Server-side):** Stores the client's public keys permitted to log in. The private key never traverses the wire; the server encrypts a challenge that can only be decrypted by the matching private key.
- **Permissions Enforcement:** `sshd` strictly enforces permissions. Loose permissions (`chmod 777`) will trigger connection failures:
  - `~/.ssh/` directory: `700` (`drwx------`)
  - Private keys (`id_*`): `600` (`-rw-------`)
  - Public keys / `authorized_keys`: `644` (`-rw-r--r--`) or `600`

---

## 2. SSH Port Forwarding: The Trio (`-L`, `-R`, `-D`)

| Type | Flag | Listening Endpoint | Forwarding Direction | Production Use Case |
| :--- | :---: | :---: | :---: | :--- |
| **Local** | **`-L`** | Local Host | Local $\rightarrow$ Remote | Accessing isolated databases (PostgreSQL/Redis) or dashboards listening only on remote `127.0.0.1`. |
| **Remote** | **`-R`** | Remote Server | Remote $\rightarrow$ Local | Exposing a local development server or webhook receiver on your laptop to a remote network (reverse proxy / ngrok). |
| **Dynamic** | **`-D`** | Local Host | Dynamic (SOCKS5) | Routing arbitrary application or browser traffic through the remote host's egress IP. |

### Local Port Forwarding Syntax:
```bash
ssh -L <LOCAL_PORT>:<TARGET_IP_FROM_SERVER_VIEW>:<TARGET_PORT> <USER>@<SERVER>
```

---

## 3. Shell Environment Mechanics & Process Inheritance

### Local vs. Exported Variables
Variables defined in a shell session are private to that shell's execution space:

```bash
# Local Shell Variable (Not inherited by child processes):
DB_NAME="production"
bash -c 'echo $DB_NAME'       # Returns empty string!

# Environment Variable (Exported into process table):
export DB_NAME="production"
bash -c 'echo $DB_NAME'       # Outputs: production
```

> **DevOps Implication:** Tools spawned from the terminal (e.g., `curl`, `git`, `docker`, build scripts) are child processes. System configurations, API tokens, and proxy settings (`http_proxy`) must be `export`ed to be inherited by these CLI tools.

---

## 4. `$PATH` Resolution Architecture

When a command is invoked without an explicit path (`/usr/bin/python3`), the shell sequentially scans the colon-separated directory list in `$PATH`:

```text
$PATH: /usr/local/bin : /usr/bin : /bin : /home/mr_alj/bin
            1               2       3            4
```

- The first executable matching the command name is executed.
- Custom user utilities placed in `~/bin/` require adding `$HOME/bin` to `$PATH`.

### Persistence Tiers:
- **`~/.bashrc`:** User-specific configuration for interactive, non-login shells.
- **`~/.profile` / `~/.bash_profile`:** User-specific login environment.
- **`/etc/environment`:** System-wide key-value environment variables loaded for all sessions.
