# Homelab: Container Update Monitoring

This documents how automatic container update checks and Telegram notifications are set up on this homelab.

## Why not Watchtower

Originally used [Watchtower](https://github.com/containrrr/watchtower), but the repo was **archived by its owner in December 2025** (last release v1.7.1) and is no longer maintained. Migrated to **[What's Up Docker (WUD)](https://github.com/getwud/wud)** instead, which is actively developed.

## Stack: What's Up Docker (WUD)

WUD watches running containers on this host and checks for newer image versions. Notifications are sent to Telegram; auto-update is **not** enabled globally (see exclusions below).

### `docker-compose.yml`

```yaml
services:
  whatsupdocker:
    image: getwud/wud:latest
    container_name: wud
    ports:
      - "3000:3000"
    environment:
      - WUD_AUTH_ADMIN_USER=admin
      - WUD_AUTH_ADMIN_PASSWORD=<see secrets, not committed>
      - WUD_TRIGGER_TELEGRAM_LOCAL_BOTTOKEN=<see secrets, not committed>
      - WUD_TRIGGER_TELEGRAM_LOCAL_CHATID=<see secrets, not committed>
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped
```

> **Do not commit real tokens/passwords to this repo.** Use a `.env` file (gitignored) or a secrets manager and reference variables instead, e.g. `${WUD_TELEGRAM_BOTTOKEN}`.

### Access

- Web UI: `http://<host-ip>:3000`
- Notifications: sent to a private Telegram bot/chat via Shoutrrr-style trigger

### Setting up Telegram notifications

1. Create a bot via **@BotFather** on Telegram (`/newbot`) → copy the bot token.
2. Message the new bot once, then message **@userinfobot** to get your numeric chat ID.
3. Set `WUD_TRIGGER_TELEGRAM_LOCAL_BOTTOKEN` and `WUD_TRIGGER_TELEGRAM_LOCAL_CHATID` accordingly.

## Nextcloud AIO — excluded from auto-update

Nextcloud is deployed via **Nextcloud All-in-One (AIO)**, which manages its own set of containers (`nextcloud-aio-*`) through a mastercontainer that talks to the Docker socket directly.

**Nextcloud's own guidance is not to let external tools (Watchtower, WUD, etc.) update these containers.** The AIO mastercontainer tracks and manages its containers' lifecycle itself; external tools restarting/updating them behind its back has caused documented breakage in the community (mastercontainer socket-path mismatches, containers left stopped mid-update with no easy recovery button).

**Containers excluded from auto-update:**
- `nextcloud-aio-mastercontainer`
- `nextcloud-aio-apache`
- `nextcloud-aio-nextcloud`
- `nextcloud-aio-imaginary`
- `nextcloud-aio-redis`
- `nextcloud-aio-database`
- `nextcloud-aio-notify-push`
- `nextcloud-aio-onlyoffice`

These are monitored for notifications only (if/when supported) or left out of WUD's scope entirely. **Updates for Nextcloud AIO are handled through its own dashboard** at `https://<host-ip>:8080`, or via its daily-backup-triggered auto-update feature.

## Summary of decisions

| Component | Auto-update via WUD? | Notes |
|---|---|---|
| General homelab containers | ✅ Yes | Notified + auto-updated |
| Nextcloud AIO containers | ❌ No | Managed via AIO's own dashboard |

## References

- [WUD documentation](https://getwud.app/docs)
- [WUD Telegram trigger docs](https://getwud.app/docs/configuration/triggers/telegram/)
- [Nextcloud AIO GitHub](https://github.com/nextcloud/all-in-one)
