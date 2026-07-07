# Raspberry Pi Docker host

Services running on the Pi, and how they fit together.

```
Internet
   |
   |  (Path A: normal public IP)         (Path B: CGNAT)
   |  router port-forward 80/443          cloudflared (outbound-only)
   v                                              v
                    Nginx Proxy Manager (docker/proxy)
                     - TLS termination
                     - routes by hostname
                                 |
                                 v
                       Forgejo (docker/forgejo)
                     - git web UI + git-over-SSH/HTTPS
```

| Folder | Purpose |
|---|---|
| [`proxy/`](proxy/README.md) | Nginx Proxy Manager — reverse proxy + Let's Encrypt TLS in front of Forgejo. Everything else routes through this. |
| [`forgejo/`](forgejo/README.md) | The git server itself (SQLite-backed). Joins the `proxy` network; no longer publishes port 3000 directly. |
| [`ddclient/`](ddclient/README.md) | **Path A only** (you have a normal public IP). Keeps a Cloudflare DNS record pointed at your home IP. |
| [`cloudflared/`](cloudflared/README.md) | **Path B only** (ISP uses CGNAT, no public IP available). Cloudflare Tunnel — outbound-only connection, no port forwarding needed. Replaces `ddclient` and router port-forwarding. |

Use Path A **or** Path B, not both — check whether your ISP gives you a
real public IP or CGNAT (compare your router's WAN IP to `curl -4 ifconfig.me`
run from inside your network; if they differ, you're behind CGNAT).

## Setup order

1. Create the shared Docker network once (both `proxy` and `forgejo` attach
   to it):
   ```
   docker network create proxy
   ```

2. Start the reverse proxy:
   ```
   cd proxy && docker compose up -d
   ```

3. Start Forgejo:
   ```
   cd ../forgejo && docker compose up -d
   ```

4. In the NPM admin UI (`http://<pi-ip>:81`), add a Proxy Host for your
   domain forwarding to `forgejo:3000`, and request a certificate:
   - HTTP-01 (needs port 80 reachable) if you're on Path A, or
   - DNS Challenge via Cloudflare if you're on Path B, or just prefer not to
     expose port 80 at all

5. Get traffic to the Pi, depending on your path:
   - **Path A**: `cd ../ddclient && docker compose up -d`, then forward
     ports 80/443 (and 2222 for git-over-SSH) on your router to the Pi.
   - **Path B**: `cd ../cloudflared && docker compose up -d`, then set up
     the tunnel's Public Hostname pointing at `npm:80` in the Cloudflare
     dashboard. No router changes needed/possible. Use git-over-HTTPS
     instead of SSH remotes (see `cloudflared/README.md`).

Each folder's own README has the full details for that service.
