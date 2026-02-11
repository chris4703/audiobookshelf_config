# 📚 AudioBookshelf Homelab – Architecture & Automation Notes

**Host:** macOS (Apple Silicon)  
**Primary App:** AudioBookshelf  
**Storage:** Docker named volume (`storyteller_data`)  
**External Media Drive:** G-RAID Thunderbolt volume  
**Email Delivery:** Gmail SMTP via local relay (Kindle compatible)

---

## 1. High-Level Goals

This AudioBookshelf setup is designed to:

- Keep **media storage isolated** in Docker volumes
- Allow **hands-off ingestion** of new content
- Maintain **safe, periodic backups**
- Work around **macOS background permission restrictions**
- Reliably deliver **ebooks to Kindle** using Send-to-Kindle requirements
- Remain **upgrade-safe** (no forks of AudioBookshelf)

---

## 2. Core Docker Stack

### Containers

| Service | Purpose |
|------|------|
| `audiobookshelf` | Main AudioBookshelf server |
| `abs-smtp-relay` | SMTP proxy to rewrite Kindle email subject |

### Volumes

| Volume | Purpose |
|------|------|
| `storyteller_data` (external) | Audiobooks & ebooks |
| `audiobookshelf_config` | App configuration |
| `audiobookshelf_metadata` | Metadata & database |

---

## 3. Docker Compose (Current State)

```yaml
services:
  audiobookshelf:
    image: ghcr.io/advplyr/audiobookshelf:latest
    container_name: audiobookshelf
    ports:
      - "13378:80"
    volumes:
      - storyteller_data:/audiobooks
      - audiobookshelf_config:/config
      - audiobookshelf_metadata:/metadata
    environment:
      - TZ=America/Phoenix
    restart: unless-stopped
    depends_on:
      - abs-smtp-relay

  abs-smtp-relay:
    build:
      context: ./smtp-relay
    container_name: abs-smtp-relay
    restart: unless-stopped
    environment:
      LISTEN_PORT: "2525"
      GMAIL_HOST: "smtp.gmail.com"
      GMAIL_PORT: "587"
      GMAIL_USER: "<gmail address>"
      GMAIL_PASS: "<gmail app password>"

volumes:
  storyteller_data:
    external: true
  audiobookshelf_config:
  audiobookshelf_metadata:
```

---

## 4. Automated Content Ingest

### Problem (macOS-specific)
- Background processes (launchd) **cannot reliably read `/Volumes/...`**
- Even with Full Disk Access, background shells hit `Operation not permitted`

### Solution
- A **LaunchAgent triggers a trusted `.app` wrapper**
- The wrapper runs a shell script
- All filesystem scanning happens **inside Docker**, not on the host

### Key Ingest Behaviors
- Runs every **5 minutes**
- Checks `_incoming` folder *inside the container*
- Merges folders cleanly into `/audiobooks`
- Never overwrites existing files
- Removes source files after successful copy
- Logs locally, then mirrors logs to the external drive (best-effort)

---

## 5. macOS Permission Workaround (Critical)

macOS applies privacy rules to background processes and removable volumes.

**Working pattern:**
1. Wrap ingest script in a minimal `.app`
2. Grant the app **Full Disk Access**
3. LaunchAgent runs the app via `open -a`
4. App executes the script with full volume access

This avoids SIP hacks or TCC database modification.

---

## 6. Logging Strategy

- **Primary:** `~/Library/Logs/abs_sync_incoming.log`
- **Mirror:** `/Volumes/.../_logs/abs_sync_incoming.log` (best-effort)

External logging failures never stop the job.

---

## 7. Backups

### Media Volume
- Monthly `tar.gz` snapshot of `storyteller_data`
- `pigz -1` for fast multi-core compression
- Excludes `image-cache/`
- Retains **2 most recent archives**

### Metadata
- `/metadata/backups` copied separately
- Enables DB restore without full media restore

---

## 8. Kindle Ebook Delivery

### Problem
- Kindle requires email subject **`Convert`**
- AudioBookshelf hard-codes subject text

### Solution
A local SMTP relay:
- Intercepts outgoing mail
- Rewrites subject to `Convert` for `@kindle.com`
- Forwards to Gmail SMTP using STARTTLS

No AudioBookshelf fork required.

---

## 9. Email Flow

```
AudioBookshelf
   ↓
abs-smtp-relay (subject rewrite)
   ↓
Gmail SMTP
   ↓
Amazon Send-to-Kindle
   ↓
Kindle device
```

---

## 10. Known Good State

- Incoming auto-ingests
- Logs update every 5 minutes
- Kindle deliveries succeed
- No background permission errors
- Docker restarts cleanly

---

## 11. Design Philosophy

This setup prioritizes:

- Predictability
- Observability
- Recoverability

It is intentionally boring—and therefore reliable.

