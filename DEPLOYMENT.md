# Bolt Deployment Guide

This guide explains how to deploy the Bolt app (bolt.diy) from this repository on the same VPS used for n8n and Supabase, and how to replicate it on another host or as an additional instance.

## Components and files
- Docker compose definitions:
  - `docker-compose.yaml` – dev and prod targets, publishes port 5173 to the host.
  - `docker-compose.prod.yml` – production profile that attaches to the shared `stack_net` network and expects a reverse proxy (e.g., Caddy) to front it.
- Dockerfile – builds production and development images.
- Env files: `.env` and `.env.local` (not committed) hold API keys and runtime config.

## Prerequisites
- Ubuntu/Debian host with Docker and docker compose plugin installed.
- External Docker network `stack_net` (shared with the Caddy/n8n/Supabase stack):
  ```bash
  docker network create stack_net
  ```
- DNS + TLS/reverse proxy:
  - Add a hostname in Cloudflare (e.g., `bolt.leveredgeai.com`) pointing to the VPS, set to **Proxied** (orange).
  - Add a Caddy site block to proxy to the Bolt container (see “Expose through Caddy” below).
- Secrets/API keys for the providers you intend to use (OpenAI, Anthropic, etc.) in `.env` / `.env.local`.

## Deploy (production) on this VPS
1. From `~/bolt`, create or update `.env` (and `.env.local` if needed):
   - Set any provider keys you need (e.g., `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GROQ_API_KEY`, etc.).
   - Set `DEFAULT_NUM_CTX` and other options as desired.
2. Bring up the production service using the production compose file:
   ```bash
   cd ~/bolt
   docker compose -f docker-compose.prod.yml up -d
   ```
   - This runs the `bolt` service, listening on port 5173 inside Docker, attached to `stack_net`.
3. Expose through Caddy (in `~/stack/Caddyfile`), add a site block such as:
   ```caddyfile
   bolt.leveredgeai.com {
       encode gzip
       reverse_proxy bolt-diy:5173
   }
   ```
   Then reload Caddy:
   ```bash
   cd ~/stack
   docker exec caddy caddy reload --config /etc/caddy/Caddyfile
   ```
4. Verify:
   ```bash
   docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
   ```
   Visit `https://bolt.leveredgeai.com` (behind Cloudflare). Apply Cloudflare Access if you want to restrict the UI.

## Deploy on a new server
1. Install Docker and docker compose plugin.
2. Create the external network:
   ```bash
   docker network create stack_net
   ```
3. Copy or clone the repo to `~/bolt`.
4. Populate `.env` / `.env.local` with your API keys and runtime settings.
5. Run the production compose:
   ```bash
   docker compose -f docker-compose.prod.yml up -d
   ```
6. Configure your reverse proxy (e.g., Caddy) and DNS as above. If you use Caddy, mount `~/bolt` service into your `Caddyfile` with the appropriate hostname and reload Caddy.

## Running development or prebuilt profiles (optional)
- Development (live reload, binds source):
  ```bash
  docker compose -f docker-compose.yaml --profile development up
  ```
- Prebuilt image (no local build):
  ```bash
  docker compose -f docker-compose.yaml --profile prebuilt up -d
  ```

## Multiple instances on the same host
- Use distinct hostnames (e.g., `bolt-dev.example.com`) and add matching proxied DNS records.
- Copy `docker-compose.prod.yml` to a new file (e.g., `docker-compose.devstack.yml`) and:
  - Change the service/container name to avoid collisions (e.g., `bolt-diy-dev`).
  - Optionally use a separate Docker network (e.g., `stack_net_dev`) or reuse `stack_net` intentionally.
  - Keep the internal port 5173 and adjust the Caddy site block to point to the new container name.

## Operational commands
- Start / restart (prod): `docker compose -f docker-compose.prod.yml up -d`
- Stop: `docker compose -f docker-compose.prod.yml down`
- Logs: `docker compose -f docker-compose.prod.yml logs -f`
- Rebuild image (if you modify source and want a fresh image): `docker compose -f docker-compose.prod.yml build --no-cache`
