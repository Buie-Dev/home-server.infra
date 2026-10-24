![CI](https://github.com/Buie-Dev/home-server.infra/actions/workflows/ci.yml/badge.svg?branch=dev)

# home-server.infra

Docker Compose stack that runs the shared infrastructure for my home server services.

## Why I built it

I wanted a single place that manages all the infrastructure on my Linux server. Every other service in my home server project connects to the databases, MQTT broker, and network defined here. It also runs a Cloudflare Tunnel so the mobile app and web app work when I'm away from WiFi.

## How it fits

This repo must start first. It creates the `home-server` Docker network and runs the core services that everything else depends on.

- [home-server.api](https://github.com/Buie-Dev/home-server.api) is the Flask REST API that reads and writes to MySQL
- [homewebsocket.api](https://github.com/Buie-Dev/homewebsocket.api) is the Python WebSocket server that bridges WebSocket connections to MQTT
- [home-server.web](https://github.com/Buie-Dev/home-server.web) is the React web app, served by nginx in this stack
- [home-server.mobile](https://github.com/Buie-Dev/home-server.mobile) is the React Native mobile app

## Stack

- Docker Compose
- MySQL 8 (port 3307)
- MongoDB (port 27017)
- Eclipse Mosquitto MQTT broker (port 1883)
- Node-RED (port 1880)
- nginx reverse proxy (port 8080)
- Cloudflare Tunnel (cloudflared)

## Running it locally

1. Clone the repo

```bash
git clone https://github.com/Buie-Dev/home-server.infra.git
cd home-server.infra
```

2. Create the `.env` file and add your Cloudflare Tunnel token

```bash
cp .env.example .env
```

3. Create the Docker secrets files

```bash
mkdir -p secrets
echo "your_password" > secrets/mysql_root_password.txt
echo "your_password" > secrets/mysql_user_password.txt
echo "your_username" > secrets/mongo_root_username.txt
echo "your_password" > secrets/mongo_root_password.txt
```

4. Start all services

```bash
docker compose up -d
```

## Configuration

| Variable / File | Purpose |
|---|---|
| `CLOUDFLARE_TUNNEL_TOKEN` (.env) | Authenticates the Cloudflare Tunnel |
| `secrets/mysql_root_password.txt` | MySQL root password |
| `secrets/mysql_user_password.txt` | MySQL app user password |
| `secrets/mongo_root_username.txt` | MongoDB root username |
| `secrets/mongo_root_password.txt` | MongoDB root password |

## Services

| Container | Image | Host Port | Purpose |
|---|---|---|---|
| `mysqldb` | mysql | 3307 | Relational data for schedules, zones, history |
| `mongodb` | mongo | 27017 | Document storage |
| `mosquitto` | eclipse-mosquitto | 1883, 9001 | MQTT broker used by all services |
| `node-red` | nodered/node-red | 1880 | Flow-based automation UI |
| `nginx-gateway` | nginx | 8080 | Reverse proxy that routes HTTP and WebSocket traffic |
| `cloudflared` | cloudflare/cloudflared | none | Exposes nginx to the internet via Cloudflare Tunnel |

## Tests

There is no application code to test. CI validates the Docker Compose file and nginx config on every push.

```bash
docker compose config --quiet   # Validate docker-compose.yaml
docker exec nginx-gateway nginx -t   # Validate nginx config
```

## Architecture

nginx acts as the single entry point for all traffic. It routes `/api` to the Flask REST API on port 5001 and `/ws` to the WebSocket server on port 8765. Both backend services run in separate Docker Compose projects and are reached via `host.docker.internal`.

Cloudflare Tunnel runs alongside nginx and maintains a persistent outbound connection to Cloudflare's edge. Traffic from the custom domain arrives at Cloudflare, travels down the tunnel, and reaches nginx without any router port forwarding or static IP.

```
Mobile/Web App
    │
    ▼
Cloudflare Edge (TLS termination)
    │
    ▼ (tunnel)
cloudflared container
    │
    ▼
nginx-gateway (port 8080)
    ├── /api  → home-server.api (port 5001)
    ├── /ws   → homewebsocket.api (port 8765)
    └── /*    → web app static files
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
    MySQL       MongoDB     Mosquitto MQTT
    (3307)      (27017)     (1883)
                                │
                    ┌───────────┴───────────┐
                    ▼                       ▼
            irrigation-controller   garden-controller
            (Raspberry Pi)          (RPi Zero 2W)
```

## How I work on it

Changes start as specs in `docs/specs/`. Claude Code implements them on a branch and I review and merge every pull request.
