# Domain Configuration — Deployment Guide

> **Your setup:** Hostinger VPS (`156.67.214.104`), Ubuntu, Nginx, Certbot, PM2, MongoDB  
> **Domains:** `app.tainc.org` (frontend), `server.tainc.org` (backend API)  
> **Backend port:** `5005` (production)

This guide walks you through deploying the Custom Domain feature on your existing Hostinger VPS. Follow every step **in order**.

---

## Table of Contents

1. [Overview — How It Works](#1-overview--how-it-works)
2. [Step 1 — Add Environment Variables](#2-step-1--add-environment-variables)
3. [Step 2 — Deploy the Updated Code](#3-step-2--deploy-the-updated-code)
4. [Step 3 — Seed the Feature Permissions](#4-step-3--seed-the-feature-permissions)
5. [Step 4 — Install the Certbot Wrapper Script](#5-step-4--install-the-certbot-wrapper-script)
6. [Step 5 — Configure Nginx Minimal Catch-All](#6-step-5--configure-nginx-minimal-catch-all)
7. [Step 6 — How Per-Domain Nginx Configs Work (Automatic)](#7-step-6--how-per-domain-nginx-configs-work-automatic)
8. [Step 7 — Open Firewall Ports](#8-step-7--open-firewall-ports)
9. [Step 8 — Restart Everything](#9-step-8--restart-everything)
10. [Step 9 — Test the Full Flow](#10-step-9--test-the-full-flow)
11. [How Orgs Use It (End-User Flow)](#11-how-orgs-use-it-end-user-flow)
12. [Troubleshooting](#12-troubleshooting)

---

## 1. Overview — How It Works

```
┌──────────────────────────────────────────────────────────────────┐
│  Customer's DNS Provider (GoDaddy, Cloudflare, etc.)             │
│  A Record: mybusiness.com → 156.67.214.104                       │
└──────────────────────┬───────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│  Nginx (VPS: 156.67.214.104)                                     │
│                                                                   │
│  1. Known domains → static config (app.tainc.org, server.tainc)  │
│  2. Verified custom domains → individual per-domain server blocks │
│     (auto-created by certbot-wrapper.sh on SSL provisioning)      │
│  3. Unknown domains → minimal catch-all (returns 444 / drops)     │
└──────────────────────┬───────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│  React Frontend (loaded in browser)                               │
│                                                                   │
│  1. Detects "mybusiness.com" is NOT a known hostname              │
│  2. Calls GET /api/v1/custom-domain/resolve?domain=mybusiness.com │
│  3. Backend returns { orgId, orgName, domain }                    │
│  4. Frontend stores orgId in sessionStorage                       │
│  5. Shows the org's login page / portal                           │
└──────────────────────────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│  Express Backend (server.tainc.org:5005)                          │
│                                                                   │
│  • CORS: allows all verified custom domains dynamically           │
│  • DNS verify: node dns.resolve4() checks A record                │
│  • Auto SSL: calls certbot-wrapper.sh on successful verification  │
│  • Scheduler: re-checks active domains every 2 hours              │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2. Step 1 — Add Environment Variables

SSH into your VPS and edit the backend `.env` file:

```bash
ssh root@156.67.214.104
cd /var/www/tainc/tainc-backend    # adjust to your actual path
nano .env
```

Add these lines at the bottom:

```env
# ─── Domain Configuration ───
SERVER_IP=156.67.214.104
SSL_CERT_EMAIL=support@tainc.org
CERTBOT_WRAPPER_PATH=/usr/local/bin/certbot-wrapper.sh
```

Save and exit (`Ctrl+X`, then `Y`, then `Enter`).

---

## 3. Step 2 — Deploy the Updated Code

```bash
cd /var/www/tainc    # your project root on VPS
git pull origin main # or your production branch
```

### Build & restart the backend

```bash
cd tainc-backend
npm install
npm run build
pm2 restart tainc-backend    # use your actual PM2 process name
```

> **Find your PM2 process name:** `pm2 list`

### Build & deploy the frontend

```bash
cd ../tainc-frontend
npm install
npm run build
```

The build output goes to `dist/`. Nginx serves it from there.

---

## 4. Step 3 — Seed the Feature Permissions

The "Domain Configuration" feature needs to exist in the database so ADMIN / ORG_ADMIN users can see the sidebar item and access the page.

### Option A: Call the seeder API (recommended)

From your local machine or using `curl` on the VPS:

```bash
curl -X POST https://crmbackend.octopi-digital.com/api/v1/seeder/execute \
  -H "Content-Type: application/json" \
  -d '{
    "operation": "seed-feature-model",
    "secretKey": "change-me-in-production"
  }'
```

> Replace `YOUR_ADMIN_SECRET_KEY` with the value of `ADMIN_SECRET_KEY` in your `.env` file.  
> If you never set one, the default is `change-me-in-production` — you should change it!

**Expected response:**

```json
{
  "success": true,
  "message": "Feature model seeded successfully",
  "data": {
    "inputCount": 18,
    "normalizedCount": 18,
    "upsertedCount": 1,
    "modifiedCount": 17,
    "matchedCount": 17,
    "usedDefaultSeed": true
  }
}
```

The `upsertedCount: 1` means the new "Domain Configuration" feature was created. Existing features are just matched/updated (idempotent).

### Then seed the role permissions

```bash
# Seed SUPER_ADMIN permissions
curl -X POST https://crmbackend.octopi-digital.com/api/v1/seeder/execute \
  -H "Content-Type: application/json" \
  -d '{
    "operation": "seed-superadmin-features",
    "secretKey": "change-me-in-production"
  }'

# Seed ADMIN permissions (for all orgs)
curl -X POST https://crmbackend.octopi-digital.com/api/v1/seeder/execute \
  -H "Content-Type: application/json" \
  -d '{
    "operation": "seed-admin-features",
    "secretKey": "change-me-in-production"
  }'
```

After this, the **Domain** sidebar item will appear for ADMIN and ORG_ADMIN users.

---

## 5. Step 4 — Install the Certbot Wrapper Script

Since everything runs as **root** on your VPS, this is straightforward — no sudoers rules needed.

### 5.1 Copy the script to a system-wide location

```bash
cp /var/www/tainc/tainc-backend/scripts/certbot-wrapper.sh /usr/local/bin/certbot-wrapper.sh
chmod 755 /usr/local/bin/certbot-wrapper.sh
```

### 5.2 Verify it works

```bash
# Should print usage info
/usr/local/bin/certbot-wrapper.sh
# Output: Usage: /usr/local/bin/certbot-wrapper.sh {provision|reload|renew} [domain] [email]

# Test nginx reload
/usr/local/bin/certbot-wrapper.sh reload
# Output: >>> Reloading Nginx... >>> Nginx reloaded.
```

That's it. Since PM2 and Node run as root, the script can call `certbot` and `nginx` directly.

---

## 6. Step 5 — Configure Nginx Minimal Catch-All

You need a **minimal catch-all** that drops requests for unknown/unverified domains. Each verified custom domain gets its **own individual Nginx config file** — created automatically by the certbot-wrapper when SSL is provisioned.

> **Why not one big catch-all?** Putting all domains under a single `server_name _` block breaks SSL — you can't serve different certificates for different domains from one server block. The per-domain approach gives each domain its own server block + its own SSL cert.

### 6.1 Check your existing configs

```bash
ls -la /etc/nginx/sites-available/
ls -la /etc/nginx/sites-enabled/
```

You should see something like:
- `app.tainc.org.conf` — serves the React frontend
- `server.tainc.org.conf` — proxies to Express backend on port 5005

### 6.2 Create the minimal catch-all config

```bash
nano /etc/nginx/sites-available/custom-domain-catchall.conf
```

Paste:

```nginx
# ──────────────────────────────────────────────────────
# Minimal catch-all: drops requests for any domain that
# points to this VPS but has no dedicated server block.
# Verified custom domains get their own individual config
# files — auto-created by certbot-wrapper.sh.
# ──────────────────────────────────────────────────────
server {
    listen 80 default_server;
    server_name _;

    location / {
        return 444;
    }
}
```

> **What does `return 444` do?** It's an Nginx-specific status that closes the connection immediately without sending a response. This protects against random scanners hitting your IP.

### 6.3 Remove `default_server` from existing configs

Only ONE server block can have `default_server`. Check your existing configs:

```bash
grep -r "default_server" /etc/nginx/sites-available/
```

If any existing config has `default_server`, remove it from that file:

```bash
# Example: edit whichever file has it
nano /etc/nginx/sites-available/app.tainc.org.conf
# Change "listen 80 default_server;" to just "listen 80;"
```

### 6.4 Enable the catch-all and test

```bash
ln -sf /etc/nginx/sites-available/custom-domain-catchall.conf /etc/nginx/sites-enabled/

# Test for syntax errors
nginx -t

# If OK, reload
systemctl reload nginx
```

### 6.5 Verify the catch-all works

```bash
# Test with a fake domain — should drop the connection
curl -v -H "Host: unknown-domain.com" http://127.0.0.1/
# Expected: "Empty reply from server" (connection closed, 444)
```

---

## 7. Step 6 — How Per-Domain Nginx Configs Work (Automatic)

You do **NOT** need to create Nginx configs for each custom domain manually. The system does it automatically when a domain is verified from the dashboard.

### 7.1 What happens when an org verifies a domain

```
Org clicks "Verify" in Domain → Configuration
              │
              ▼
Backend verifies DNS (A record = 156.67.214.104)
              │
              ▼
Backend calls: certbot-wrapper.sh provision mybusiness.com support@tainc.org
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│  certbot-wrapper.sh does 3 things:                          │
│                                                             │
│  1. Creates /etc/nginx/sites-available/mybusiness.com.conf  │
│     (individual HTTP server block serving the React SPA)    │
│                                                             │
│  2. Symlinks it to /etc/nginx/sites-enabled/                │
│     and reloads nginx                                       │
│                                                             │
│  3. Runs: certbot --nginx -d mybusiness.com --redirect      │
│     (obtains SSL cert & injects HTTPS into the config)      │
└─────────────────────────────────────────────────────────────┘
              │
              ▼
Now mybusiness.com has:
  - /etc/nginx/sites-available/mybusiness.com.conf (with SSL)
  - /etc/letsencrypt/live/mybusiness.com/ (cert files)
  - Auto-redirect HTTP → HTTPS
```

### 7.2 What a per-domain config looks like (auto-generated)

After the certbot-wrapper runs, the Nginx config for `mybusiness.com` looks like:

```nginx
# Before certbot (created by certbot-wrapper.sh):
server {
    listen 80;
    server_name mybusiness.com;

    root /var/www/tainc/tainc-frontend/dist;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    gzip on;
    gzip_types text/plain text/css application/json application/javascript
               text/xml application/xml application/xml+rss text/javascript;
}

# After certbot --nginx --redirect runs, it becomes:
# (certbot adds the 443 block + redirect automatically)
```

### 7.3 What about API calls from the custom domain?

The React frontend calls the backend at `server.tainc.org` directly (configured in the frontend's environment config). The per-domain Nginx config only needs to serve the React SPA — no API proxy needed.

The backend has dynamic CORS that automatically allows all verified custom domains, so `mybusiness.com` can make API calls to `server.tainc.org` without issues.

### 7.4 What about domain removal?

When an org removes a domain from the dashboard, the backend automatically:
1. Removes `/etc/nginx/sites-available/mybusiness.com.conf`
2. Removes `/etc/nginx/sites-enabled/mybusiness.com.conf`
3. Revokes the SSL certificate
4. Cleans up any stale references
5. Reloads Nginx

### 7.5 Auto-repair

The backend has built-in auto-repair (`autoRepairNginx`) that runs before every SSL provisioning. It:
- Detects broken Nginx configs (missing cert files, stale references)
- Removes configs for domains with deleted SSL certs
- Recreates the catch-all if it's missing
- Retries up to 5 times until `nginx -t` passes

---

## 8. Step 7 — Open Firewall Ports

Your firewall should allow HTTP (80) and HTTPS (443) for Certbot and custom domains:

```bash
sudo ufw status

# If not already allowed:
sudo ufw allow 'Nginx Full'    # opens 80 + 443
sudo ufw allow 'OpenSSH'       # make sure SSH stays open!
sudo ufw enable                 # only if not already enabled
```

---

## 9. Step 8 — Restart Everything

```bash
# Restart PM2 backend
pm2 restart tainc-backend

# Reload Nginx
systemctl reload nginx

# Verify PM2 is healthy
pm2 status
pm2 logs tainc-backend --lines 20
```

Look for these lines in the PM2 logs:
```
✅ Custom domain CORS cache loaded: X domains
```

---

## 10. Step 9 — Test the Full Flow

### 9.1 Test with your own domain

Pick a test domain you own (e.g., `test.yourdomain.com`).

**A) Add DNS record at your domain registrar:**

| Type | Host | Value | TTL |
|------|------|-------|-----|
| A | test | 156.67.214.104 | 300 |

**B) Wait 2-5 minutes for DNS to propagate. Verify:**

```bash
dig +short test.yourdomain.com
# Should return: 156.67.214.104

# Or from any machine:
nslookup test.yourdomain.com
```

**C) Log in to your tainc app at `https://app.tainc.org`**

1. Go to **Domain** in the sidebar (under Settings)
2. Click on the **Configuration** tab
3. Enter `test.yourdomain.com` and click **Add Domain**
4. Click **Verify** next to the domain
5. If DNS is correct → status becomes **Active**, SSL starts provisioning

**D) Test the custom domain in a browser:**

```
http://test.yourdomain.com
```

After Certbot runs (automatic on verification), it becomes:

```
https://test.yourdomain.com
```

### 9.2 Test the domain resolver API directly

```bash
curl "https://server.tainc.org/api/v1/custom-domain/resolve?domain=test.yourdomain.com"
```

Expected:
```json
{
  "success": true,
  "data": {
    "orgId": "...",
    "orgName": "Your Org Name",
    "domain": "test.yourdomain.com"
  }
}
```

---

## 11. How Orgs Use It (End-User Flow)

Share these instructions with your org admins:

### For your customers (Org Admins):

1. **Go to your domain provider** (GoDaddy, Namecheap, Cloudflare, etc.)
2. **Add an A record:**

   | Type | Host | Points to |
   |------|------|-----------|
   | A | @ (or subdomain) | `156.67.214.104` |

3. **Wait 5-10 minutes** for DNS propagation
4. **Log in** to your tainc dashboard
5. Go to **Domain → Configuration**
6. **Add your domain** and click **Verify**
7. Once verified, SSL is provisioned automatically
8. Your custom domain is now live!

---

## 12. Troubleshooting

### "Domain verification failed"

```bash
# Check DNS from the VPS itself
dig +short mybusiness.com
# Must return 156.67.214.104

# If it returns nothing — DNS hasn't propagated yet, wait longer
# If it returns a different IP — the A record points to the wrong IP
```

### SSL provisioning failed

```bash
# Check certbot logs
cat /var/log/letsencrypt/letsencrypt.log | tail -50

# Manual test
/usr/local/bin/certbot-wrapper.sh provision mybusiness.com support@tainc.org

# Common issues:
# 1. Port 80 blocked → sudo ufw allow 80
# 2. DNS not pointing here → certbot validation fails
# 3. Rate limit → Let's Encrypt allows 5 certs per domain per week
```

### Custom domain loads but shows blank page

```bash
# Check that the per-domain Nginx config exists and has the correct root path
cat /etc/nginx/sites-available/mybusiness.com.conf
# The "root" directive should point to your frontend dist folder

ls /var/www/tainc/tainc-frontend/dist/index.html
# If file not found → update the FRONTEND_ROOT in certbot-wrapper.sh and re-provision
```

### Custom domain not loading at all (connection refused / timeout)

```bash
# Check if a per-domain config was created
ls /etc/nginx/sites-available/mybusiness.com.conf
ls /etc/nginx/sites-enabled/mybusiness.com.conf

# If missing → the domain wasn't verified yet, or SSL provisioning failed
# Re-verify the domain from the dashboard

# Check nginx is healthy
nginx -t
systemctl status nginx
```

### Custom domain loads app but doesn't resolve the org

```bash
# Test the resolve endpoint
curl "https://server.tainc.org/api/v1/custom-domain/resolve?domain=mybusiness.com"

# If 404 → the domain wasn't added/verified in the dashboard
# If CORS error → clear browser cache, the domain should be in the CORS cache
```

### PM2 process keeps restarting

```bash
pm2 logs tainc-backend --err --lines 50
# Look for the error message
```

### CORS errors on custom domain

The backend auto-caches active domains in memory for CORS. If you added a domain and it's verified but still getting CORS errors:

```bash
# Restart PM2 to reload the cache
pm2 restart tainc-backend
```

### Check all active custom domains in the DB

```bash
# On VPS mongosh:
mongosh "mongodb://127.0.0.1:27017/taincDB?replicaSet=rs0"

db.customdomains.find({ isDeleted: false }).pretty()

# Check active domains only:
db.customdomains.find({ status: "active", isDeleted: false }).pretty()
```

### Certbot auto-renewal

Certbot sets up a systemd timer automatically. Verify:

```bash
systemctl list-timers | grep certbot
# Should show: certbot.timer

# Test dry-run:
certbot renew --dry-run
```

---

## Quick Reference — File Locations on VPS

| What | Path |
|------|------|
| Backend code | `/var/www/tainc/tainc-backend/` |
| Frontend build | `/var/www/tainc/tainc-frontend/dist/` |
| Backend `.env` | `/var/www/tainc/tainc-backend/.env` |
| Certbot wrapper | `/usr/local/bin/certbot-wrapper.sh` |
| Sudoers rule | Not needed (running as root) |
| Nginx catch-all | `/etc/nginx/sites-available/custom-domain-catchall.conf` |
| Nginx per-domain configs | `/etc/nginx/sites-available/{domain}.conf` (auto-created) |
| Nginx main site | `/etc/nginx/sites-available/app.tainc.org.conf` |
| Nginx backend proxy | `/etc/nginx/sites-available/server.tainc.org.conf` |
| Certbot logs | `/var/log/letsencrypt/letsencrypt.log` |
| PM2 logs | `pm2 logs tainc-backend` |

---

## Quick Reference — API Endpoints

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/api/v1/custom-domain/resolve?domain=X` | None (public) | Resolve domain → orgId |
| `POST` | `/api/v1/custom-domain/` | ADMIN, ORG_ADMIN | Add a new domain |
| `GET` | `/api/v1/custom-domain/` | ADMIN, ORG_ADMIN | List org's domains |
| `POST` | `/api/v1/custom-domain/verify/:domainId` | ADMIN, ORG_ADMIN | Verify DNS & provision SSL |
| `DELETE` | `/api/v1/custom-domain/:domainId` | ADMIN, ORG_ADMIN | Remove a domain |
| `POST` | `/api/v1/seeder/execute` | Secret key | Seed features (body: `{ operation, secretKey }`) |
