# 🐳 Installing Supabase on Synology NAS (Synology Container Manager)

**Complete step-by-step guide for installing Supabase on Synology DSM with Container Manager**

## 📋 Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Critical Notes](#critical-notes)
- [Phase 1: Preparation](#phase-1-preparation)
- [Phase 2: Preparing Files on Synology](#phase-2-preparing-files-on-synology)
- [Phase 3: Installation in Container Manager](#phase-3-installation-in-container-manager)
- [Phase 4: Testing Supabase](#phase-4-testing-supabase)
- [Phase 5: Setting up Reverse Proxy](#phase-5-setting-up-reverse-proxy-optional)
- [Troubleshooting](#troubleshooting)
- [FAQ](#faq)

---

## 🎯 Overview

This guide shows you how to install **Supabase** (Open-Source Firebase Alternative) completely on your Synology NAS - **without Portainer**, directly with the **Synology Container Manager**.

**What is Supabase?**
- 🗄️ PostgreSQL Database
- 🔐 Authentication & User Management
- 📦 File Storage
- ⚡ Realtime Subscriptions
- 🔌 Auto-generated REST API
- 📊 Web Dashboard

**After this guide you will have:**
- ✅ Complete Supabase system on your Synology
- ✅ 13 containers working together
- ✅ Dashboard accessible via browser
- ✅ Production-ready configuration
- ✅ Optional: Access via your own domain

---

## 📦 Prerequisites

### Hardware & Software

- **Synology NAS** with DSM 7.x
- **Docker/Container Manager** installed (Package Center → Container Manager)
- **At least 4 GB RAM** (recommended: 8+ GB)
- **At least 20 GB free storage**
- **Internet connection** for downloads

### User Accounts

- Admin access to DSM
- SSH access (optional but helpful)

### Time Required

- **Preparation:** 30 minutes
- **Installation:** 45 minutes
- **First Start:** 15-20 minutes
- **Total:** approx. 1.5-2 hours

---

## ⚠️ Critical Notes

### 🔴 Critical Points (from experience!)

#### 1. **Container Manager, NOT Portainer!**

> ⚠️ **IMPORTANT:** This installation ONLY works with the native **Synology Container Manager**!
>
> ❌ **Portainer does NOT work** for this installation!
>
> **Reason:** Portainer has issues with:
> - Relative paths in volume mounts
> - Environment variables from `.env` files
> - Complex Docker Compose projects with many dependencies

**Use instead:**
- ✅ DSM → Container Manager → Project

---

#### 2. **Installation must be built 3 times!**

> ⚠️ **IMPORTANT:** On first try, containers crash - this is NORMAL!
>
> **You must start the project 3 TIMES with "Build":**
> 
> 1. **Build:** Create project → Containers start (approx. 30 min.) → Crashes with error
> 2. **Build:** Click "Build" (DON'T delete!) → More containers start (approx. 15 min.) → Crashes again
> 3. **Build:** Click "Build" again → Everything runs stable! ✅
>
> **⚠️ IMPORTANT: Do NOT delete the project between builds! Just click "Build"!**
>
> **Why?**
> - Containers have dependencies on each other
> - Database must be fully initialized first
> - SQL migrations run on first start
> - On 2nd build, more containers come online
> - On 3rd build, everything is complete

**Patience is important!** After the 3rd build, everything runs stable.

---

#### 3. **Change Port 5432 → 54329!**

> ⚠️ **IMPORTANT:** The standard PostgreSQL port `5432` is often already in use!
>
> **Change it in the `.env` to `54329`:**
> ```bash
> POSTGRES_PORT=54329  # Instead of 5432
> ```
>
> **Why?**
> - Synology uses PostgreSQL internally
> - Port 5432 is often already occupied
> - Leads to "Port already in use" errors

---

#### 4. **Generate JWT tokens correctly!**

> 🔴 **CRITICAL:** Wrong JWT tokens = Supabase does NOT work!
>
> **You MUST use the official Supabase tool:**
> - 👉 https://supabase.com/docs/guides/self-hosting/docker#generate-api-keys
>
> **NEVER:**
> - ❌ Generate random strings
> - ❌ Use other token generators
> - ❌ Use the example tokens from `.env.example`
>
> **ALWAYS:**
> - ✅ Create a JWT_SECRET with at least 32 characters
> - ✅ Generate ANON_KEY with the Supabase tool from this JWT_SECRET
> - ✅ Generate SERVICE_ROLE_KEY with the SAME JWT_SECRET
>
> **All 3 keys must match!**

---

#### 5. **Change localhost → IP address!**

> ⚠️ **IMPORTANT:** URLs with `localhost` do NOT work for other devices!
>
> **In the `.env` replace ALL `localhost` with your Synology IP:**
> ```bash
> # ❌ WRONG:
> SITE_URL=http://localhost:3000
> 
> # ✅ CORRECT:
> SITE_URL=https://192.168.1.10:8843
> ```
>
> **Otherwise:**
> - Only accessible on Synology itself
> - Other PCs/phones in network cannot access
> - API requests fail

---

#### 6. **Port 8843 is HTTPS, not HTTP!**

> ⚠️ **IMPORTANT:** Kong (Reverse Proxy) runs in HTTPS mode by default on port 8843!
>
> **Access:**
> ```
> ✅ https://192.168.1.10:8843  ← Works
> ❌ http://192.168.1.10:8843   ← "400 Bad Request"
> ```
>
> **In the `.env`:**
> ```bash
> SITE_URL=https://192.168.1.10:8843  ← HTTPS!
> ```

---

## 🚀 Phase 1: Preparation

### Step 1: Download Supabase from GitHub

1. **Open browser:** https://github.com/supabase/supabase
2. **Code button** (green, top right) → **Download ZIP**
3. **Save:** e.g. `C:\Temp\supabase-master.zip`
4. **Extract:** To a folder of your choice

**Result:** You have a folder `supabase-master` with many subfolders.

---

### Step 2: Generate JWT tokens

> 🔴 **CRITICAL:** This is the most important step! Without correct tokens, nothing works!

#### 2.1 Create JWT_SECRET

**Tool:** https://www.random.org/strings/

**Settings:**
- How many strings: `1`
- Length: `32` (minimum!)
- Characters: ✅ Uppercase, ✅ Lowercase, ✅ Digits
- Click **"Get Strings"**

**Example result:**
```
YOUR-JWT-SECRET-32-CHARACTERS-HERE
```

**Save** this string in a text document (e.g. `supabase-keys.txt`):
```
JWT_SECRET=YOUR-JWT-SECRET-32-CHARACTERS-HERE
```

---

#### 2.2 Generate ANON_KEY

> ⚠️ **Use the official Supabase tool:**
> 👉 https://supabase.com/docs/guides/self-hosting/docker#generate-api-keys

**On the website:**
1. **Scroll** down to the "Generate JWT" tool
2. **Enter:** Your JWT_SECRET (from step 2.1)
3. **Dropdown "Key:"** → Select **ANON_KEY**
4. **Click:** "Generate JWT"
5. **Copy** the generated token (very long, starts with `eyJhbGci...`)
6. **Save:**
   ```
   ANON_KEY=YOUR-GENERATED-ANON-KEY-TOKEN-HERE
   ```

---

#### 2.3 Generate SERVICE_ROLE_KEY

> ⚠️ **IMPORTANT:** Use the SAME JWT_SECRET as in step 2.1!

**On the same website:**
1. **JWT_SECRET:** Should still be in the input field (DON'T delete!)
2. **Dropdown "Key:"** → Select **SERVICE_ROLE_KEY**
3. **Click:** "Generate JWT"
4. **Copy** the token
5. **Save:**
   ```
   SERVICE_ROLE_KEY=YOUR-GENERATED-SERVICE-ROLE-KEY-TOKEN-HERE
   ```

**Check:** You now have 3 keys in your text document.

---

### Step 3: Generate additional passwords

**Go to:** https://www.random.org/strings/

#### 3.1 Database and system passwords (64 characters)

**Settings:**
- How many strings: `3`
- Length: `64`
- Characters: ✅ Uppercase, ✅ Lowercase, ✅ Digits

**Copy the 3 strings and name them:**
```
POSTGRES_PASSWORD=YOUR-POSTGRES-PASSWORD-64-CHARS
DASHBOARD_PASSWORD=YOUR-DASHBOARD-PASSWORD-64-CHARS
SECRET_KEY_BASE=YOUR-SECRET-KEY-BASE-64-CHARS
```

---

#### 3.2 VAULT_ENC_KEY (32 characters)

**Settings:**
- How many strings: `1`
- Length: `32` (EXACTLY 32!)
- Characters: ✅ Uppercase, ✅ Lowercase, ✅ Digits

```
VAULT_ENC_KEY=YOUR-VAULT-ENC-KEY-32-CHARS
```

---

#### 3.3 Overview of your keys

**You should now have:**
```
JWT_SECRET=...           (32 characters)
ANON_KEY=...             (very long, ~200+ characters)
SERVICE_ROLE_KEY=...     (very long, ~200+ characters)
POSTGRES_PASSWORD=...    (64 characters)
DASHBOARD_PASSWORD=...   (64 characters)
SECRET_KEY_BASE=...      (64 characters)
VAULT_ENC_KEY=...        (32 characters)
```

**Save these securely!** You'll need them soon.

---

### Step 4: Gmail SMTP (optional)

> Optional, only if you need email sending (e.g. password reset, user invitations)

#### 4.1 Create Google App Password

1. **Google Account:** https://myaccount.google.com/
2. **Security** → **2-Step Verification** (must be enabled!)
3. **App Passwords:** https://myaccount.google.com/apppasswords
4. **App:** Mail
5. **Device:** Other → Name: `Supabase`
6. **Generate**
7. **Copy** the 16-character password (e.g. `abcd efgh ijkl mnop`)

> ⚠️ **IMPORTANT:** Remove all spaces!
> ```
> abcdefghijklmnop  ← Should look like this
> ```

**Save:**
```
GMAIL_EMAIL=your-email@gmail.com
GMAIL_APP_PASSWORD=YOUR-GMAIL-APP-PASSWORD-16-CHARS
```

---

## 🗂️ Phase 2: Preparing Files on Synology

### Step 5: Create folder structure on Synology

#### 5.1 Open File Station

1. **DSM** → **File Station**
2. **Navigate to:** `docker` (main folder)

> If the `docker` folder doesn't exist, create it in the main directory.

---

#### 5.2 Create main folder

1. **Right-click** in the `docker` folder
2. **Create** → **Create folder**
3. **Name:** `supabase`
4. **OK**

**Path:** `/volume1/docker/supabase`

---

#### 5.3 Create subfolders

> ⚠️ **IMPORTANT:** Create ONLY these 2 folders! The rest comes automatically!

**In the folder `/volume1/docker/supabase/volumes/` create:**

1. **Create:** Folder `volumes`
2. **Go into** the folder `volumes`
3. **Create:**
   - Folder `db`
   - Folder `storage`
4. **Go into** the folder `db`
5. **Create:** Folder `data`

**Final structure (what YOU create):**
```
/volume1/docker/supabase/
└── volumes/
    ├── db/
    │   └── data/     ← You create this!
    └── storage/      ← You create this!
```

**After uploading GitHub files, this will be added automatically:**
```
/volume1/docker/supabase/
└── volumes/
    ├── api/          ← Comes automatically
    ├── db/
    │   ├── data/     ← Created by you
    │   ├── init/     ← Comes automatically
    │   └── ...       ← Comes automatically
    ├── functions/    ← Comes automatically
    ├── logs/         ← Comes automatically (with vector.yml)
    ├── pooler/       ← Comes automatically
    └── storage/      ← Created by you
```

> ❌ **NO** manual "logs" folder! It comes with the GitHub upload!

---

### Step 6: Upload GitHub files

#### 6.1 Find Docker files

**On your computer:**
1. **Open** the extracted folder `supabase-master`
2. **Go to:** `supabase-master/docker/`
3. **You see:**
   - `docker-compose.yml` ← Important!
   - `.env.example` ← Important!
   - `kong.yml`
   - `volumes/` (folder)
   - More files

---

#### 6.2 Upload ALL files

> ⚠️ **IMPORTANT:** Upload EVERYTHING, not just individual files!

1. **Select ALL** files and folders in the `docker/` folder
2. **File Station:** Go to `/volume1/docker/supabase/`
3. **Upload button** → Select all marked files

**Result:** The GitHub folders merge with your created folders.

**Check:** In `/volume1/docker/supabase/` you now see:
```
docker-compose.yml
.env.example
kong.yml
volumes/
  ├── api/           ← New from upload
  ├── db/
  │   ├── data/      ← Was already there
  │   ├── init/      ← New from upload
  │   └── ...
  ├── functions/     ← New from upload
  ├── logs/          ← New from upload
  └── storage/       ← Was already there
```

---

### Step 7: Create .env file

#### 7.1 Rename file

1. **File Station:** `/volume1/docker/supabase/`
2. **Right-click** on `.env.example`
3. **Rename** → `.env` (without "example")
4. **OK**

---

#### 7.2 Download and edit file

1. **Right-click** on `.env`
2. **Download**
3. **Save** to desktop

**Open** the file with a text editor:
- **Windows:** Notepad++ (recommended)
- **Mac:** TextEdit or VS Code
- ❌ **NOT:** Windows Notepad (can cause problems)

---

#### 7.3 Find out your Synology IP address

**Important for the next steps!**

**Method 1: DSM address bar**
- Look at the browser address bar
- e.g. `http://192.168.1.10:5000/` → IP is `192.168.1.10`

**Method 2: DSM → Control Panel**
- Control Panel → Network → Network Interface
- Look at "IPv4 Address"

**Example:** `192.168.1.10` (replace this with YOUR IP in the following steps!)

---

#### 7.4 Enter all passwords/keys

> 🔴 **CRITICAL:** Every mistake here leads to problems!

**Open your text document** with the generated keys from steps 2 & 3.

---

**A) JWT tokens (lines 6-9)**

**SEARCH:**
```bash
JWT_SECRET=your-super-secret-jwt-token-with-at-least-32-characters-long
```

**REPLACE:**
```bash
JWT_SECRET=YOUR-JWT-SECRET-32-CHARACTERS-HERE
```
(Your JWT_SECRET)

---

**SEARCH:**
```bash
ANON_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**REPLACE:**
```bash
ANON_KEY=YOUR-GENERATED-ANON-KEY-TOKEN-HERE
```

---

**SEARCH:**
```bash
SERVICE_ROLE_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**REPLACE:**
```bash
SERVICE_ROLE_KEY=YOUR-GENERATED-SERVICE-ROLE-KEY-TOKEN-HERE
```

---

**B) Database password**

**SEARCH:**
```bash
POSTGRES_PASSWORD=your-super-secret-and-long-postgres-password
```

**REPLACE:**
```bash
POSTGRES_PASSWORD=YOUR-POSTGRES-PASSWORD-64-CHARS
```
(Your generated 64-character password)

---

**C) Dashboard password**

**SEARCH:**
```bash
DASHBOARD_PASSWORD=this_password_is_insecure_and_should_be_updated
```

**REPLACE:**
```bash
DASHBOARD_PASSWORD=YOUR-DASHBOARD-PASSWORD-64-CHARS
```

---

**D) Additional secrets**

**SEARCH and REPLACE:**
```bash
SECRET_KEY_BASE=your-secret-key-base
→
SECRET_KEY_BASE=YOUR-SECRET-KEY-BASE-64-CHARS

VAULT_ENC_KEY=your-vault-encryption-key
→
VAULT_ENC_KEY=YOUR-VAULT-ENC-KEY-32-CHARS
```

---

#### 7.5 Replace localhost with IP address

> 🔴 **CRITICAL:** Without this, Supabase only works on Synology itself!

**Use "Find & Replace" in your editor:**

**In Notepad++:**
- **Ctrl + H**
- Find: `localhost`
- Replace: `192.168.1.10` (YOUR Synology IP!)
- **"Replace All"**

**In VS Code:**
- **Ctrl + H**
- Find: `localhost`
- Replace: `192.168.1.10`
- **"Replace All"** (icon with two arrows)

**Examples of what gets changed:**
```bash
# Before:
SITE_URL=http://localhost:3000
API_EXTERNAL_URL=http://localhost:8000

# After:
SITE_URL=https://192.168.1.10:8843
API_EXTERNAL_URL=https://192.168.1.10:8843
```

> ⚠️ **ATTENTION:** You'll need to change ports - see next step!

---

#### 7.6 Adjust ports

> ⚠️ **IMPORTANT:** Port 5432 is often occupied, port 8843 is HTTPS!

**SEARCH:**
```bash
POSTGRES_PORT=5432
```

**REPLACE:**
```bash
POSTGRES_PORT=54329
```

---

**SEARCH:**
```bash
KONG_HTTP_PORT=8000
KONG_HTTPS_PORT=8443
```

**REPLACE:**
```bash
KONG_HTTP_PORT=8800
KONG_HTTPS_PORT=8843
```

---

**SEARCH all URLs with port 3000 or 8000 and change to 8843:**

```bash
# Before:
SITE_URL=http://192.168.1.10:3000
API_EXTERNAL_URL=http://192.168.1.10:8000
SUPABASE_PUBLIC_URL=http://192.168.1.10:8000

# After:
SITE_URL=https://192.168.1.10:8843
API_EXTERNAL_URL=https://192.168.1.10:8843
SUPABASE_PUBLIC_URL=https://192.168.1.10:8843
```

> ⚠️ **Note:** `http` → `https` because port 8843 is HTTPS!

---

#### 7.7 Set up Gmail SMTP (optional)

> Only if you need email sending!

**SEARCH:**
```bash
SMTP_ADMIN_EMAIL=admin@example.com
SMTP_HOST=supabase-mail
SMTP_PORT=2500
SMTP_USER=fake_mail_user
SMTP_PASS=fake_mail_password
SMTP_SENDER_NAME=fake_sender
```

**REPLACE:**
```bash
SMTP_ADMIN_EMAIL=your-email@gmail.com
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASS=YOUR-GMAIL-APP-PASSWORD-16-CHARS
SMTP_SENDER_NAME=Supabase Self-Hosted
```
(Your Gmail App Password - WITHOUT spaces!)

---

#### 7.8 Save file

1. **Save** (Ctrl+S)
2. **Make sure:** Filename is `.env` (not `.env.txt`!)

**Check:** Your `.env` now has:
- ✅ All passwords/keys entered (as placeholders)
- ✅ No "localhost" anymore (except in comments)
- ✅ Ports: 54329, 8800, 8843
- ✅ All URLs with your Synology IP
- ✅ All URLs with HTTPS

---

### Step 8: Adjust docker-compose.yml

> ⚠️ **IMPORTANT:** Paths must be absolute for Synology Container Manager!

#### 8.1 Download file

1. **File Station:** `/volume1/docker/supabase/`
2. **Right-click** on `docker-compose.yml`
3. **Download**
4. **Open** with Notepad++ or VS Code

---

#### 8.2 Change relative paths to absolute

**Use "Find & Replace":**

**SEARCH:**
```
./volumes/
```

**REPLACE:**
```
/volume1/docker/supabase/volumes/
```

**Click:** "Replace All"

**Result:** ~14 replacements

**Example of what gets changed:**
```yaml
# Before:
volumes:
  - ./volumes/db/data:/var/lib/postgresql/data

# After:
volumes:
  - /volume1/docker/supabase/volumes/db/data:/var/lib/postgresql/data
```

---

#### 8.3 Save file

**Save** (Ctrl+S)

**Check:** No more `./volumes/` in the file (except in comments).

---

### Step 9: Upload files back to Synology

1. **File Station:** Go to `/volume1/docker/supabase/`
2. **Upload** the edited `.env` → **Confirm** overwrite
3. **Upload** the edited `docker-compose.yml` → **Confirm** overwrite

**Check:** Both files are now on Synology with all changes.

---

## 🐳 Phase 3: Installation in Container Manager

> ⚠️ **IMPORTANT:** You will perform this process by clicking "Build" **3 TIMES**!

### Step 10: Open Container Manager

1. **DSM** → **Container Manager**
2. **Left menu bar** → **Project**

---

### Step 11: Create project (1st build)

#### 11.1 New project

1. **Click:** "Create"
2. **Window opens:** "Create project"

---

#### 11.2 Project name

**Enter name:**
```
supabase
```

> ⚠️ Lowercase, no spaces!

**Path:** Will be automatically set to `/docker/supabase`

---

#### 11.3 Select source

**Choose:** "Set path"

**Click on:** "Select"

**Navigate to:** `/docker/supabase`

**Select the file:** `docker-compose.yml`

**OK**

---

#### 11.4 Web portal settings (optional)

> Optional but practical

**Web portal settings:**
- ✅ **Enable web portal via HTTP**
- **Port:** `8843`

**Result:** An icon for Supabase will appear in the DSM main menu later.

---

#### 11.5 Create project

**Click at bottom:** "Done"

**Container Manager now starts:**
- ⏳ Downloads Docker images (5-10 minutes, depending on internet)
- ⏳ Creates 13 containers
- ⏳ Starts all containers

---

### Step 12: First build run (30 minutes)

#### 12.1 View container status

**Container Manager → Container**

**Containers are now starting:**
- ⏳ Docker images are being downloaded (first few minutes)
- ⏳ Containers are created and started
- ⏳ Database initializes itself
- ⏳ SQL migrations run

**You see containers with names like:**
```
supabase-analytics
supabase-auth
supabase-db
supabase-kong
supabase-studio
... and more
```

---

#### 12.2 What happens on 1st build

**After approx. 30 minutes:**
- ✅ Some containers running ("Running" - green)
- ❌ **Project stops with error**
- ⚠️ Some containers are "Stopped" (red)

**This is completely NORMAL on first build!**

**Common errors on 1st build:**
- "Container exited with code 1"
- "Health check failed"
- Database not yet fully initialized

**Check:** ✅ Project is stopped, some containers ran briefly

---

#### 12.3 Do NOT delete!

> ⚠️ **IMPORTANT:** Do NOT delete project! Just continue to next step!

---

### Step 13: Second build run (15 minutes)

> ⚠️ **IMPORTANT:** Do NOT delete project! Just click "Build"!

#### 13.1 Start build

1. **Container Manager** → **Project**
2. **Click on** your project `supabase`
3. **Click top right on:** "Build" (or "Action" → "Build")

**What happens:**
- ⏳ Containers are restarted
- ⏳ More containers come online
- ⏳ Database is now further initialized

---

#### 13.2 Observe (15 minutes)

**After approx. 15 minutes:**
- ✅ More containers now run stable
- ✅ Database is more complete
- ❌ **Project stops again with error**

**This is still normal on 2nd build!**

**Check:** ✅ More containers ran longer than on 1st build

---

#### 13.3 Still do NOT delete!

> ⚠️ **IMPORTANT:** Still don't delete! Continue to next step!

---

### Step 14: Third build run - FINAL! (15-20 minutes)

> 🎯 **LAST build - then everything runs!**

#### 14.1 Start build for 3rd time

1. **Container Manager** → **Project**
2. **Click on** your project `supabase`
3. **Click on:** "Build"

**What happens:**
- ⏳ All containers start again
- ⏳ Database is now complete
- ⏳ All dependencies are met

---

#### 14.2 Final start (15-20 minutes)

**Now everything runs through!**

**After 15-20 minutes you see:**
```
✅ supabase-analytics      → Running
✅ supabase-auth           → Running
✅ supabase-db             → Running
✅ supabase-edge-functions → Running
✅ supabase-imgproxy       → Running
✅ supabase-kong           → Running
✅ supabase-meta           → Running
✅ supabase-pooler         → Running
✅ realtime-dev.supabase-realtime → Running
✅ supabase-rest           → Running
✅ supabase-storage        → Running
✅ supabase-studio         → Running
✅ supabase-vector         → Running
```

**All 13 containers running stable!** 🎉

**Check:** 
- ✅ All containers are green ("Running")
- ✅ No more errors
- ✅ Project runs continuously

---

#### 14.3 Summary of the 3 builds

**1st Build (30 min):**
- Database initializes
- Some containers start
- Ends with error → NORMAL!

**2nd Build (15 min):**
- More containers come online
- Database is further along
- Still ends with error → NORMAL!

**3rd Build (15-20 min):**
- ALL containers run
- Everything fully initialized
- DONE! ✅

---

## 🎉 Phase 4: Testing Supabase

### Step 15: Open dashboard

#### 15.1 Open browser

**On your PC/Mac:**

**Open:** https://192.168.1.10:8843
(Replace `192.168.1.10` with your Synology IP!)

> ⚠️ **Important:** `https` (not `http`!) and port `8843`!

---

#### 15.2 SSL certificate warning

**You'll probably see a warning:**
```
"This connection is not secure"
or
"Your connection is not private"
```

**This is normal!** Supabase uses a self-signed certificate.

**Solution:**
- **Chrome:** "Advanced" → "Proceed to 192.168.1.10 (unsafe)"
- **Firefox:** "Advanced" → "Accept the Risk and Continue"
- **Safari:** "Show Details" → "visit this website"

---

#### 15.3 Login page

**You now see:**
- Supabase Logo
- "Sign in to your project"
- Username field
- Password field

**Login:**

**Username:**
```
supabase
```

**Password:**
```
YOUR-DASHBOARD-PASSWORD-64-CHARS
```
(The password from step 3.1 that you entered in the `.env`)

**Click:** "Sign In"

---

#### 15.4 Dashboard

**You're now in the Supabase Dashboard!** 🎉

**You see:**
- Project: "Default Project"
- Left menu bar:
  - Table Editor
  - SQL Editor
  - Authentication
  - Storage
  - Functions
  - Settings

**Congratulations!** Supabase is running! 🚀

---

### Step 16: Create first table

#### 16.1 Open Table Editor

1. **Left menu bar:** "Table Editor"
2. **Click:** "Create a new table" (green button)

---

#### 16.2 Configure table

**Name:**
```
test_posts
```

**Enable Row Level Security (RLS):**
- ❌ Disabled (for testing)

**Columns:**

Click on "+ Add column" and create:

| Name | Type | Default | Primary | Unique |
|------|------|---------|---------|--------|
| id | int8 | (auto) | ✅ | ✅ |
| title | text | NULL | ❌ | ❌ |
| content | text | NULL | ❌ | ❌ |
| created_at | timestamptz | now() | ❌ | ❌ |

**Click:** "Save"

**Table is created!**

---

#### 16.3 Add test entry

1. **Click** on the table `test_posts`
2. **Click:** "Insert row" (top right, green)
3. **Fill in:**
   - title: `My first test`
   - content: `Supabase is running on my Synology!`
4. **Click:** "Save"

**Entry appears in the table!** 🎉

---

### Step 17: Test from other devices

#### 17.1 Mac/PC on same network

**On another computer on WiFi:**

1. Open browser
2. https://192.168.1.10:8843
3. Login

**Works?** ✅ Perfect!

---

## 🌐 Phase 5: Setting up Reverse Proxy (Optional)

> Optional: Access via domain instead of IP

**Advantages:**
- ✅ Nicer URL: `https://supabase.yourdomain.com`
- ✅ External SSL certificate (Let's Encrypt)
- ✅ Access from anywhere (not just LAN)

---

### Prerequisites

- Own domain (e.g. via DynDNS)
- Router port forwarding (port 443)
- Let's Encrypt certificate (via DSM)

---

### Step 19: Create DNS entry

**Depending on setup:**

**Option A: DynDNS service**
- Create an A-record: `supabase.yourdomain.com` → Your public IP

**Option B: Internal DNS (Router/Pi-hole)**
- Create an A-record: `supabase.yourdomain.com` → `192.168.1.10`

---

### Step 20: Synology Reverse Proxy

#### 20.1 Set up Reverse Proxy

**DSM → Control Panel → Login Portal → Advanced → Reverse Proxy**

**Create new rule:**

| Field | Value |
|------|------|
| Description | Supabase |
| Source - Protocol | HTTPS |
| Source - Hostname | supabase.yourdomain.com |
| Source - Port | 443 |
| Destination - Protocol | HTTPS |
| Destination - Hostname | 192.168.1.10 |
| Destination - Port | 8843 |

**Custom Header (Tab "Custom Header"):**

**Create → WebSocket:**
```
WebSocket: ✅ Enabled
```

**Add headers:**
```
X-Forwarded-For: $proxy_add_x_forwarded_for
X-Forwarded-Proto: $scheme
X-Real-IP: $remote_addr
Host: $host
```

**Save**

---

### Step 21: Domain works!

**Test access via your domain:**
```
https://supabase.yourdomain.com
```

**Should work:**
- ✅ Login page loads
- ✅ Dashboard accessible

> **Note:** For external access from outside your network, you additionally need port forwarding (port 443) in your router and a valid SSL certificate (e.g. Let's Encrypt via DSM).

---

## 🆘 Troubleshooting

### Problem 1: "Port 5432 already in use"

**Symptom:** Container doesn't start, error with port 5432

**Cause:** PostgreSQL port already in use (e.g. by Synology)

**Solution:**
1. Open `.env`
2. Change: `POSTGRES_PORT=54329` (instead of 5432)
3. Upload `docker-compose.yml`
4. Build project again

---

### Problem 2: "400 Bad Request - HTTP to HTTPS port"

**Symptom:** Browser shows "400 Bad Request" on access

**Cause:** You're trying to access HTTPS port with HTTP

**Solution:**
1. Use `https://` (not `http://`)
2. Port 8843 is HTTPS-only!
3. Check `.env`: URLs must have `https://`

---

### Problem 3: Containers crash after 3 builds

**Symptom:** Even after 3rd build, containers don't run stable

**Possible causes:**

**A) JWT tokens wrong:**
1. Check: Was Supabase tool used?
2. Check: All 3 keys generated from ONE JWT_SECRET?
3. Solution: Regenerate tokens (step 2), update `.env`

**B) Paths wrong:**
1. Check `docker-compose.yml`: Are paths absolute? (`/volume1/...`)
2. Solution: Repeat step 8

**C) Ports occupied:**
1. Check which container crashes (view logs)
2. Change ports in `.env`

---

### Problem 4: Dashboard loads but "Authentication failed"

**Symptom:** Login page loads but login doesn't work

**Cause:** DASHBOARD_PASSWORD wrong

**Solution:**
1. Check `.env`: Is `DASHBOARD_PASSWORD` correct?
2. Generate password (step 3.1)
3. Upload `.env`
4. Build project again

---

### Problem 5: Other devices cannot access

**Symptom:** Works on Synology, not on PC/phone

**Cause:** URLs in `.env` still have `localhost`

**Solution:**
1. Open `.env`
2. Find & Replace: `localhost` → `192.168.1.10` (your IP)
3. Upload `.env`
4. Build project again

---

### Problem 6: Emails are not sent

**Symptom:** "Invite user" doesn't send email

**Checks:**
1. Gmail 2FA enabled?
2. App password created?
3. Spaces removed from password?

**Solution:**
1. New app password (step 4)
2. Update `.env` (WITHOUT spaces!)
3. Build project again

---

## ❓ FAQ

### Why 3 builds?

**Answer:** Containers have dependencies:
1. **1st build:** Database must initialize first, SQL migrations run
2. **2nd build:** More containers come online, build on DB
3. **3rd build:** All dependencies met, everything runs stable

**Normal:** After 3x build everything runs stable - don't delete before that!

---

### Why not Portainer?

**Answer:** Portainer has technical issues:
- Relative paths (`./volumes/`) don't work
- `.env` files are not loaded correctly
- Complex Compose files with many services are problematic

**Synology Container Manager:**
- Native integration
- Better path support
- More stable for such projects

---

### Can I use Portainer later?

**Answer:** No, not recommended.

**Reason:** Portainer only sees containers IT ITSELF created. Containers from Synology Container Manager are not displayed.

**Alternative:** Use Container Manager for management.

---

### Why port 54329 instead of 5432?

**Answer:** Synology uses PostgreSQL internally.
- Port 5432 is often occupied
- Leads to conflicts
- 54329 is free and works

---

### Why HTTPS in .env?

**Answer:** Kong container (port 8843) runs in HTTPS mode.
- Port 8843 = HTTPS port
- Port 8800 = HTTP port
- .env must match the port

**Test confirms:**
```
http://IP:8843  → 400 Error
https://IP:8843 → ✅ Works
```

---

### Can I use other ports?

**Answer:** Yes!

**Change in `.env`:**
```bash
KONG_HTTP_PORT=8800   # Any free port
KONG_HTTPS_PORT=8843  # Any free port
POSTGRES_PORT=54329   # Any free port
```

**Important:** Also change in all URLs in `.env`!

---

### How do I backup my data?

**Answer:** Important folders:
- `/volume1/docker/supabase/volumes/db/data/` (Database)
- `/volume1/docker/supabase/volumes/storage/` (User files)

**Backup methods:**
1. Synology Hyper Backup
2. Rsync/Snapshot
3. PostgreSQL pg_dump

**Backup regularly!**

---

### How do I update Supabase?

**Answer:**

1. **Create backup** (see above)
2. **GitHub:** Download new `docker-compose.yml`
3. **Adjust paths** (step 8)
4. **Upload** to Synology
5. **Project** → Stop → Delete → Create new
6. Start containers

**Or:** Container Manager → Project → Update image

---

### Can I have multiple projects?

**Answer:** Yes!

**For each project:**
- Own folder: `/volume1/docker/supabase-project2/`
- Own `.env` with different ports
- Own project in Container Manager

**Important:** Ports must not overlap!

---

### How do I reset everything?

**Answer:**

**Complete fresh start:**
1. Container Manager → Project `supabase` → Delete
2. File Station → `/volume1/docker/supabase/` → Delete
3. Start from step 5

**Only reset data:**
1. Stop project
2. File Station → `volumes/db/data/` → Delete all files
3. Build project again

---

## 📚 Additional Resources

**Official Documentation:**
- 📖 Self-Hosting Guide: https://supabase.com/docs/guides/self-hosting
- 🔑 JWT Generator: https://supabase.com/docs/guides/self-hosting/docker#generate-api-keys
- 🐳 Docker Setup: https://supabase.com/docs/guides/self-hosting/docker

**Community:**
- 💬 Discord: https://discord.supabase.com
- 🗣️ Reddit: r/Supabase

**Development:**
- 🐙 GitHub: https://github.com/supabase/supabase
- 🔧 CLI Tool: https://github.com/supabase/cli
- 📦 Releases: https://github.com/supabase/supabase/releases

---

## 🙏 Credits

Based on:
- [Supabase Self-Hosting Docs](https://supabase.com/docs/guides/self-hosting)
- Community experiences
- Real test installations

**Special thanks to:**
- YouTube tutorial: https://www.youtube.com/watch?v=BMgFmyx9LdI&t

---

## 📄 License

This guide is licensed under **CC BY-SA 4.0**.

**You may:**
- ✅ Share & redistribute
- ✅ Adapt & modify
- ✅ Use commercially

**Under the following conditions:**
- 📝 Attribution
- 🔄 Share under same conditions

---

**🎉 Good luck with your Supabase installation! 🚀**
