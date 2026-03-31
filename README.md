# Supabase Self-Hosted Production Installer

🚀 **Complete production-ready Supabase installation script**

[![Version](https://img.shields.io/badge/version-3.25-blue.svg)](https://github.com/Igor-Nersisyan/supabase-installer)
[![Ubuntu](https://img.shields.io/badge/Ubuntu-20.04%20|%2022.04%20|%2024.04-orange.svg)](https://ubuntu.com/)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

## 🎯 What is this?

A battle-tested installer that deploys self-hosted Supabase with enterprise features and critical production fixes.

**Why use this installer?**
Standard Supabase guides often result in memory leaks, broken Edge Functions (DNS issues), upload limits, and SSL certificates that expire silently. This installer fixes all known issues out of the box.

## ✨ Key Features

### 📦 10GB File Upload Support (v3.15+)
- **Problem solved:** Standard configs use string values for size limits, which causes failures.
- **Our fix:** Applies correct **Integer** values for `FILE_SIZE_LIMIT` and `UPLOAD_FILE_SIZE_LIMIT`.
- **Full stack support:** Nginx (11GB), Kong (11GB), and Storage (10GB) are perfectly synced.
- **TUS Protocol:** Full support for resumable uploads with direct Storage bypass (v3.22).

### 🔄 TUS Resumable Upload Fix (v3.22)
- Storage container exposed on port 5000 for direct TUS access
- Nginx routes TUS requests directly to Storage (bypassing Kong)
- `TUS_URL_PATH` corrected to `/upload/resumable` (no `/storage/v1` prefix)
- CORS headers handled by Nginx for TUS endpoints
- No separate fix script needed — works out of the box

### ⚡ Performance & Stability
- **Memory Optimized:** Analytics container optimized to use ~450MB instead of 1.5GB+.
- **Edge Functions Stability:** Docker Daemon configured with `dns-opts=["ndots:0"]` to prevent DNS resolution failures after restarts.
- **Kong Timeouts:** Increased to **5 minutes** (300s) to support long-running AI/n8n workflows (default is 60s). Uses pure sed injection (v3.25) — no PyYAML corruption.

### 🛡️ Enterprise Security
- **Database Hardening (v3.3):** Includes a script to whitelist IPs or restrict access to Docker-only.
- **Auto SSL:** Let's Encrypt certificate with triple-layer auto-renewal (v3.24+).
- **Log Rotation:** Docker logs limited to 10MB/file to prevent disk overflow.

### ✉️ Email Templates (v3.18)
- Templates stored as HTML files in `email_templates/` directory
- Served via dedicated nginx template-server container
- Easy to edit and customize — just edit HTML and restart auth
- Available variables: `{{ .ConfirmationURL }}`, `{{ .Email }}`, `{{ .Token }}`, `{{ .SiteURL }}`

### 🔑 Google OAuth (v3.19)
- Pre-configured environment variables in `.env`
- Just add your Google Client ID and Secret from Google Cloud Console
- Restart auth service to enable

### 📡 Protected Webhooks with Streaming (v3.21)
- Three protected webhook endpoints (`webhook-endpoint-1/2/3`)
- Support for JSON and file uploads (multipart/form-data)
- SSE streaming support for AI agent responses (`?stream=true`)
- User authentication and rate limiting built-in

### 🔒 SSL Auto-Renewal (v3.24+)
- **`--deploy-hook`** saved in certbot renewal config at first certificate generation
- **Ubuntu systemd timer** (`certbot.timer`) uses saved `renew_hook` automatically
- **Cron fallback** — daily renewal check at 2 AM as backup
- **Nginx acme-challenge** location ensures webroot verification works

### 🔧 Kong Timeout Fix (v3.25)
- Replaced PyYAML-based kong.yml modification with pure sed
- PyYAML was corrupting kong.yml formatting (stripping comments, breaking indentation), causing Kong to fail to start
- Pure sed preserves original file structure and only adds timeout values

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
wget https://raw.githubusercontent.com/igornersisian/supabase-installer/main/install-supabase.sh

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
3. **Reset to open:** Allow all connections.
4. **View current status:** Check existing rules.
5. **Add IP:** Add more trusted servers later.

### 3. Customize Email Templates
```bash
nano /opt/supabase-project/email_templates/confirmation.html
# Edit HTML files, then restart:
cd /opt/supabase-project && docker compose restart auth
```

### 4. Enable Google OAuth (Optional)
```bash
nano /opt/supabase-project/.env
# Set GOOGLE_ENABLED=true, add CLIENT_ID & SECRET
cd /opt/supabase-project && docker compose restart auth
```

## 🧩 Included Edge Functions

The installer pre-deploys useful functions:

1.  **`n8n-proxy`**: A public entry point for n8n webhooks.
    - URL: `https://your-domain.com/functions/v1/n8n-proxy`
    - Requires `N8N_WEBHOOK_URL` in `.env`.
2.  **`webhook-endpoint-1/2/3`**: Protected endpoints requiring Auth Bearer token.
    - Support JSON and file uploads
    - SSE streaming with `?stream=true`
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

# Restart only auth (after email template changes)
docker compose restart auth

# Check logs
docker logs supabase-edge-functions --tail 50
docker logs supabase-auth --tail 50
docker logs supabase-storage --tail 50
docker logs supabase-kong --tail 50
```

### Verify 10GB Upload Support
```bash
docker exec supabase-storage printenv | grep -iE "size|tus"
# Should show:
# FILE_SIZE_LIMIT=10737418240
# UPLOAD_FILE_SIZE_LIMIT=10737418240
# UPLOAD_FILE_SIZE_LIMIT_STANDARD=10737418240
# TUS_URL_PATH=/upload/resumable
```

### Verify SSL Auto-Renewal
```bash
# Check certbot renewal config has deploy hook
grep renew_hook /etc/letsencrypt/renewal/your-domain.conf

# Check systemd timer is active
systemctl list-timers | grep certbot

# Check cron fallback
crontab -l | grep certbot

# Test renewal (dry run)
certbot renew --dry-run
```

### Configuration
Edit `/opt/supabase-project/.env` to change:
- SMTP Settings (emails)
- Webhook URLs (N8N_WEBHOOK_URL, ENDPOINT_1/2/3_WEBHOOK_URL)
- Google OAuth credentials
- JWT Secrets

## 📊 Resource Usage
Typical consumption after optimization:
- **Total RAM**: ~2.5 - 3.0 GB
- **Disk**: ~2 GB (base installation)

## 🐛 Troubleshooting

### Kong fails to start
**Symptom:** `container supabase-kong is unhealthy`, logs show `did not find expected key`

This was caused by PyYAML corrupting kong.yml formatting in versions prior to v3.25. Fixed in v3.25 by using pure sed for timeout injection. If you hit this on an older version:
```bash
cd /opt/supabase-project
cp volumes/api/kong.yml.template volumes/api/kong.yml
source .env
sed -i "s|\$SUPABASE_ANON_KEY|$ANON_KEY|g" volumes/api/kong.yml
sed -i "s|\$SUPABASE_SERVICE_KEY|$SERVICE_ROLE_KEY|g" volumes/api/kong.yml
sed -i "s|\$DASHBOARD_USERNAME|supabase|g" volumes/api/kong.yml
sed -i "s|\$DASHBOARD_PASSWORD|$DASHBOARD_PASSWORD|g" volumes/api/kong.yml
sed -i '/url: http:\/\/functions:9000\//a\    connect_timeout: 300000\n    write_timeout: 300000\n    read_timeout: 300000' volumes/api/kong.yml
docker rm -f supabase-kong && docker compose up -d
```

### Realtime health check returns 404
This is normal. The health endpoint might be hidden, but WebSocket connections will work. Test with a real client.

### Edge Functions timeout after 60s
This installer sets timeouts to 300s. If you see timeouts, verify:
```bash
grep -A 5 "name: functions-v1" /opt/supabase-project/volumes/api/kong.yml
```

### Database connection failed from external IP
Check if you ran `harden_supabase_db.sh`. If so, add your IP using option 5.

### SSL certificate expired
```bash
# Check certificate status
certbot certificates

# Check renewal config
grep renew_hook /etc/letsencrypt/renewal/your-domain.conf

# Force renewal
certbot renew --force-renewal

# Reload nginx to pick up new cert
systemctl reload nginx
```

## 📝 Changelog

### v3.25 (Latest)
- **Fixed Kong startup failure**: Replaced PyYAML-based kong.yml modification with pure sed. PyYAML was stripping comments and breaking YAML indentation, causing Kong to fail parsing its config.

### v3.24
- **Fixed SSL auto-renewal**: Added `--deploy-hook "systemctl reload nginx"` to initial `certbot certonly` command. Certbot saves this in the renewal config, so Ubuntu's systemd timer automatically reloads nginx after certificate renewal.

### v3.23
- Added certbot renewal cron job with nginx reload deploy-hook as fallback

### v3.22
- Integrated TUS resumable upload fix (direct Storage bypass, no separate fix needed)
- `TUS_URL_PATH` corrected, `TUS_URL_SCHEME` and `TUS_URL_PORT` added
- Storage port 5000 exposed for direct TUS access

### v3.21
- Fixed streaming — direct body proxy instead of TransformStream (no early termination)

### v3.20
- Protected webhook endpoints now support streaming (SSE) responses

### v3.19
- Added Google OAuth configuration via `.env`

### v3.18
- Email templates via nginx template-server container

### v3.15
- Fixed 10GB upload support — integer values instead of strings for all size limits

## 🤝 Contributing

Found a bug or have a suggestion? Please open an issue or submit a PR!

## 📄 License

MIT License - feel free to use in your projects!

## ⭐ Support

If this script saved you time, consider giving it a star on GitHub!

---

**Note**: This is an independent project, not officially affiliated with Supabase.
