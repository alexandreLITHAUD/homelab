# Gitea — Self-hosted Git on Mac Mini

Phase 1 of the homelab roadmap. Goal: get a working Gitea instance, reachable for
push/pull from your NixOS laptop, before moving on to the reverse proxy.

---

## Prerequisites

- Docker + Docker Compose installed on the Mac Mini (Phase 0 of the roadmap)
- Shared Docker network created: `docker network create homelab`

---

## Folder structure

```
~/homelab/gitea/
├── docker-compose.yml
├── .env
└── data/            # created automatically on first run
```

---

## .env

```env
GITEA_VERSION=1.22
GITEA_HTTP_PORT=3000
GITEA_SSH_PORT=2222
GITEA_DOMAIN=gitea.local
GITEA_ROOT_URL=http://gitea.local/
TZ=Europe/Paris
```

> `GITEA_SSH_PORT=2222`: we avoid port 22 to not conflict with the Mac Mini's own SSH daemon.

---

## docker-compose.yml

```yaml
services:
  gitea:
    image: gitea/gitea:${GITEA_VERSION}
    container_name: gitea
    environment:
      - USER_UID=1000
      - USER_GID=1000
      - GITEA__server__DOMAIN=${GITEA_DOMAIN}
      - GITEA__server__ROOT_URL=${GITEA_ROOT_URL}
      - GITEA__server__SSH_PORT=${GITEA_SSH_PORT}
      - TZ=${TZ}
    restart: unless-stopped
    volumes:
      - ./data:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "${GITEA_HTTP_PORT}:3000"
      - "${GITEA_SSH_PORT}:22"
    networks:
      - homelab

networks:
  homelab:
    external: true
```

> Database: built-in SQLite by default, more than enough for personal use. No need
> for a separate Postgres container for now — you can migrate later if you want to
> practice that kind of migration.

---

## Deployment checklist

- [ ] Copy this folder to the Mac Mini (`~/homelab/gitea/`)
- [ ] `docker compose up -d`
- [ ] Check the logs: `docker compose logs -f gitea`
- [ ] Open `http://<mac-mini-ip>:3000` in a browser
- [ ] Finish the install through the web wizard (database path, repo root, etc. are
      pre-filled thanks to the env vars — don't change anything unless you know why)
- [ ] Create the admin account
- [ ] Log in and check the Gitea dashboard

---

## SSH setup (for password-less push/pull)

On your **NixOS laptop**:

```bash
ssh-keygen -t ed25519 -C "laptop-nixos" -f ~/.ssh/gitea_ed25519
```

Add to `~/.ssh/config`:

```
Host gitea-home
    HostName <mac-mini-ip>
    Port 2222
    User git
    IdentityFile ~/.ssh/gitea_ed25519
```

Then in Gitea (web UI): **Settings → SSH/GPG Keys → Add Key**, paste the contents
of `~/.ssh/gitea_ed25519.pub`.

Test:

```bash
ssh -T gitea-home
# should reply: "Hi there, <username>! You've successfully authenticated..."
```

---

## Create the first repo (the dashboard project)

- [ ] In Gitea: **New Repository** → name `dashboard`, private visibility
- [ ] Locally:

```bash
mkdir dashboard && cd dashboard
git init
echo "# Dashboard" > README.md
git add README.md
git commit -m "init"
git remote add origin gitea-home:username/dashboard.git
git push -u origin main
```

---

## End-of-phase criteria

- [ ] Gitea stays up (`restart: unless-stopped` verified after a Mac Mini reboot)
- [ ] SSH push/pull work from the NixOS laptop without re-entering a password
- [ ] The `dashboard` repo exists and has at least one commit

Once these three boxes are checked → move to **Phase 2 (Caddy reverse proxy)**,
to replace `http://ip:3000` with `http://gitea.local`.

---

## Known gotchas

- If Gitea shows a "Failed to connect to database" error on first run: check the
  permissions on the `./data` folder (must be writable by the UID/GID set in
  `docker-compose.yml`).
- If SSH push fails with `Permission denied (publickey)`: make sure the public key
  was actually added to the right user account in Gitea, not just generated locally.
- Don't expose port 3000 directly to the Internet — once Tailscale or the reverse
  proxy is in place, this port should only be reachable locally/over VPN.
