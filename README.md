# Supabase Self-Hosted Production Installer

🚀 **Complete production-ready Supabase installation script**

[![Version](https://img.shields.io/badge/version-3.18-blue.svg)](https://github.com/Igor-Nersisyan/supabase-installer)
[![Ubuntu](https://img.shields.io/badge/Ubuntu-20.04%20|%2022.04%20|%2024.04-orange.svg)](https://ubuntu.com/)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

## 🎯 What is this?

A battle-tested installer that deploys self-hosted Supabase with enterprise features and critical production fixes.

**Why use this installer?**
Standard Supabase guides often result in memory leaks, broken Edge Functions (DNS issues), and upload limits. This installer fixes all known issues out of the box.

## ✨ Key Features (v3.15)

### 📦 10GB File Upload Support (Fixed)
- **Problem solved:** Standard configs use string values for size limits, which causes failures.
- **Our fix:** Applies correct **Integer** values for `FILE_SIZE_LIMIT` and `UPLOAD_FILE_SIZE_LIMIT`.
- **Full stack support:** Nginx (11GB), Kong (11GB), and Storage (10GB) are perfectly synced.
- **TUS Protocol:** Full support for resumable uploads.

### ⚡ Performance & Stability
- **Memory Optimized:** Analytics container optimized to use ~450MB instead of 1.5GB+.
- **Edge Functions Stability:** Docker Daemon configured with `dns-opts=["ndots:0"]` to prevent DNS resolution failures after restarts.
- **Kong Timeouts:** Increased to **5 minutes** (300s) to support long-running AI/n8n workflows (default is 60s).

### 🛡️ Enterprise Security
- **Database Hardening (v3.3):** Includes a script to whitelist IPs or restrict access to Docker-only.
- **Auto SSL:** Let's Encrypt certificate with auto-renewal.
- **Log Rotation:** Docker logs limited to 10MB/file to prevent disk overflow.

## 📋 Requirements

- **OS**: Ubuntu 20.04, 22.04, or 24.04
- **RAM**: Minimum 3GB (4GB+ recommended)
- **CPU**: 2+ cores
- **Storage**: 20GB+ free space
- **Ports**: 80, 443, 5432 (PostgreSQL)

## 🚀 Quick Install

Run as root:

```bash
# Download installer
wget https://raw.githubusercontent.com/Igor-Nersisyan/supabase-installer/main/install-supabase.sh

# Make executable
chmod +x install-supabase.sh

# Run as root
sudo ./install-supabase.sh
```

You'll be prompted for:
- Domain name (e.g., `supabase.example.com`)
- Email address (for SSL)

## 📁 Post-Installation

### 1. Credentials
All passwords and API keys are saved in a secured file:
```bash
cat /root/supabase-credentials.txt
```
*Save these securely and delete the file afterwards.*

### 2. Database Hardening (Critical)
By default, PostgreSQL (port 5432) is open. Run the hardening script to secure it:

```bash
bash /root/harden_supabase_db.sh
```
**Options:**
1. **Same Server:** Restricts access to internal Docker network only (for local n8n).
2. **Different Server:** Whitelists specific external IPs.
3. **Add IP:** Add more trusted servers later.

## 🧩 Included Edge Functions

The installer pre-deploys useful functions:

1.  **`n8n-proxy`**: A public entry point for n8n webhooks.
    *   URL: `https://your-domain.com/functions/v1/n8n-proxy`
    *   Requires `N8N_WEBHOOK_URL` in `.env`.
2.  **`webhook-endpoint-1/2/3`**: Protected endpoints requiring Auth Bearer token.
3.  **`hello`**: Simple health check function.

## 🛠️ Management

### Service Control
```bash
cd /opt/supabase-project

# Check status
docker compose ps

# Restart all services
docker compose down && docker compose up -d

# Restart only functions (faster)
docker compose restart functions
```

### Checking 10GB Upload Support
You can verify the configuration works by checking the container environment:
```bash
docker exec supabase-storage printenv | grep LIMIT
# Should show:
# FILE_SIZE_LIMIT=10737418240
# UPLOAD_FILE_SIZE_LIMIT=10737418240
```

### Configuration
Edit `/opt/supabase-project/.env` to change:
- SMTP Settings (emails)
- Webhook URLs
- JWT Secrets

## 📊 Resource Usage
Typical consumption after optimization:
- **Total RAM**: ~2.5 - 3.0 GB
- **Disk**: ~2 GB (base installation)

## 🐛 Troubleshooting

**"Realtime health check returns 404"**
This is normal. The health endpoint might be hidden, but WebSocket connections will work. Test with a real client.

**"Edge Functions timeout after 60s"**
This installer sets timeouts to 300s. If you see timeouts, check `volumes/api/kong.yml`.

**"Database connection failed from external IP"**
Check if you ran `harden_supabase_db.sh`. If so, add your IP using option 5.

## 📄 License
MIT
