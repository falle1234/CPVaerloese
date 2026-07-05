# Forgejo on the Raspberry Pi Docker host

Self-hosted Git server, using the official multi-arch Forgejo image (works on
the Pi 3 B+'s arm64 architecture) with SQLite so no separate database
container is needed.

## Setup

1. Copy the env template and adjust it:
   ```
   cp .env.example .env
   ```
   Set `FORGEJO_DOMAIN` to the Pi's hostname/IP or a real domain if you're
   putting a reverse proxy in front of it.

2. Put this folder's `data/` directory on the USB SSD (not the SD card) if
   you followed the Docker host setup guide — Forgejo does frequent small
   writes (git objects, SQLite, indexes).

3. Start it:
   ```
   docker compose up -d
   ```

4. Finish setup in the browser at `http://<pi-ip-or-domain>:3000`. The
   install page pre-fills sane defaults from the environment variables above
   (SQLite, domain, ports) — just create the initial admin account.

5. Clone/push over SSH using the mapped port, e.g.:
   ```
   git clone ssh://git@<pi-ip-or-domain>:2222/<user>/<repo>.git
   ```

## Notes

- Port 2222 (host) → 22 (container) is used for git-over-SSH so it doesn't
  clash with the Pi's own sshd on port 22.
- Back up the `data/` directory — it contains the SQLite database, repos,
  and uploaded content.
- If you expose this beyond your LAN, put it behind a reverse proxy (Caddy /
  nginx / Traefik) for TLS rather than exposing port 3000 directly.
