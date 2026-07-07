# Cloudflare Tunnel (works behind CGNAT)

If your ISP uses CGNAT, your router has no public IP to forward ports to —
port forwarding (and `ddclient`, which only updates a public-IP A record)
can't work at all for you. A Cloudflare Tunnel avoids the problem entirely:
`cloudflared` makes an *outbound* connection from the Pi to Cloudflare's
edge, so no inbound ports or public IP are needed.

## Setup

1. In the [Cloudflare Zero Trust dashboard](https://one.dash.cloudflare.com/)
   → **Networks → Tunnels → Create a tunnel** → choose **Cloudflared** →
   name it (e.g. `raspberry-pi`).

2. On the install-command step, it shows something like:
   ```
   docker run cloudflare/cloudflared:latest tunnel --no-autoupdate run --token eyJhIjoi...
   ```
   Copy just the token value (after `--token`) into `.env`:
   ```
   cp .env.example .env
   # paste the token into TUNNEL_TOKEN=
   ```

3. Still in the tunnel setup wizard, go to **Public Hostname** and add:
   - Subdomain/domain: `git.yourdomain.dk`
   - Service type: `HTTP`
   - URL: `npm:80`

   `npm:80` works because `cloudflared` joins the same `proxy` Docker
   network as Nginx Proxy Manager and reaches it by container name.
   Cloudflare automatically creates the DNS record for this hostname (a
   CNAME to the tunnel) — no manual DNS record or `ddclient` needed for it.

4. Create the shared network if it doesn't exist yet, and start the tunnel:
   ```
   docker network create proxy   # skip if it already exists
   docker compose up -d
   ```

5. In the Cloudflare dashboard → **SSL/TLS → Overview**, set the encryption
   mode to **Full (strict)**. This makes Cloudflare validate NPM's own
   Let's Encrypt certificate (from the DNS-01 setup in `docker/proxy/`) at
   the tunnel endpoint, so traffic stays encrypted all the way from the
   visitor's browser to NPM, not just to Cloudflare's edge.

6. Remove the port 80/443 forwarding rules on your router — they're no
   longer used (and wouldn't have worked under CGNAT anyway).

## Git-over-SSH no longer works the same way

CGNAT blocks *all* unsolicited inbound TCP, and a Cloudflare Tunnel doesn't
change that for arbitrary raw TCP like SSH — every client would need
`cloudflared` installed and configured too, which isn't realistic for a
normal `git clone`/`push` workflow.

The practical fix: use **git-over-HTTPS** instead of SSH remotes. Forgejo
supports this natively — clone with
`https://git.yourdomain.dk/<user>/<repo>.git` and authenticate with a
Forgejo access token (Settings → Applications → Generate New Token) instead
of an SSH key. HTTPS traffic rides through the tunnel exactly like the web
UI does.

If you still want a fallback path for LAN-only SSH access when you're on
the same network as the Pi, you can keep the existing port-2222 mapping in
`docker/forgejo/docker-compose.yml` — it'll just be unreachable from outside
the LAN, which is expected under CGNAT.

## What this replaces

- `docker/ddclient/` is no longer needed for `git.yourdomain.dk` — DNS now
  points at the tunnel, not your home IP. Keep it only if you still need a
  directly-resolvable A record for something else on your LAN.
