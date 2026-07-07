# Nginx Proxy Manager (reverse proxy in front of Forgejo)

"NPM" here is [Nginx Proxy Manager](https://nginxproxymanager.com/) — a
reverse proxy with a web UI for managing proxy hosts and free Let's Encrypt
certificates. This puts it in front of Forgejo's web UI so Forgejo itself no
longer needs to publish port 3000 to the host.

Note: this only fronts **HTTP(S)**. Git-over-SSH (port 2222 on the Pi) is a
raw TCP protocol, not HTTP, so it can't be reverse-proxied here — clients
keep connecting to the Pi directly on that port for `git clone`/`push` over
SSH.

## Setup

1. Create the shared Docker network once (both this stack and the Forgejo
   stack attach to it):
   ```
   docker network create proxy
   ```

2. Start Nginx Proxy Manager:
   ```
   cd docker/proxy
   docker compose up -d
   ```

3. Log in to the admin UI at `http://<pi-ip>:81` with the NPM default
   credentials (`admin@example.com` / `changeme`) and **change them
   immediately**.

4. Add a Proxy Host:
   - Domain name: the domain/hostname you want Forgejo reachable at (must
     match `FORGEJO_DOMAIN` in `docker/forgejo/.env`)
   - Scheme: `http`
   - Forward hostname/IP: `forgejo` (the container name — reachable because
     both containers share the `proxy` network)
   - Forward port: `3000`
   - Enable "Block Common Exploits" and "Websockets Support"
   - On the SSL tab, request a Let's Encrypt certificate and enable "Force
     SSL" (needs port 80/443 reachable from the internet for the HTTP-01
     challenge if using a public domain; for LAN-only use, use NPM's
     self-signed certificate option instead)

5. Update `docker/forgejo/.env` so `FORGEJO_DOMAIN` matches the domain you
   configured in NPM, then recreate Forgejo so it joins the `proxy` network
   and drops its direct port-3000 publish:
   ```
   cd docker/forgejo
   docker compose up -d
   ```

   If Forgejo was already installed with the old domain baked into
   `data/gitea/conf/app.ini`, editing `.env` alone won't change it — update
   the `DOMAIN`, `ROOT_URL`, and `SSH_DOMAIN` values in that file directly
   and restart the container.

## Start-up order

The `proxy` network must exist before either stack starts. If you bring the
whole host up from scratch, run `docker network create proxy` first, then
start the `proxy` and `forgejo` stacks in either order.
