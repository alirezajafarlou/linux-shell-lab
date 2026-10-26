# Computer Networking Fundamentals for DevOps

An operational, terminal-first reference for Layer 2/3/4/7 packet traversal, ARP address resolution, TCP/UDP socket lifecycles, DNS lookups, and HTTP/HTTPS execution on Debian 13.

---

## 1. Network Addressing & Packet Traversal

Every networked host maintains two distinct identity layers:

| Layer | Identifier | Scope | Resolution Protocol | Forwarding Device |
| :--- | :--- | :--- | :--- | :--- |
| **Layer 2 (Data Link)** | **MAC Address** | Local broadcast domain (LAN only). Cannot cross routers. | **ARP (Address Resolution Protocol)** | **Switch** |
| **Layer 3 (Network)** | **IP Address** | Logical global routing across interconnected networks. | **Routing Table / Default Gateway** | **Router** |

### Address Resolution Protocol (ARP)
When a host transmits an IP datagram to a local neighbor:
1. It queries its kernel ARP cache (`ip neigh`).
2. If absent, it broadcasts an ARP request (`Who has 192.168.122.142? Tell 192.168.122.1`).
3. The target node replies with its hardware MAC address (`lladdr`).
4. Packets are encapsulated into Layer 2 Ethernet frames.

---

## 2. Kernel Routing & The Default Gateway

The Linux kernel inspects the routing table (`ip route`) sequentially:
- Packets targeted at local subnets (e.g., `192.168.122.0/24`) are emitted directly onto the local link (`dev enp1s0`).
- Unmatched non-local destinations match the `default via <GATEWAY_IP>` rule, handing the packet to the Layer 3 upstream router.

---

## 3. Transport Layer: TCP vs. UDP & Sockets

A **Socket** represents an endpoint for communication: `IP_ADDRESS : PORT`.

- **TCP:** Connection-oriented, guaranteed, ordered byte stream. Utilizes a **3-Way Handshake**:
  1. `Client ──► SYN ──► Server` (Sequence synchronization)
  2. `Server ──► SYN-ACK ──► Client` (Acknowledgment & server sequence)
  3. `Client ──► ACK ──► Server` (Connection state becomes `ESTABLISHED`)
- **UDP:** Connectionless, minimal overhead, fire-and-forget datagrams. Ideal for DNS queries and streaming.

### Binding Semantics:
- `127.0.0.1:PORT`: Loopback only. Inaccessible from external network adapters.
- `0.0.0.0:PORT`: Binds to all available IPv4 host interfaces (local, LAN, WAN).

---

## 4. Domain Name System (DNS) Architecture

DNS recursively resolves human-readable FQDNs into routable IP addresses:
```text
Root Nameserver (.) ──► TLD Nameserver (.com) ──► Authoritative Nameserver
```

### Key Record Types:
- **`A`**: IPv4 mapping (`example.com ──► 93.184.216.34`).
- **`AAAA`**: IPv6 mapping.
- **`CNAME`**: Canonical alias to another domain name.
- **`MX`**: Mail exchanger routing.
- **`TXT`**: Arbitrary text attributes (domain verification, SPF/DKIM).
- **`TTL` (Time To Live):** Resolver cache duration in seconds.

---

## 5. Application Layer: HTTP/HTTPS & Request Lifecycle

A standard HTTPS request (`curl -v https://example.com`) executes five sequential phases:
1. **DNS Lookup:** Resolves hostname to A/AAAA records.
2. **TCP Handshake:** Establishes Layer 4 connection on port 443.
3. **TLS Handshake:** Negotiates cipher suite, verifies CA certificates (`/etc/ssl/certs/`), and establishes symmetric session encryption.
4. **HTTP Request:** Client transmits HTTP method (`GET`), request headers, and payload.
5. **HTTP Response:** Server returns a 3-digit status code, response headers, and payload body.

### Common HTTP Status Codes in DevOps:
- `200 OK` / `201 Created`: Request succeeded.
- `301 Moved Permanently`: Canonical redirection.
- `400 Bad Request` / `401 Unauthorized` / `403 Forbidden` / `404 Not Found`: Client-side error.
- `500 Internal Server Error`: Application runtime panic.
- `502 Bad Gateway`: Reverse proxy (Nginx) cannot communicate with upstream application socket.
- `504 Gateway Timeout`: Upstream service failed to respond within timeout window.
