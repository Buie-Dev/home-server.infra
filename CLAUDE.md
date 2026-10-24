# CLAUDE.md

## What this repo is

Docker Compose stack that runs the shared infrastructure for my home server. This is the central piece that the API and WebSocket services depend on for networking, databases, and the MQTT broker. It also provides a Cloudflare Tunnel so the mobile app and web app work away from WiFi.

## Related repos

- [home-server.api](https://github.com/Buie-Dev/home-server.api) — Flask REST API, joins the `home-server` Docker network
- [homewebsocket.api](https://github.com/Buie-Dev/homewebsocket.api) — Python WebSocket server, bridges WebSocket to MQTT
- [home-server.web](https://github.com/Buie-Dev/home-server.web) — React web app
- [home-server.mobile](https://github.com/Buie-Dev/home-server.mobile) — React Native mobile app

## Stack

- Docker Compose
- MySQL 8 (port 3307)
- MongoDB (port 27017)
- Eclipse Mosquitto MQTT broker (port 1883)
- Node-RED (port 1880)
- nginx reverse proxy (port 8080)
- Cloudflare Tunnel (cloudflared)

## Commands

```bash
docker compose up -d       # Start all services
docker compose down        # Stop all services
docker compose logs -f     # Follow logs for all services
docker compose config      # Validate docker-compose.yaml
nginx -t                   # Validate nginx config (inside container)
```

To validate nginx config from outside the container:

```bash
docker exec nginx-gateway nginx -t
```

## Folder layout

```
docker-compose.yaml        # All services, networks, volumes, secrets
mosquitto/config/           # Mosquitto broker config
nginx/nginx.conf            # Reverse proxy routing rules
ngrok/resolv.conf           # Custom DNS resolver (legacy, from ngrok era)
secrets/                    # Docker secrets (gitignored)
.env                        # Cloudflare tunnel token (gitignored)
.env.example                # Template for .env
```

## How the network works

All services join a bridge network called `home-server`. Other repos (home-server.api, homewebsocket.api) join this network as external. nginx routes `/api` to the REST API on port 5001 and `/ws` to the WebSocket server on port 8765 via `host.docker.internal`.

## Standards

- Two long-lived branches: `main` (stable) and `dev` (ongoing work)
- Spec branches named `spec/NNN-short-name`, branched from `dev`, merged back to `dev`
- Commit messages: imperative, sentence case, no AI attribution
- Docker Compose repos validate with `docker compose config` in CI, plus `nginx -t`
- `.gitignore` covers `.env` and `secrets/`
- `.env.example` lists every variable with placeholder values
- No license file

## Boundaries

- Never push. Give commands and wait for confirmation.
- Never change service behavior unless a spec says to.
- Never commit `.env`, secrets, real domains, real IPs, or hostnames.
- Never merge a pull request. I review and merge.
- Only write things that are true in the code.

## Workflow

Changes start as specs in `docs/specs/`. Claude Code implements them on a spec branch, runs validation, and commits. I review and merge every pull request.
