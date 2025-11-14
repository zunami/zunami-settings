# Authentik on Synology NAS - Complete Guide

![Authentik](https://img.shields.io/badge/Authentik-2025.10.1-blue) ![Synology](https://img.shields.io/badge/Synology-DSM%207.x-orange) ![Docker](https://img.shields.io/badge/Docker-Compose-blue)

> Self-hosted SSO/Identity Provider on Synology NAS with Docker & Portainer. Tested on DS923+, works on DS920+, DS720+, DS420+ and similar.

**⏱️ Setup time: 15 minutes**

---

## 📋 Table of Contents

- [Installation](#installation)
- [Configuration](#configuration)
- [Common Issues](#common-issues)
- [Updates](#updates)
- [Best Practices](#best-practices)

---

## Installation

### Prerequisites

- Synology NAS with DSM 7.0+
- 4GB+ RAM (8GB recommended)
- ~2GB free disk space

### Step 1: Enable SSH

**DSM** → **Control Panel** → **Terminal & SNMP** → Enable SSH

```bash
ssh admin@YOUR_NAS_IP
```

### Step 2: Install Portainer

```bash
docker run -d \
  -p 9000:9000 \
  --name=portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

Access: `http://YOUR_NAS_IP:9000` (create admin account)

### Step 3: Prepare Directories

```bash
sudo mkdir -p /volume1/docker/authentik/{database,redis,media,templates,certs,backups}
sudo chmod -R 755 /volume1/docker/authentik
```

### Step 4: Generate Secrets

```bash
# PostgreSQL password (32 chars):
openssl rand -base64 32

# Authentik secret (60 chars):
openssl rand -base64 60
```

**Save these!** You'll need them next.

### Step 5: Deploy Stack

**Portainer** → **Stacks** → **Add stack** → Name: `authentik` → **Web editor**

<details>
<summary><b>📄 Click to show docker-compose.yml</b></summary>

```yaml
version: '3.8'

services:
  postgresql:
    image: docker.io/library/postgres:16-alpine
    container_name: authentik-db
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -d authentik -U authentik"]
      start_period: 20s
      interval: 30s
      retries: 5
      timeout: 5s
    volumes:
      - /volume1/docker/authentik/database:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: YOUR_POSTGRES_PASSWORD_HERE
      POSTGRES_USER: authentik
      POSTGRES_DB: authentik
    networks:
      - authentik_network

  redis:
    image: docker.io/library/redis:7-alpine
    container_name: authentik-redis
    restart: unless-stopped
    command: --save 60 1 --loglevel warning
    healthcheck:
      test: ["CMD-SHELL", "redis-cli ping | grep PONG"]
      start_period: 20s
      interval: 30s
      retries: 5
      timeout: 3s
    volumes:
      - /volume1/docker/authentik/redis:/data
    networks:
      - authentik_network

  server:
    image: ghcr.io/goauthentik/server:2025.10.1
    container_name: authentik-server
    restart: unless-stopped
    command: server
    environment:
      AUTHENTIK_REDIS__HOST: redis
      AUTHENTIK_POSTGRESQL__HOST: postgresql
      AUTHENTIK_POSTGRESQL__USER: authentik
      AUTHENTIK_POSTGRESQL__NAME: authentik
      AUTHENTIK_POSTGRESQL__PASSWORD: YOUR_POSTGRES_PASSWORD_HERE
      AUTHENTIK_SECRET_KEY: YOUR_AUTHENTIK_SECRET_KEY_HERE
      AUTHENTIK_ERROR_REPORTING__ENABLED: "false"
      AUTHENTIK_LOG_LEVEL: info
      AUTHENTIK_URL: http://YOUR_NAS_IP:9100
    volumes:
      - /volume1/docker/authentik/media:/media
      - /volume1/docker/authentik/templates:/templates
    ports:
      - "9100:9000"
      - "9543:9443"
    depends_on:
      - postgresql
      - redis
    networks:
      - authentik_network

  worker:
    image: ghcr.io/goauthentik/server:2025.10.1
    container_name: authentik-worker
    restart: unless-stopped
    command: worker
    environment:
      AUTHENTIK_REDIS__HOST: redis
      AUTHENTIK_POSTGRESQL__HOST: postgresql
      AUTHENTIK_POSTGRESQL__USER: authentik
      AUTHENTIK_POSTGRESQL__NAME: authentik
      AUTHENTIK_POSTGRESQL__PASSWORD: YOUR_POSTGRES_PASSWORD_HERE
      AUTHENTIK_SECRET_KEY: YOUR_AUTHENTIK_SECRET_KEY_HERE
      AUTHENTIK_ERROR_REPORTING__ENABLED: "false"
      AUTHENTIK_LOG_LEVEL: info
      AUTHENTIK_URL: http://YOUR_NAS_IP:9100
    volumes:
      - /volume1/docker/authentik/media:/media
      - /volume1/docker/authentik/templates:/templates
      - /volume1/docker/authentik/certs:/certs
    depends_on:
      - postgresql
      - redis
    networks:
      - authentik_network

networks:
  authentik_network:
    driver: bridge
```

</details>

**🔴 Replace these 4 placeholders:**
1. `YOUR_POSTGRES_PASSWORD_HERE` (3 times)
2. `YOUR_AUTHENTIK_SECRET_KEY_HERE` (2 times)
3. `YOUR_NAS_IP` (2 times)

Click **"Deploy the stack"**. Wait 2-3 minutes.

### Step 6: Verify Installation

```bash
# Check containers:
docker ps | grep authentik

# All 4 should show "Up" status:
# - authentik-server
# - authentik-worker
# - authentik-db
# - authentik-redis

# Watch startup:
docker logs -f authentik-server
# Wait for: "Application startup complete"
```

Access: `http://YOUR_NAS_IP:9100`

---

## Configuration

### Initial Setup

1. Open `http://YOUR_NAS_IP:9100/if/flow/initial-setup/`
2. Create admin account:
   - Email: `admin@example.com`
   - Username: `akadmin`
   - Password: (strong password)
3. Set domain: `YOUR_NAS_IP:9100`
4. Finish setup

### Create First User

**Admin Interface** → **Directory** → **Users** → **Create**

- Username: `testuser`
- Email: `test@example.com`
- Active: ✅

Go to **Password** tab → Set password → Save

**Test login:** Open incognito browser → `http://YOUR_NAS_IP:9100` → Login

---

## Common Issues

### 🔴 Issue: Containers Won't Start

**Check logs:**
```bash
docker logs authentik-server --tail 50
docker logs authentik-worker --tail 50
docker logs authentik-db --tail 50
```

**Most common causes:**

| Error | Solution |
|-------|----------|
| `password authentication failed` | Check `POSTGRES_PASSWORD` is **identical** in postgresql, server, and worker |
| `could not connect to server` | Wait 30s, database needs time to initialize |
| `Permission denied` | `sudo chmod -R 755 /volume1/docker/authentik` |

---

### 🔴 Issue: Cannot Access Web Interface

**Troubleshooting:**

```bash
# 1. Check if server is running:
docker ps | grep authentik-server

# 2. Test port locally:
curl http://localhost:9100

# 3. Check firewall:
# DSM → Control Panel → Security → Firewall
# Allow port 9100

# 4. Port conflict? Change in docker-compose.yml:
ports:
  - "9200:9000"  # Changed to 9200
```

---

### 🔴 Issue: Version Mismatch Error

**Symptoms:**
```
ERROR: relation "authentik_events_systemtask" does not exist
ERROR: column authentik_core_authenticatedsession.expiring does not exist
```

**Cause:** Server and worker have different versions.

**Check:**
```bash
docker inspect authentik-server | grep Image
docker inspect authentik-worker | grep Image
# Must be IDENTICAL!
```

**Fix:**
```yaml
# In docker-compose.yml, both must match:
server:
  image: ghcr.io/goauthentik/server:2025.10.1  # ← Same
worker:
  image: ghcr.io/goauthentik/server:2025.10.1  # ← Same
```

Redeploy:
```bash
docker-compose down
docker-compose up -d
```

---

### 🟡 Warning: High Memory Usage

**If using >2GB RAM, optimize:**

```yaml
# Add to worker environment:
worker:
  environment:
    AUTHENTIK_WORKER__THREADS: 2  # Default is 4

# Add to postgresql:
postgresql:
  command: postgres -c shared_buffers=256MB -c max_connections=100
```

---

### 🟢 Warning: Redis TCP Backlog (Harmless)

```
WARNING: The TCP backlog setting of 511 cannot be enforced
```

**This is normal on Synology and harmless for <50 users.** Ignore it.

---

### 🟢 Warning: PostgreSQL Lock Warnings (Harmless)

```
WARNING: you don't own a lock of type ExclusiveLock
```

**This is a known Celery/Authentik behavior.** Ignore unless login fails.

---

## Updates

### Before Updating

**1. Backup database:**
```bash
docker exec authentik-db pg_dump -U authentik authentik > \
  /volume1/docker/authentik/backups/backup_$(date +%Y%m%d).sql
```

**2. Backup config:**
- Portainer → Stacks → authentik → Editor
- Copy entire docker-compose.yml to your computer

### Update Process

**1. Check release notes:**
- https://github.com/goauthentik/authentik/releases

**2. Update docker-compose.yml:**

```yaml
server:
  image: ghcr.io/goauthentik/server:2025.10.1  # ← New version
worker:
  image: ghcr.io/goauthentik/server:2025.10.1  # ← Same version!
```

**3. Pull images:**
```bash
docker pull ghcr.io/goauthentik/server:2025.10.1
```

**4. Deploy:**
- Portainer → Stacks → authentik → Editor
- Update versions
- ☑️ Check "Re-pull images"
- Click "Update the stack"

**5. Monitor:**
```bash
docker logs -f authentik-server
# Wait for: "Running migrations: ..." → "Application startup complete"
```

**6. Verify:**
- Login to web interface
- Check version in Admin → System

### Rollback (If Update Fails)

```bash
# 1. Stop everything:
docker-compose down

# 2. Restore config:
# Edit docker-compose.yml back to old version

# 3. Restore database (if needed):
docker-compose up -d postgresql  # Start only DB
sleep 10
docker exec -i authentik-db psql -U authentik authentik < \
  /volume1/docker/authentik/backups/backup_YYYYMMDD.sql

# 4. Start all:
docker-compose up -d
```

---

## Best Practices

### 1. Never Use `:latest` Tag

❌ **BAD:**
```yaml
image: ghcr.io/goauthentik/server:latest
```

✅ **GOOD:**
```yaml
image: ghcr.io/goauthentik/server:2025.10.1
```

**Why?**
- Synology Docker cache issues
- Portainer "Re-pull" unreliable
- No control over updates
- Harder to rollback

---

### 2. Server & Worker Must Match

```yaml
server:
  image: ghcr.io/goauthentik/server:2025.10.1  # ← Same
worker:
  image: ghcr.io/goauthentik/server:2025.10.1  # ← Same
```

**Different versions = Database migration failures!**

---

### 3. Automated Backups

**DSM** → **Control Panel** → **Task Scheduler** → **Create** → **User-defined script**

- Task: `Authentik Backup`
- User: `root`
- Schedule: Daily at 3:00 AM

```bash
#!/bin/bash
docker exec authentik-db pg_dump -U authentik authentik > \
  /volume1/docker/authentik/backups/backup_$(date +%Y%m%d).sql

# Keep only last 7 days:
find /volume1/docker/authentik/backups/ -name "backup_*.sql" -mtime +7 -delete
```

---

### 4. Security Hardening

**A) Strong Passwords:**
- PostgreSQL: 32+ characters
- Authentik secret: 60+ characters
- Admin user: 16+ characters

**B) Restrict Network:**
- DSM → Control Panel → Security → Firewall
- Allow port 9100 only from LAN
- Or use HTTPS reverse proxy

**C) Enable HTTPS:**
- Use Synology's reverse proxy (Application Portal)
- Or deploy Nginx Proxy Manager
- Or use Caddy with auto Let's Encrypt

---

### 5. Pin PostgreSQL & Redis Versions

```yaml
postgresql:
  image: postgres:16-alpine  # Pin major version

redis:
  image: redis:7-alpine  # Pin major version
```

**Never auto-update PostgreSQL major versions!** (16 → 17 requires manual migration)

---

## Quick Reference

### Essential Commands

```bash
# View logs:
docker logs -f authentik-server
docker logs -f authentik-worker

# Restart:
docker restart authentik-server authentik-worker

# Full restart:
docker-compose restart

# Stop all:
docker-compose down

# Start all:
docker-compose up -d

# Backup database:
docker exec authentik-db pg_dump -U authentik authentik > backup.sql

# Check status:
docker ps | grep authentik

# Check resources:
docker stats authentik-server authentik-worker authentik-db
```

---

### Default URLs

- **Web Interface:** `http://NAS_IP:9100`
- **Admin:** `http://NAS_IP:9100/if/admin/`
- **API:** `http://NAS_IP:9100/api/v3/`
- **Portainer:** `http://NAS_IP:9000`

---

### Default Ports

- `9100` → HTTP (web interface)
- `9543` → HTTPS (if configured)
- `9000` → Portainer
- Internal: `5432` (PostgreSQL), `6379` (Redis)

---

## Troubleshooting Checklist

**Installation fails?**

- [ ] All 4 placeholders replaced in docker-compose.yml?
- [ ] `POSTGRES_PASSWORD` identical in 3 places?
- [ ] `AUTHENTIK_SECRET_KEY` identical in 2 places?
- [ ] Directories created: `/volume1/docker/authentik/...`?
- [ ] Port 9100 not used by other service?
- [ ] Waited 2-3 minutes for startup?

**Update fails?**

- [ ] Backup created before update?
- [ ] Server and worker have **identical** versions?
- [ ] Images manually pulled via SSH?
- [ ] Watched logs for migration errors?
- [ ] Database healthy? (`docker logs authentik-db`)

**Can't login?**

- [ ] Setup wizard completed?
- [ ] Admin user created?
- [ ] Password set correctly?
- [ ] Check server logs for errors?
- [ ] Database accessible? (`docker exec -it authentik-db psql -U authentik`)

---

## Resources

- **Authentik Docs:** https://docs.goauthentik.io/
- **GitHub:** https://github.com/goauthentik/authentik
- **Discord:** https://discord.gg/jg33eMhnj6
- **Releases:** https://github.com/goauthentik/authentik/releases

---

## Contributing

Found an issue? Have improvements?

- Open an [Issue](../../issues)
- Submit a [Pull Request](../../pulls)
- Join [Discussions](../../discussions)

---

## License

MIT License - See [LICENSE](LICENSE) file.

Authentik itself is licensed separately: https://github.com/goauthentik/authentik/blob/main/LICENSE

---

**Tested on:** Synology DS923+ (DSM 7.3) | Docker 20.10+ | Authentik 2025.10.1

**Tags:** `authentik` `synology` `nas` `docker` `portainer` `sso` `self-hosted` `homelab`

**Made with ❤️ for the homelab community**
