# ddclient (Dynamic DNS updates to Cloudflare)

Keeps a Cloudflare DNS A record pointed at the Pi's home public IP, since
residential connections typically don't have a static IP.

## Setup

1. In the Cloudflare dashboard, create a scoped API Token: **My Profile →
   API Tokens → Create Token**, with `Zone → DNS → Edit` permission limited
   to your zone (e.g. `yourdomain.dk`). Don't use the legacy Global API Key.

2. Copy the example config and fill in your real values:
   ```
   cp config/ddclient.conf.example config/ddclient.conf
   ```
   Edit `config/ddclient.conf`:
   - `zone` → your domain (e.g. `yourdomain.dk`)
   - `password` → the API token from step 1
   - the last line → the full record(s) to keep updated (e.g.
     `git.yourdomain.dk`); add more lines for additional subdomains

   `config/ddclient.conf` is gitignored since it contains the token — never
   commit it.

3. Start it:
   ```
   docker compose up -d
   ```

4. Check it's working:
   ```
   docker compose logs -f ddclient
   ```
   and confirm the A record's content/timestamp updates in the Cloudflare
   DNS dashboard.

## Notes

- `proxied=no` keeps the record "DNS only" (grey cloud) rather than proxied
  through Cloudflare's edge. This is required here: Cloudflare's proxy only
  forwards a fixed set of HTTP(S) ports and never proxies raw TCP like the
  git-SSH port (2222), so `git.yourdomain.dk` needs to resolve directly to
  the Pi's IP for SSH clones/pushes to work. NPM still handles TLS for the
  HTTP(S) side.
- `daemon=300` polls/updates at most every 5 minutes — fine for a home
  connection where the IP rarely changes.
- Only run one DDNS updater per record. If you ever add another (router
  built-in DDNS, etc.), disable one of them to avoid both racing to update
  the same record.
