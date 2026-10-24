# home-server

Docker Compose stack for shared home server infrastructure. Start this repo first — it defines the `home-server` Docker network and core services that all other repos join.

---

## Table of Contents

- [Quick Start](#quick-start)
- [Services](#services)
- [Definitions](#definitions)
- [nginx / Cloudflare Architecture](#nginx--cloudflare-architecture)
  - [The Problem It Solves](#the-problem-it-solves)
  - [Full Traffic Flow Diagram](#full-traffic-flow-diagram)
  - [Component Deep Dive](#component-deep-dive)
  - [Why These Design Decisions?](#why-these-design-decisions)
- [Configuration](#configuration)
- [Troubleshooting](#troubleshooting)

---

## Quick Start

```bash
cp .env.example .env    # fill in CLOUDFLARE_TUNNEL_TOKEN
docker compose up -d
```

---

## Services

| Container | Image | Host Port | Purpose |
|---|---|---|---|
| `mysqldb` | mysql | 3307 | Relational data (schedules, zones, history) |
| `mongodb` | mongo | 27017 | Document storage |
| `mosquitto` | eclipse-mosquitto | 1883, 9001 | MQTT broker |
| `node-red` | nodered/node-red | 1880 | Flow-based automation UI |
| `nginx-gateway` | nginx | 8080 | Reverse proxy — routes HTTP + WebSocket |
| `cloudflared` | cloudflare/cloudflared | — | Exposes nginx to the internet via Cloudflare Tunnel (example.com) |

---

## Definitions

Before reading the architecture section, here are plain-English definitions for every concept used.

### Docker & Containers

**Docker** is a tool that lets you run applications inside isolated boxes called **containers**. A container packages an application and everything it needs to run (code, libraries, config) so it behaves the same on any machine.

**Docker Compose** is a tool for defining and running multiple containers together using a single `docker-compose.yaml` file. Instead of starting each container manually, you run `docker compose up` and everything starts together.

**Image** is the template a container is built from (e.g. `nginx:latest` is the official nginx image). A container is a running instance of an image.

### Docker Networking

**Docker network** is a private virtual network that containers can talk to each other on. Containers on the same network can reach each other by name (e.g. `nginx-gateway` can talk to `mosquitto` just by using that name as a hostname).

**Bridge network** (default) — Docker creates a virtual network switch. Containers on it can talk to each other and can reach the internet via NAT (Network Address Translation, like how your phone reaches the internet through your home router). However, they are isolated from the host machine's network by default.

**Host network mode** (`network_mode: host`) — The container shares the host machine's network stack directly. There is no isolation — the container sees the same network interfaces, IP addresses, and ports as the Linux machine itself. This is less isolated but allows the container to reach everything the host can reach.

**Port mapping** (e.g. `"8080:80"`) — By default, a bridge-network container's ports are not accessible from outside Docker. Port mapping punches a hole: `"8080:80"` means traffic arriving at port 8080 on the host machine is forwarded to port 80 inside the container.

**`host.docker.internal`** — A special hostname that bridge-network containers can use to reach the host machine itself (the Linux server). In this project, `extra_hosts: "host.docker.internal:host-gateway"` explicitly adds this mapping because it isn't automatic on Linux (it is on Mac/Windows).

### HTTP, HTTPS, and WebSocket

**HTTP** — The standard protocol for web requests. A client sends a request, a server sends a response, done. Stateless.

**HTTPS** — HTTP encrypted with TLS. Traffic is scrambled so it can't be read in transit. Required for requests coming from mobile apps over the internet.

**WebSocket** — A protocol that starts as HTTP but then "upgrades" to a persistent two-way connection. The server can push data to the client at any time without the client asking first. Used here so the app receives live irrigation state updates instantly.

**REST API** — An API that uses standard HTTP methods (GET, POST, PUT, DELETE) to perform actions. Used here for schedules, zones, history.

### Reverse Proxy (nginx)

A **proxy** sits between a client and a server, forwarding requests on behalf of the client.

A **reverse proxy** sits in front of one or more servers and receives all incoming requests. It decides which server to forward each request to based on rules (e.g. the URL path). The client only knows about the proxy — not the individual servers behind it.

**nginx** (pronounced "engine-x") is a high-performance web server commonly used as a reverse proxy. In this project it acts as a traffic router:
- Requests to `/ws` → WebSocket server
- All other requests → Flask REST API

Benefits of using nginx:
- Single entry point for all traffic (one port, one host)
- Handles WebSocket upgrade headers automatically
- Easy to add more services behind it later without changing the app

### Tunneling (Cloudflare)

The home server sits on a private home network. Its IP address is not reachable from the internet — your router's firewall blocks all unsolicited inbound traffic, and your ISP assigns a dynamic public IP that changes.

**Cloudflare Tunnel** solves this by creating a **tunnel**: a persistent outbound connection from your server to Cloudflare's edge. Since the connection is outbound (your server calls out to Cloudflare, not the other way around), it passes through the firewall. Cloudflare then accepts inbound traffic on your custom domain and forwards it down the tunnel to your server.

**Custom domain** — `example.com` is a permanent domain managed via Cloudflare DNS. The mobile app is hardcoded to this address.

### DNS

**DNS (Domain Name System)** translates human-readable domain names into IP addresses that computers use to connect to each other.

**DNS server** — The server that does the translation. Your home router (`192.168.x.x`) acts as a local DNS server and forwards requests to your ISP's DNS upstream.

---

## nginx / Cloudflare Architecture

### The Problem It Solves

The mobile app needs to talk to two services on the home server:
1. **homeserver.api** — Flask REST API on port 5001
2. **homewebsocket.api** — WebSocket server on port 8765

When the phone is at home on WiFi, it can reach these directly by IP. When it's away from home, the server is behind a firewall and has no stable public IP.

The solution:

```
nginx       — single entry point, routes /ws to WebSocket and /* to REST
cloudflared — punches through the firewall, exposes nginx at example.com
```

---

### Full Traffic Flow Diagram

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  MOBILE APP  (iPhone / Android)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  HomeApp reads REACT_APP_DOMAIN from .env

  If set:
    REST calls  → https://example.com/api
    WebSocket   → wss://example.com/ws

  If NOT set (LAN only):
    REST calls  → http://192.168.x.x:5001/
    WebSocket   → ws://192.168.x.x:8765


         │  HTTPS request  (e.g. GET /irrigations/zones)
         │  or
         │  WebSocket upgrade  (wss://.../ws)
         │
         ▼

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  CLOUDFLARE EDGE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Receives traffic at example.com.
  Forwards it down the persistent tunnel to the cloudflared agent
  running on the home server.

  Public URL:   https://example.com
  Terminates TLS here — traffic inside the tunnel is plain HTTP.

         │  HTTP (unencrypted, inside encrypted tunnel)
         │
         ▼

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  HOME SERVER  (home-server,192.168.x.x)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌──────────────────────────────────────────────────────────────┐
  │  cloudflared container                   [home-server net]  │
  │                                                              │
  │  • Maintains outbound tunnel to Cloudflare edge             │
  │  • Receives forwarded traffic from Cloudflare               │
  │  • Forwards it to nginx on the home-server network          │
  └───────────────────────┬──────────────────────────────────────┘
                          │  HTTP  →  localhost:8080
                          │
                          ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  nginx-gateway container                [home-server network]│
  │  Container port 80 → Host port 8080                          │
  │  Config: ./nginx/nginx.conf                                  │
  │                                                              │
  │  Routing rules:                                              │
  │                                                              │
  │   PATH       UPSTREAM                    PROTOCOL           │
  │   ────────   ────────────────────────    ────────────────   │
  │   /ws        host.docker.internal:8765   WebSocket          │
  │   /api       host.docker.internal:5001   HTTP (REST)        │
  │   /*         (reserved for React site)   —                  │
  │                                                              │
  │  host.docker.internal = host gateway IP                     │
  │  (added via extra_hosts in docker-compose.yaml)             │
  └────────────┬──────────────────────┬───────────────────────  ┘
               │                      │
               │  HTTP :5001          │  WebSocket :8765
               │                      │
               ▼                      ▼
  ┌────────────────────┐   ┌──────────────────────────────────┐
  │  homeserver.api    │   │  homewebsocket.api               │
  │  [home-server net] │   │  [home-server net]               │
  │  Flask REST API    │   │  Python WebSocket bridge         │
  │  port 5001         │   │  port 8765                       │
  │                    │   │                                  │
  │  GET /zones        │   │  Receives: request, stop,        │
  │  GET /schedules    │   │           stopAll, healthcheck   │
  │  GET /history      │   │  Publishes to MQTT broker        │
  │  etc.              │   │  Forwards MQTT state to clients  │
  └────────┬───────────┘   └───────────────┬──────────────────┘
           │                               │
           │  SQL queries                  │  MQTT pub/sub
           ▼                               ▼
  ┌──────────────────┐         ┌────────────────────────────┐
  │  mysqldb         │         │  mosquitto                 │
  │  MySQL :3307     │         │  MQTT broker :1883         │
  │  [home-server]   │         │  [home-server network]     │
  └──────────────────┘         └───────────────┬────────────┘
                                               │  MQTT
                                ───────────────┴───────────────
                                │                             │
                                ▼                             ▼
                    ┌───────────────────┐       ┌────────────────────┐
                    │ irrigation-       │       │ garden-controller  │
                    │ controller (RPi)  │       │ (RPi Zero 2W)      │
                    │ Queue management  │       │ GPIO relay control │
                    └───────────────────┘       └────────────────────┘
```

---

### Component Deep Dive

#### cloudflared container

- **Image:** `cloudflare/cloudflared:latest`
- **Network:** `home-server` bridge network
- **What it does:** Dials out to Cloudflare's edge and maintains a persistent tunnel. When a request arrives at `example.com`, Cloudflare sends it down the tunnel to this container, which forwards it to `nginx-gateway` on the `home-server` network.
- **Config (`.env`):**
  - `CLOUDFLARE_TUNNEL_TOKEN` — authenticates the tunnel with your Cloudflare account (from the Cloudflare Zero Trust dashboard)

#### nginx container (`nginx-gateway`)

- **Image:** `nginx:latest`
- **Network:** `home-server` bridge network
- **Ports:** `8080:80` — host port 8080 → container port 80
- **What it does:** Acts as a reverse proxy. All traffic (REST and WebSocket) arrives here first. nginx reads the request path and routes it to the right backend service.
- **Config (`nginx/nginx.conf`):**
  - `/ws` — WebSocket path. nginx adds the required `Upgrade` and `Connection` headers that tell the upstream server to switch protocols.
  - `/api` — REST API path. The `/api` prefix is stripped before the request is forwarded to Flask (e.g. `/api/irrigations/zones` → `/irrigations/zones`).
  - `/*` — Reserved for a future React site.
- **`host.docker.internal`** — nginx needs to reach `homeserver.api` and `homewebsocket.api`, which run as separate Docker services outside this compose file. They're not on the `home-server` network so they can't be reached by container name. Instead, nginx connects to the host machine's IP via the special hostname `host.docker.internal`, and from there reaches the services via their published host ports (5001 and 8765).

---

### Why These Design Decisions?

| Decision | Why |
|---|---|
| nginx as a reverse proxy | Single public-facing entry point. The app only needs one hostname/port instead of separate addresses for REST and WebSocket. Easier to add more services later. |
| Cloudflare Tunnel for internet access | The home server is behind a NAT firewall with a dynamic public IP. Cloudflare's outbound tunnel bypasses both without requiring router port-forwarding or a static IP. |
| Custom domain (`example.com`) | A permanent domain managed via Cloudflare DNS — the mobile app can be hardcoded to it. |
| `host.docker.internal` in nginx | `homeserver.api` and `homewebsocket.api` are separate Docker Compose projects. They join the `home-server` network but their service names aren't available to nginx at startup. Reaching them via host ports is simpler and works regardless of startup order. |

---

## Configuration

### `.env`

Required to start the stack:

```
CLOUDFLARE_TUNNEL_TOKEN=<from Cloudflare Zero Trust dashboard → Tunnels>
```

### `HomeApp/.env` (mobile app)

To route the app through Cloudflare (required when away from home):

```
REACT_APP_DOMAIN=example.com
```

When this is set, all REST and WebSocket traffic goes through Cloudflare. When it is commented out, the app connects directly to `192.168.x.x` on the local network.

### Docker secrets

Sensitive database passwords are stored as files (not in environment variables) and read by Docker at runtime:

```
secrets/mysql_root_password.txt
secrets/mysql_user_password.txt
secrets/mongo_root_username.txt
secrets/mongo_root_password.txt
```

---

## Troubleshooting

### Cloudflare tunnel not connecting

1. Check logs: `docker compose logs cloudflared -f`
2. Verify `.env` has `CLOUDFLARE_TUNNEL_TOKEN`
3. Check tunnel status in the Cloudflare Zero Trust dashboard

### nginx can't reach backend services

```bash
docker exec nginx-gateway curl -s http://host.docker.internal:5001/
docker exec nginx-gateway curl -s http://host.docker.internal:8765/
```

If these fail, check that `homeserver.api` and `homewebsocket.api` are running and their ports are published to the host.

### Inspect live traffic

Use the Cloudflare Zero Trust dashboard to inspect tunnel status and traffic metrics.
