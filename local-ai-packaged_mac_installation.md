# local-ai-packaged Installation on Mac Mini M4

> **Documentation created with:** Claude.ai (Anthropic)  
> **Date:** 2025-12-03  
> **Installation timeframe:** 2025-12-02 21:00 - 2025-12-03 07:50 CET  
> **Status:** ✅ Successfully completed

---

## 📋 Table of Contents

1. [System Overview](#system-overview)
2. [Prerequisites](#prerequisites)
3. [Installation](#installation)
4. [Critical Issues & Solutions](#critical-issues--solutions)
5. [Final Configuration](#final-configuration)
6. [Container Overview](#container-overview)
7. [Important Insights for AI](#important-insights-for-ai)
8. [Start/Stop Commands](#startstop-commands)
9. [Service URLs](#service-urls)

---

## 🖥️ System Overview

### Hardware
- **Model:** Mac mini M4 (2024)
- **CPU:** Apple M4 (10 Cores: 4 Performance + 6 Efficiency)
- **RAM:** 24 GB LPDDR5
- **Storage:** 1 TB SSD
- **Network:** 192.168.1.x

### Software Versions
```
macOS:              26.0.1 (Darwin 25.0.0)
Docker Desktop:     29.0.1
Python:             3.13.7
Ollama:             Installed locally (NOT in Docker)
Hostname:           custom-hostname
```

### Repository
```
Source:     https://github.com/coleam00/local-ai-packaged
Commit:     Latest (as of 2025-12-02)
Path:       /Users/username/docker/local-ai-packaged/
```

---

## 🚀 Prerequisites

### 1. Install Ollama Locally

```bash
# Download and install Ollama DMG
# https://ollama.ai/download

# Pull models
ollama pull qwen2.5-coder:32b
ollama pull mistral-small:24b
ollama pull phi4:14b

# Ollama runs on port 11434
curl http://localhost:11434/api/tags
```

**⚠️ IMPORTANT:** Ollama MUST be installed locally (not in Docker)!

---

### 2. Install Docker Desktop

```bash
# Download Docker Desktop for Mac M-Series
# https://www.docker.com/products/docker-desktop

# Start Docker Desktop
# Settings → Resources → Memory: 8GB+ recommended
```

---

### 3. Python & Dependencies

```bash
# Python 3.13.7 already installed with macOS
python3 --version

# No additional Python packages required
# Repository includes everything
```

---

## 📦 Installation

### 1. Clone Repository

```bash
cd ~/docker/
git clone https://github.com/coleam00/local-ai-packaged.git
cd local-ai-packaged
```

---

### 2. Configure Environment Variables

```bash
cp .env.example .env
nano .env
```

**Critical changes in `.env`:**

```bash
# Line 21: Ollama Host for LOCAL Ollama
OLLAMA_HOST=http://host.docker.internal:11434

# Line 102: VAULT_ENC_KEY
VAULT_ENC_KEY=<32-alphanumeric-chars>

# Generate new passwords:
N8N_ENCRYPTION_KEY=<32-alphanumeric-chars>
POSTGRES_PASSWORD=<24-alphanumeric-chars>
DASHBOARD_USERNAME=your_username
DASHBOARD_PASSWORD=<24-alphanumeric-chars>
NEO4J_AUTH=neo4j/<20-alphanumeric-chars>
FLOWISE_USERNAME=your_username
FLOWISE_PASSWORD=<20-alphanumeric-chars>

# Langfuse ENCRYPTION_KEY
ENCRYPTION_KEY=<64-hex-chars-0-9-a-f>
```

**Password Requirements:**

| Variable | Length | Format | Example Format |
|----------|--------|--------|----------------|
| `VAULT_ENC_KEY` | 32 chars | Alphanumeric | `AbC123XyZ456...` (32 total) |
| `N8N_ENCRYPTION_KEY` | 32 chars | Alphanumeric | `gH8iPqR4sTuV...` (32 total) |
| `POSTGRES_PASSWORD` | 24 chars | Alphanumeric | `vEcGkR1oKmV...` (24 total) |
| `DASHBOARD_PASSWORD` | 24 chars | Alphanumeric | `wYqXgW2fNbQ...` (24 total) |
| `NEO4J_AUTH` | 20 chars | neo4j/password | `neo4j/uKwI3iN9...` (20 after /) |
| `FLOWISE_PASSWORD` | 20 chars | Alphanumeric | `Fl0wIs2024...` (20 total) |
| `ENCRYPTION_KEY` | 64 chars | Hex (0-9,a-f) | `5daa...bad` (64 hex total) |

**Password Generation Commands:**

```bash
# For 32-character alphanumeric (VAULT_ENC_KEY, N8N_ENCRYPTION_KEY)
python3 -c "import secrets; print(secrets.token_urlsafe(24)[:32])"

# For 24-character alphanumeric (POSTGRES, DASHBOARD)
python3 -c "import secrets; print(secrets.token_urlsafe(18)[:24])"

# For 20-character alphanumeric (NEO4J, FLOWISE)
python3 -c "import secrets; print(secrets.token_urlsafe(15)[:20])"

# For 64 hex characters (ENCRYPTION_KEY - Langfuse)
openssl rand -hex 32
```

**⚠️ CRITICAL:** 
- VAULT_ENC_KEY must be EXACTLY 32 characters!
- ENCRYPTION_KEY must be EXACTLY 64 hex characters (0-9, a-f)!
- Generate ALL passwords yourself - never reuse examples!

---

### 3. Configure Supabase .env

```bash
nano supabase/docker/.env
```

**⚠️ CRITICAL:** `VAULT_ENC_KEY` must be IDENTICAL to main .env!

```bash
# Supabase .env - Find line with VAULT_ENC_KEY
VAULT_ENC_KEY=<exact-same-32-chars-as-main-env>
```

**BOTH .env files must have the same VAULT_ENC_KEY!**

---

### 4. Modify docker-compose.yml

```bash
nano docker-compose.yml
```

**Change Flowise Volume (around line 150):**

```yaml
# WRONG (doesn't work on Mac):
volumes:
  - ~/.flowise:/root/.flowise

# CORRECT (Mac Docker Desktop):
volumes:
  - ./flowise_data:/root/.flowise
```

**Reason:** Docker Desktop on Mac cannot mount `~/.flowise` (File Sharing Permissions).

---

### 5. Start Services

```bash
cd ~/docker/local-ai-packaged

# Start with profile "none" (because Ollama runs locally)
python3 start_services.py --profile none
```

**Expected output:**
```
Starting services with profile: none
[+] Running 27/27
✔ Container supabase-db         Started
✔ Container supabase-auth       Started
✔ Container n8n                 Started
...
```

---

## 🚨 Critical Issues & Solutions

### Issue 1: Langfuse Worker Crashes Immediately

**Symptoms:**
```bash
docker logs localai-langfuse-worker-1

Error: ENCRYPTION_KEY must be 256 bits, 64 string characters in hex format
```

**Root Cause:**
- ENCRYPTION_KEY in `.env` was too short or wrong format
- Langfuse requires EXACTLY 64 hex characters (0-9, a-f)

**Solution:**

```bash
# 1. Generate new key (64 hex characters)
openssl rand -hex 32

# 2. Add to .env
nano ~/docker/local-ai-packaged/.env
# ENCRYPTION_KEY=<paste your 64-hex key here>

# 3. Restart container
docker stop localai-langfuse-worker-1
docker rm localai-langfuse-worker-1
docker compose -p localai -f docker-compose.yml up -d langfuse-worker
```

**Verify format:**
- Must be EXACTLY 64 characters
- Only 0-9 and a-f allowed
- Example format: `1a2b3c4d5e...67890` (64 chars total)

**Status:** ✅ Fixed - Container runs stable

---

### Issue 2: Supabase Pooler Crashes with Cipher Error

**Symptoms:**
```bash
docker logs supabase-pooler

Unknown cipher or invalid key size
AES-256-GCM cannot use key
```

**Root Cause:**
- VAULT_ENC_KEY was different in two `.env` files
- Wrong key format (BASE64 instead of plain 32-character string)

**Solution:**

```bash
# 1. Generate 32-character key (NOT BASE64!)
python3 -c "import secrets; print(secrets.token_urlsafe(24)[:32])"

# 2. Add to BOTH .env files IDENTICALLY
nano ~/docker/local-ai-packaged/.env
# VAULT_ENC_KEY=<your-32-char-key>  (Line 102)

nano ~/docker/local-ai-packaged/supabase/docker/.env
# VAULT_ENC_KEY=<exact-same-32-char-key>

# 3. Restart supavisor container
docker compose -p localai -f supabase/docker/docker-compose.yml up -d supavisor
```

**Verify:**
- Must be EXACTLY 32 characters
- Alphanumeric (a-z, A-Z, 0-9, -, _)
- Must be IDENTICAL in both .env files

**⚠️ CRITICAL:** Both .env files must have EXACTLY the same VAULT_ENC_KEY!

**Status:** ✅ Fixed - Container healthy

---

### Issue 3: Open WebUI Embedding Model Download Fails

**Symptoms:**
```bash
docker logs open-webui

RuntimeError: Data processing error: CAS service error
ReqwestMiddleware Error: Request failed after 5 retries
```

**Root Cause:**
- HuggingFace download unstable
- Sentence-Transformers model too large for first attempts

**Solution:**

```bash
# Simply wait and restart container
docker restart open-webui

# After 2-3 attempts, download succeeds
docker logs open-webui --follow
# Wait for: "Fetching 30 files: 100%"
```

**Alternative (if download never works):**

```bash
# Set smaller embedding model in .env
nano .env
RAG_EMBEDDING_MODEL=BAAI/bge-small-en-v1.5
```

**Status:** ✅ Fixed - Model successfully loaded after 3rd attempt

---

### Issue 4: Realtime Container "unhealthy"

**Symptoms:**
```bash
docker ps
# realtime-dev.supabase-realtime   Up 25 minutes (unhealthy)

docker logs realtime-dev.supabase-realtime
# HEAD /api/tenants/realtime-dev/health
# Sent 403 in 150µs
```

**Root Cause:**
- Realtime health check expects JWT token
- Known Supabase issue in self-hosted setups

**Solution:**
- **NO fix needed!**
- Container runs perfectly despite "unhealthy" status
- WebSocket/Realtime features work normally

**Status:** ⚠️ Unhealthy but functional (acceptable)

---

## ⚙️ Final Configuration

### docker-compose.yml Changes

```yaml
# Line 21: Ollama Host
environment:
  - OLLAMA_HOST=http://host.docker.internal:11434

# Line ~150: Flowise Volume
volumes:
  - ./flowise_data:/root/.flowise  # Not ~/.flowise
```

---

### Main .env File Requirements

**Path:** `/Users/username/docker/local-ai-packaged/.env`

**Critical Keys with Specifications:**

```bash
# Line 21
OLLAMA_HOST=http://host.docker.internal:11434

# Password specifications:
N8N_ENCRYPTION_KEY=<32-alphanumeric-chars>
POSTGRES_PASSWORD=<24-alphanumeric-chars>
DASHBOARD_USERNAME=your_username
DASHBOARD_PASSWORD=<24-alphanumeric-chars>
NEO4J_AUTH=neo4j/<20-alphanumeric-chars>
FLOWISE_USERNAME=your_username
FLOWISE_PASSWORD=<20-alphanumeric-chars>

# Line 102
VAULT_ENC_KEY=<32-alphanumeric-chars>

# Langfuse (somewhere in file)
ENCRYPTION_KEY=<64-hex-chars-0-9-a-f>
```

---

### Supabase .env File

**Path:** `/Users/username/docker/local-ai-packaged/supabase/docker/.env`

**Critical:**

```bash
# MUST BE IDENTICAL to main .env!
VAULT_ENC_KEY=<exact-same-32-alphanumeric-chars>
```

---

## 📦 Container Overview

### Started Services (27+ Containers)

```
✅ MAIN SERVICES:
   • n8n                    (Port 5678/8001)
   • Open WebUI             (Port 3000/8002)
   • Flowise                (Port 3001/8003)
   • Supabase Studio        (Port 8000/8005)
   • Langfuse Web           (Port 3002/8007)
   • Langfuse Worker        (Background)

✅ DATABASES:
   • PostgreSQL             (Port 5432)
   • Neo4j                  (Port 7474/8008)
   • Qdrant                 (Port 6333)
   • Redis/Valkey           (Cache)
   • ClickHouse             (Analytics)

✅ SUPABASE STACK:
   • supabase-db            (healthy)
   • supabase-auth          (healthy)
   • supabase-rest          (running)
   • supabase-storage       (healthy)
   • supabase-pooler        (healthy) ✅ FIXED
   • supabase-kong          (healthy)
   • supabase-meta          (healthy)
   • supabase-studio        (healthy)
   • supabase-analytics     (healthy)
   • supabase-imgproxy      (healthy)
   • supabase-vector        (healthy)
   • realtime-dev           (unhealthy - works anyway)

✅ SUPPORTING:
   • SearXNG                (Port 8006)
   • Caddy                  (Reverse Proxy)
   • MinIO                  (S3 Storage)
```

---

## 🧠 Important Insights for AI

### Critical Differences Mac vs. Linux

1. **Docker Socket Path:**
   ```bash
   # Mac Docker Desktop
   /var/run/docker.sock → Symlink to ~/.docker/run/docker.sock
   ```

2. **Volume Mounts:**
   ```yaml
   # DOES NOT WORK on Mac:
   - ~/.flowise:/root/.flowise
   
   # WORKS on Mac:
   - ./flowise_data:/root/.flowise
   ```

3. **Host Networking:**
   ```yaml
   # Use host.docker.internal instead of localhost
   OLLAMA_HOST=http://host.docker.internal:11434
   ```

4. **Network Interface:**
   ```bash
   # Mac doesn't have "en0" interface by default
   # Find IP directly via ifconfig
   ```

---

### Encryption Keys - Absolute Requirements

| Key | Length | Format | Generation Command |
|-----|--------|--------|-------------------|
| `VAULT_ENC_KEY` | 32 chars | Alphanumeric | `python3 -c "import secrets; print(secrets.token_urlsafe(24)[:32])"` |
| `N8N_ENCRYPTION_KEY` | 32 chars | Alphanumeric | `python3 -c "import secrets; print(secrets.token_urlsafe(24)[:32])"` |
| `ENCRYPTION_KEY` (Langfuse) | 64 chars | Hex (0-9,a-f) | `openssl rand -hex 32` |
| `POSTGRES_PASSWORD` | 24 chars | Alphanumeric | `python3 -c "import secrets; print(secrets.token_urlsafe(18)[:24])"` |
| `DASHBOARD_PASSWORD` | 24 chars | Alphanumeric | `python3 -c "import secrets; print(secrets.token_urlsafe(18)[:24])"` |
| `NEO4J_AUTH` | 20 chars | neo4j/password | `python3 -c "import secrets; print('neo4j/' + secrets.token_urlsafe(15)[:20])"` |
| `FLOWISE_PASSWORD` | 20 chars | Alphanumeric | `python3 -c "import secrets; print(secrets.token_urlsafe(15)[:20])"` |

**Critical Rules:**
1. VAULT_ENC_KEY must be EXACTLY 32 characters (not more, not less)
2. VAULT_ENC_KEY must be IDENTICAL in both .env files
3. ENCRYPTION_KEY must be EXACTLY 64 hex characters (0-9, a-f only)
4. Never use BASE64 encoding for VAULT_ENC_KEY
5. Generate all passwords fresh - never reuse examples

---

### Docker Compose Project Name

```bash
# All containers run under project "localai"
docker compose -p localai -f docker-compose.yml <command>
docker compose -p localai -f supabase/docker/docker-compose.yml <command>

# Orphan warnings are NORMAL (different compose files)
```

---

### Container Start Order

```bash
# 1. Supabase first
cd ~/docker/local-ai-packaged
python3 start_services.py --profile none

# Automatically starts:
# - Supabase stack (supabase/docker/docker-compose.yml)
# - Main services (docker-compose.yml)
```

---

### Common Error Sources

1. **Ollama not reachable:**
   - Check: `curl http://localhost:11434/api/tags`
   - Fix: `OLLAMA_HOST=http://host.docker.internal:11434` in .env

2. **Langfuse Worker crashes:**
   - Check: `docker logs localai-langfuse-worker-1`
   - Fix: ENCRYPTION_KEY must be 64 hex characters (0-9, a-f)

3. **Supabase Pooler crashes:**
   - Check: `docker logs supabase-pooler`
   - Fix: VAULT_ENC_KEY identical in BOTH .env files and exactly 32 chars

4. **n8n cannot import workflows:**
   - Normal on first start (no webhooks present)
   - Error: "Active version not found" → Ignore

5. **Open WebUI Embedding download hangs:**
   - Wait and restart multiple times
   - Alternative: Set smaller model in .env

---

## 🔄 Start/Stop Commands

### Start Services

```bash
cd ~/docker/local-ai-packaged
python3 start_services.py --profile none
```

**Profile Options:**
- `--profile none` → Ollama local (RECOMMENDED for Mac)
- `--profile gpu` → Ollama in Docker with GPU
- `--profile cpu` → Ollama in Docker without GPU

---

### Stop Services

```bash
cd ~/docker/local-ai-packaged

# Stop Supabase
docker compose -p localai -f supabase/docker/docker-compose.yml down

# Stop main services
docker compose -p localai -f docker-compose.yml down
```

---

### Restart Individual Containers

```bash
# Restart container
docker restart <container-name>

# Stop, remove, recreate
docker stop <container-name>
docker rm <container-name>
docker compose -p localai -f docker-compose.yml up -d <service-name>
```

---

### Check Status

```bash
# Show all containers
docker ps --format "table {{.Names}}\t{{.Status}}"

# Show only unhealthy containers
docker ps --filter "health=unhealthy"

# View logs
docker logs <container-name> --tail 50 --follow

# Disk usage
docker system df
```

---

## 🌐 Service URLs

### Main Services

```
n8n:              http://localhost:5678/
Open WebUI:       http://localhost:3000/
Flowise:          http://localhost:3001/
Supabase Studio:  http://localhost:8000/
Langfuse:         http://localhost:3002/
Neo4j Browser:    http://localhost:7474/
Qdrant Dashboard: http://localhost:6333/dashboard
```

### Via Caddy Reverse Proxy

```
n8n:              http://localhost:8001/
Open WebUI:       http://localhost:8002/
Flowise:          http://localhost:8003/
Supabase:         http://localhost:8005/
Neo4j:            http://localhost:8008/
SearXNG:          http://localhost:8006/
```

---

## 📝 Account Creation

### n8n (http://localhost:5678/)
- First visit: Create account automatically
- Configure Ollama: `http://host.docker.internal:11434`

### Open WebUI (http://localhost:3000/)
- First visit: Create account automatically
- Ollama connection detected automatically

### Flowise (http://localhost:3001/)
- Login with: `FLOWISE_USERNAME` / `FLOWISE_PASSWORD` from .env
- Credentials already configured

### Supabase (http://localhost:8000/)
- Login with: `DASHBOARD_USERNAME` / `DASHBOARD_PASSWORD` from .env
- Credentials already configured

### Langfuse (http://localhost:3002/)
- First visit: Create organization/account automatically
- Create project for AI monitoring

### Neo4j (http://localhost:7474/)
- Login with: `NEO4J_AUTH` from .env (format: `neo4j/password`)
- Credentials already configured

---

## 🧪 Functionality Tests

### 1. Open WebUI Chat Test

```bash
# 1. Open Open WebUI
open http://localhost:3000/

# 2. Create account
# 3. Select model (e.g., qwen2.5-coder:32b)
# 4. Start chat: "Hello! Can you help me with coding?"
# 5. Response should stream smoothly ✅
```

---

### 2. n8n Workflow Test

```bash
# 1. Open n8n
open http://localhost:5678/

# 2. Create account
# 3. New Workflow → Add Node → AI → Ollama
# 4. Credentials: http://host.docker.internal:11434
# 5. Model: qwen2.5-coder:32b
# 6. Test execution ✅
```

---

### 3. Flowise Chatflow Test

```bash
# 1. Open Flowise
open http://localhost:3001/

# 2. Login with credentials from .env
# 3. New Chatflow → Chat Model → Ollama
# 4. Base URL: http://host.docker.internal:11434
# 5. Model: mistral-small:24b
# 6. Test → should respond ✅
```

---

## 📊 Resource Usage

### After Successful Installation

```
Images:      25.03GB (33 images)
Containers:  359.1MB (29 active)
Volumes:     3.709GB (13 active)
Total:       ~29GB
```

### RAM Usage

```
Docker Desktop:  ~4-6GB
All Containers:  ~8-12GB
macOS System:    ~4GB
Available:       ~8GB (out of 24GB total)
```

---

## 🔐 Backup Strategy

### Backup Critical Files

```bash
# Create backup folder
mkdir -p ~/Desktop/local-ai-backup

# Copy .env files
cp ~/docker/local-ai-packaged/.env ~/Desktop/local-ai-backup/main.env
cp ~/docker/local-ai-packaged/supabase/docker/.env ~/Desktop/local-ai-backup/supabase.env

# Copy docker-compose.yml
cp ~/docker/local-ai-packaged/docker-compose.yml ~/Desktop/local-ai-backup/

# Back up to NAS/Cloud!
```

**⚠️ IMPORTANT:** Backup both .env files regularly!

---

## 🐛 Troubleshooting

### Container Crashes Immediately

```bash
# 1. Check logs
docker logs <container-name> --tail 100

# 2. Check container status
docker inspect <container-name> | grep Status

# 3. Force restart
docker stop <container-name>
docker rm <container-name>
docker compose -p localai -f docker-compose.yml up -d <service>
```

---

### Ollama Not Reachable

```bash
# 1. Is Ollama running?
curl http://localhost:11434/api/tags

# 2. Restart Ollama (Mac)
# Close Ollama.app and reopen

# 3. Check .env
grep OLLAMA_HOST .env
# Must be: http://host.docker.internal:11434
```

---

### Supabase Pooler Error

```bash
# 1. Check VAULT_ENC_KEY in BOTH .env files
grep VAULT_ENC_KEY ~/docker/local-ai-packaged/.env
grep VAULT_ENC_KEY ~/docker/local-ai-packaged/supabase/docker/.env

# Must be IDENTICAL and exactly 32 characters!

# 2. Recreate container
docker compose -p localai -f supabase/docker/docker-compose.yml up -d supavisor
```

---

### Langfuse Worker Error

```bash
# 1. Check ENCRYPTION_KEY
grep ENCRYPTION_KEY .env | grep -v "^#"

# Must be exactly 64 hex characters (0-9, a-f only)

# 2. Regenerate if wrong
openssl rand -hex 32

# 3. Add to .env and restart container
docker stop localai-langfuse-worker-1
docker rm localai-langfuse-worker-1
docker compose -p localai -f docker-compose.yml up -d langfuse-worker
```

---

## 📚 Additional Resources

- **Repository:** https://github.com/coleam00/local-ai-packaged
- **Ollama:** https://ollama.ai/
- **Docker Desktop:** https://www.docker.com/products/docker-desktop
- **n8n Docs:** https://docs.n8n.io/
- **Open WebUI:** https://github.com/open-webui/open-webui
- **Supabase:** https://supabase.com/docs

---

## ✅ Installation Success Checklist

- [ ] Ollama running locally and reachable
- [ ] Docker Desktop installed and running
- [ ] Repository cloned to `~/docker/local-ai-packaged`
- [ ] `.env` configured with correct password lengths
- [ ] `supabase/docker/.env` configured (VAULT_ENC_KEY identical)
- [ ] `docker-compose.yml` modified (Flowise volume)
- [ ] ENCRYPTION_KEY: exactly 64 hex characters (0-9, a-f)
- [ ] VAULT_ENC_KEY: exactly 32 alphanumeric, identical in BOTH .env
- [ ] All passwords match specified character counts
- [ ] Services started with `python3 start_services.py --profile none`
- [ ] 27+ containers running (docker ps)
- [ ] Langfuse Worker running (healthy)
- [ ] Supabase Pooler running (healthy)
- [ ] Open WebUI Embedding model loaded
- [ ] All service URLs reachable
- [ ] Backup created (both .env + docker-compose.yml)

---

## 🎉 Successful Installation

**Final Status Check:**

```bash
# Container overview
docker ps --format "table {{.Names}}\t{{.Status}}" | head -30

# Should show:
# - 28-29 containers "Up" 
# - Most "healthy"
# - Only realtime-dev can be "unhealthy" (OK!)

# Unhealthy containers (should be only 1)
docker ps --filter "health=unhealthy"
```

**If everything runs: INSTALLATION SUCCESSFUL! 🎊**

---

## 📄 License

This documentation was created with Claude.ai and is licensed under the same license as the [local-ai-packaged repository](https://github.com/coleam00/local-ai-packaged).

---

**Created:** 2025-12-03  
**Version:** 1.1  
**Status:** Production  
**Platform:** macOS (Apple Silicon M4)  
**Docker:** Desktop 29.0.1  
**Created with:** Claude.ai (Anthropic)
