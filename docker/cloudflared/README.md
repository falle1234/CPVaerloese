# Cloudflare Tunnel (works behind CGNAT)

If your ISP uses CGNAT, your router has no public IP to forward ports to —
port forwarding (and `ddclient`, which only updates a public-IP A record)
can't work at all for you. A Cloudflare Tunnel avoids the problem entirely:
`cloudflared` makes an *outbound* connection from the Pi to Cloudflare's
edge, so no inbound ports or public IP are needed.

This uses a **locally-managed tunnel** (a `config.yml` read directly by
`cloudflared`) rather than the dashboard-managed kind, so origin settings
like `originServerName` (needed for end-to-end TLS to Nginx Proxy Manager)
are plain, versionable YAML instead of hunting through a dashboard UI that
keeps changing shape.

## One-time setup (needs a browser — run on your own machine, not the Pi)

1. Install `cloudflared` locally (e.g. `brew install cloudflared`, or see
   [Cloudflare's install docs](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads/)).

2. Authenticate against your Cloudflare account:
   ```
   cloudflared tunnel login
   ```
   This opens a browser, has you pick the zone (`yourdomain.dk`), and saves
   a cert to `~/.cloudflared/cert.pem`.

3. Create the tunnel:
   ```
   cloudflared tunnel create raspberry-pi
   ```
   This prints a **Tunnel ID** and writes a credentials file to
   `~/.cloudflared/<TUNNEL_ID>.json`.

4. Point DNS at the tunnel:
   ```
   cloudflared tunnel route dns raspberry-pi git.yourdomain.dk
   ```
   This creates the CNAME record automatically — no manual DNS record or
   `ddclient` needed for this hostname.

5. Copy the credentials file to the Pi, into this folder's `config/`
   directory (it's gitignored — never commit it):
   ```
   scp ~/.cloudflared/<TUNNEL_ID>.json pi@<pi-ip>:~/CPVaerloese/docker/cloudflared/config/
   ```

## Configure and run (on the Pi)

1. Copy the example config:
   ```
   cp config/config.yml.example config/config.yml
   ```

2. Edit `config/config.yml`:
   - `tunnel` → the Tunnel ID from step 3 above
   - `credentials-file` → `/etc/cloudflared/<TUNNEL_ID>.json` (matching the
     file you copied over)
   - `hostname` / `originServerName` → your actual domain

3. Create the shared network if it doesn't exist yet, and start the tunnel:
   ```
   docker network create proxy   # skip if it already exists
   docker compose up -d
   ```

4. Check it connected:
   ```
   docker compose logs -f cloudflared
   ```

This config uses `https://npm:443` with `originServerName` set, so
`cloudflared` sends the correct SNI and NPM serves its real Let's Encrypt
certificate — traffic stays encrypted all the way from the visitor's
browser to NPM, not just to Cloudflare's edge. In the Cloudflare dashboard
under **SSL/TLS**, set the encryption mode to **Full (strict)** to enforce
this end-to-end.

If you'd rather skip TLS on this internal hop entirely (it's already inside
the Pi's private Docker network), change the ingress rule to
`service: http://npm:80` and drop the `originRequest` block — simpler, and
still fine security-wise for a home setup.

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
