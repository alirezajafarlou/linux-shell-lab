# Docker Networking and Service Connectivity

## Objectives

- Understand basic Linux network diagnostics.
- Inspect Docker Compose network topology.
- Understand container DNS and service discovery.
- Distinguish container ports from published host ports.
- Diagnose connectivity problems layer by layer.

## 1. Linux Network Diagnostics

Linux provides tools for examining network configuration and testing connectivity.

| Command | Purpose |
|---|---|
| `ip -br addr` | Display network interfaces and IP addresses |
| `ip route` | Display the routing table |
| `ping <gateway>` | Test basic IP connectivity to the gateway |
| `getent hosts <hostname>` | Test hostname resolution |
| `curl -i <URL>` | Inspect an HTTP response, including headers |
| `ss -lnt` | Display listening TCP sockets |

A successful test establishes that the tested operation worked at that moment. It does not automatically establish end-to-end connectivity for every application.

## 2. Docker Compose Network Topology

The `health-api` project runs two services:

- `api`: Node.js application.
- `postgres`: PostgreSQL database.

The Compose configuration defines two networks:

| Network | API | PostgreSQL |
|---|---|---|
| `health-api_backend` | Connected | Connected |
| `health-api_egress` | Connected | Not connected |

The backend network is configured as internal, providing an isolated network for application-to-database communication. The egress network gives the API a route for outbound connectivity.

### Inspecting networks

List Docker networks:

```bash
docker network ls
```

Inspect a container's network membership:

```bash
docker inspect -f '{{json .NetworkSettings.Networks}}' health-api
docker inspect -f '{{json .NetworkSettings.Networks}}' health-api-db
```

The inspected containers shared `health-api_backend`.

Their observed IP addresses were:

- API: `172.18.0.3`
- PostgreSQL: `172.18.0.2`

These addresses are runtime details and may change when containers are recreated.

## 3. Container DNS and Service Discovery

The API connects to PostgreSQL using the hostname `postgres` and port `5432`.

```text
postgres:5432
```

Docker Compose provides DNS-based service discovery on shared networks. The hostname resolves to the database container's current network address.

This avoids hardcoding a container IP address into the application configuration.

If the database container is recreated and its IP changes, the service name remains stable. The application can resolve the name again when establishing a new connection.

DNS resolution does not guarantee that the database is ready or that existing connections will recover automatically.

## 4. Container Ports and Host Port Publishing

The API publishes port `3000`:

```yaml
ports:
  - "3000:3000"
```

This maps host port `3000` to container port `3000`.

PostgreSQL has no `ports` mapping in the Compose configuration. Its port `5432` is therefore not published on the host.

However, the API can still reach PostgreSQL through the shared backend network.

### Inspecting published ports

```bash
docker compose port api 3000
docker compose port postgres 5432
```

Observed results:

- API: `0.0.0.0:3000`
- PostgreSQL: no host port published

The API was also visible on the host's IPv6 wildcard address in the `ss -lnt` output.

`0.0.0.0` means all IPv4 interfaces on the host. Actual reachability depends on routing, firewall rules, and other network controls.

### Listening sockets

```bash
ss -lnt
```

This displays listening TCP sockets on the host. Docker may implement published ports using NAT rules rather than a conventional listening socket, so this command alone does not fully describe Docker's forwarding behavior.

## 5. API Health and HTTP Responses

An API exposes an interface that clients use to request operations and receive responses.

For example:

```bash
curl -i http://localhost:3000/live
curl -i http://localhost:3000/ready
```

The `/live` endpoint returned HTTP `200 OK` with:

```json
{"status":"alive"}
```

The `/ready` endpoint also returned HTTP `200 OK` during the test.

Liveness and readiness serve different purposes:

- **Liveness:** indicates whether the application is alive according to its configured check.
- **Readiness:** indicates whether the application considers itself ready to serve requests.

A successful response confirms that the tested endpoint responded successfully at that time.

## 6. Layered Connectivity Troubleshooting

When a service is unreachable, investigate the relevant layers in order:

1. **Process:** Is the application running?
2. **Health:** Is the container's configured health check passing?
3. **Listener:** Is the application listening on the expected port?
4. **DNS:** Does the hostname resolve correctly?
5. **Network:** Are the required containers connected to a shared network?
6. **Publishing and routing:** Is the required host port published, and can traffic reach the host?
7. **Firewall:** Are host or network rules blocking the connection?
8. **Application:** Does the endpoint handle the request successfully?
9. **Dependencies:** Can the application reach PostgreSQL or other required services?

Choose tests that correspond to the suspected failure. A container can be healthy while a client is unable to reach its service.

## Key Takeaways

- Docker Compose creates networks that control container connectivity.
- Containers on a shared Compose network can discover one another by service name.
- Container IP addresses can change; service names provide a stable discovery mechanism.
- Publishing a port makes a container service accessible through a host port, subject to network controls.
- A container does not need a published host port to communicate with another container on a shared network.
- Health checks, listening sockets, DNS lookups, and HTTP requests test different aspects of a system.
- Troubleshooting should identify the failing layer instead of assuming that one successful check proves everything works.

