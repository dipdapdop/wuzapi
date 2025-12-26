# Deploying WuzAPI to Fly.io

This guide covers deploying WuzAPI to [Fly.io](https://fly.io), a platform optimized for applications requiring persistent connections, stateful sessions, and regional deployment.

**Note:** This guide uses the actual configuration from our deployment (`wuzapi-shy-river-5375` in Sydney region with custom domain `wuzapi.gasbridge.nz`). Adapt the specific values for your deployment.

## Table of Contents

- [Why Fly.io?](#why-flyio)
- [Prerequisites](#prerequisites)
- [Initial Setup](#initial-setup)
- [Database Configuration](#database-configuration)
- [Secrets Management](#secrets-management)
- [Custom Domain Setup](#custom-domain-setup)
- [GitHub Actions CI/CD](#github-actions-cicd)
- [Troubleshooting](#troubleshooting)
- [Maintenance](#maintenance)

## Why Fly.io?

WuzAPI is an excellent fit for Fly.io because:

- **Persistent WebSocket Connections**: WhatsApp requires long-lived connections that Fly.io handles natively
- **Stateful Applications**: Fly.io supports persistent volumes and databases for session management
- **Regional Deployment**: Deploy close to your users for optimal performance
- **Native PostgreSQL Support**: Managed or self-hosted Postgres options
- **Affordable**: Starting at ~$5-10/month for production workloads
- **Full VM Control**: Run background processes and maintain state

## Prerequisites

1. **Fly.io Account**
   - Sign up at [fly.io](https://fly.io/app/sign-up)
   - Free tier available for testing

2. **Flyctl CLI**
   ```bash
   # macOS/Linux
   curl -L https://fly.io/install.sh | sh

   # Windows (PowerShell)
   pwsh -Command "iwr https://fly.io/install.ps1 -useb | iex"
   ```

3. **Authentication**
   ```bash
   flyctl auth login
   ```

4. **Git Repository**
   - For automated deployments via GitHub Actions

## Initial Setup

### 1. Initialize Fly Application

```bash
# From your wuzapi directory
flyctl launch

# Follow the prompts:
# - Choose an app name (e.g., 'wuzapi-shy-river-5375')
# - Select a region: 'syd' for Sydney, Australia
# - Don't add PostgreSQL yet (we'll do it manually)
# - Don't deploy yet
```

This creates a `fly.toml` configuration file.

### 2. Configure fly.toml

The deployed configuration (`fly.toml`):

```toml
app = 'wuzapi-shy-river-5375'
primary_region = 'syd'  # Sydney, Australia

[build]

[env]
  # Non-sensitive environment variables
  WEBHOOK_FORMAT = "json"
  SESSION_DEVICE_NAME = "WuzAPI-Fly"
  TZ = "Australia/Sydney"
  WUZAPI_PORT = "8080"

[http_service]
  internal_port = 8080  # wuzapi listens on port 8080
  force_https = true
  auto_stop_machines = 'off'  # Keep running - WhatsApp needs persistent connections
  auto_start_machines = true
  min_machines_running = 1    # Always keep at least 1 machine running
  processes = ['app']

[[vm]]
  cpu_kind = 'shared'
  cpus = 1
  memory_mb = 512  # 512MB is sufficient for wuzapi

# Health check configuration
[[services.http_checks]]
  interval = "30s"
  timeout = "5s"
  grace_period = "10s"
  method = "GET"
  path = "/health"

# Custom domain
[[http_service.domains]]
  name = "wuzapi.gasbridge.nz"
```

**Key Configuration Notes:**

- `auto_stop_machines = 'off'`: Prevents machines from stopping, which would disconnect WhatsApp sessions
- `min_machines_running = 1`: Ensures the service is always available
- `internal_port = 8080`: Matches WuzAPI's default port
- `memory_mb = 512`: Sufficient for most use cases; increase for heavy loads

## Database Configuration

WuzAPI requires a PostgreSQL database for production deployments.

### Create Fly Postgres

```bash
# Create a PostgreSQL cluster
flyctl postgres create --name wuzapi-db --region syd --initial-cluster-size 1

# Save the connection details shown (username, password, hostname)
# Example output:
#   Username:    postgres
#   Password:    <generated-password>
#   Hostname:    wuzapi-db.internal
#   Flycast:     fdaa:3b:36ff:0:1::2
#   Proxy port:  5432
#   Postgres port: 5433

# Attach to your app (sets DATABASE_URL automatically)
flyctl postgres attach wuzapi-db --app wuzapi-shy-river-5375
```

### Create the WuzAPI Database

```bash
# Connect to Postgres
flyctl postgres connect --app wuzapi-db

# Create database
CREATE DATABASE wuzapi;

# Verify
\l
# Should show:
#   wuzapi                | postgres | UTF8     | ...
#   wuzapi_shy_river_5375 | postgres | UTF8     | ... (auto-created)

# Exit
\q
```

**Note:** Fly creates a database named `wuzapi_shy_river_5375` automatically when you attach. You can use either:
- `wuzapi` (as we configured)
- `wuzapi_shy_river_5375` (auto-created)

Just ensure `DB_NAME` secret matches your choice.

## Secrets Management

WuzAPI requires several secrets for secure operation. Set these using `flyctl secrets set`.

### Required Secrets

```bash
flyctl secrets set \
  WUZAPI_ADMIN_TOKEN="$(openssl rand -hex 32)" \
  DB_USER="postgres" \
  DB_PASSWORD="your-postgres-password-from-create" \
  DB_NAME="wuzapi" \
  DB_HOST="wuzapi-db.internal" \
  DB_PORT="5432" \
  DB_SSLMODE="disable" \
  --app wuzapi-shy-river-5375
```

### Optional but Recommended Secrets

```bash
flyctl secrets set \
  WUZAPI_GLOBAL_ENCRYPTION_KEY="$(openssl rand -hex 32)" \
  WUZAPI_GLOBAL_HMAC_KEY="$(openssl rand -hex 32)" \
  WUZAPI_GLOBAL_WEBHOOK="https://your-webhook-endpoint.com" \
  --app wuzapi-shy-river-5375
```

**Important Notes:**

- **Auto-Generation**: If not provided, `WUZAPI_GLOBAL_ENCRYPTION_KEY` and `WUZAPI_GLOBAL_HMAC_KEY` are auto-generated on first run
- **Retrieve from Logs**: After first deployment, check logs to save auto-generated keys:
  ```bash
  flyctl logs --app wuzapi-shy-river-5375 | grep -E "ENCRYPTION_KEY|HMAC_KEY"
  ```
- **Persist Keys**: Set them as secrets to ensure they persist across deployments:
  ```bash
  flyctl secrets set \
    WUZAPI_GLOBAL_ENCRYPTION_KEY="key-from-logs" \
    WUZAPI_GLOBAL_HMAC_KEY="key-from-logs" \
    --app wuzapi-shy-river-5375
  ```

### View Configured Secrets

```bash
# List secrets (shows names only, not values)
flyctl secrets list --app wuzapi-shy-river-5375

# Expected output:
# NAME                    DIGEST
# DATABASE_URL            67ca66a5edffb628
# DB_HOST                 600c5cae738ccdf0
# DB_NAME                 6cb57494e1c6c980
# DB_PASSWORD             a2da060c18463dfd
# DB_PORT                 5bed575942cb6315
# DB_SSLMODE              fa61a13817d73a23
# DB_USER                 8a66529be806f1bd
# WUZAPI_ADMIN_TOKEN      8ae16d8dc7f8fd10
# WUZAPI_GLOBAL_WEBHOOK   8a581e6aec9471d2
```

### Migrating from Existing Deployment

If migrating from Docker/Raspberry Pi with existing sessions:

```bash
# On your current server
cat .env | grep -E "ENCRYPTION_KEY|HMAC_KEY|ADMIN_TOKEN"

# Set these exact values in Fly to preserve sessions
flyctl secrets set \
  WUZAPI_GLOBAL_ENCRYPTION_KEY="key-from-existing-env" \
  WUZAPI_GLOBAL_HMAC_KEY="key-from-existing-env" \
  WUZAPI_ADMIN_TOKEN="token-from-existing-env" \
  --app wuzapi-shy-river-5375
```

## Initial Deployment

```bash
# Deploy the application
flyctl deploy --app wuzapi-shy-river-5375

# Watch deployment progress
flyctl logs --app wuzapi-shy-river-5375

# Check status
flyctl status --app wuzapi-shy-river-5375

# Open in browser
flyctl open --app wuzapi-shy-river-5375
```

The application will be available at:
- Main URL: `https://wuzapi-shy-river-5375.fly.dev`
- Dashboard: `https://wuzapi-shy-river-5375.fly.dev/dashboard`
- API Docs: `https://wuzapi-shy-river-5375.fly.dev/api`

## Custom Domain Setup

### 1. Add SSL Certificate to Fly

```bash
# Add your custom domain
flyctl certs add wuzapi.gasbridge.nz --app wuzapi-shy-river-5375

# Check certificate status
flyctl certs show wuzapi.gasbridge.nz --app wuzapi-shy-river-5375
```

### 2. Configure DNS in Cloudflare

**Critical:** Disable proxy mode (set to DNS only / gray cloud) to allow Fly.io to manage SSL.

**DNS Configuration for `gasbridge.nz` in Cloudflare:**

| Type | Name | Target | TTL | Proxy Status |
|------|------|--------|-----|--------------|
| CNAME | wuzapi | wuzapi-shy-river-5375.fly.dev | Auto | DNS only ☁️ (gray cloud) |

**Steps in Cloudflare Dashboard:**

1. Log in to Cloudflare
2. Select domain: `gasbridge.nz`
3. Go to DNS → Records
4. Click "Add record"
5. Type: `CNAME`
6. Name: `wuzapi`
7. Target: `wuzapi-shy-river-5375.fly.dev`
8. **Click the orange cloud to turn it gray** (DNS only)
9. Save

### 3. Verify Certificate Issuance

```bash
# Wait for certificate to be issued (usually 1-5 minutes)
flyctl certs check wuzapi.gasbridge.nz --app wuzapi-shy-river-5375

# You should see: "The certificate for wuzapi.gasbridge.nz has been issued"
```

### 4. Update fly.toml

The custom domain is already configured in `fly.toml`:

```toml
[[http_service.domains]]
  name = "wuzapi.gasbridge.nz"
```

If you added it manually, redeploy:

```bash
flyctl deploy --app wuzapi-shy-river-5375
```

### 5. Test Access

```bash
# Test health endpoint
curl -I https://wuzapi.gasbridge.nz/health

# Should return: HTTP/2 200

# Test dashboard
curl -I https://wuzapi.gasbridge.nz/dashboard

# Should return: HTTP/2 200
```

## GitHub Actions CI/CD

Automate deployments using GitHub Actions.

### 1. Get Fly API Token

```bash
# Create a deploy token
flyctl tokens create deploy -x 999999h

# Copy the token output
```

### 2. Add GitHub Secret

1. Go to your GitHub repository
2. Settings → Secrets and variables → Actions
3. Click "New repository secret"
4. Name: `FLY_API_TOKEN`
5. Value: Paste the token from step 1

### 3. GitHub Actions Workflow

The repository includes `.github/workflows/fly-deploy.yml`:

```yaml
# See https://fly.io/docs/app-guides/continuous-deployment-with-github-actions/
#
# IMPORTANT: Runtime secrets must be set using flyctl, not in this file
# Run this command once to set all secrets:
#
#   flyctl secrets set \
#     WUZAPI_ADMIN_TOKEN="your-token" \
#     WUZAPI_GLOBAL_ENCRYPTION_KEY="your-encryption-key" \
#     WUZAPI_GLOBAL_HMAC_KEY="your-hmac-key" \
#     DB_USER="your-db-user" \
#     DB_PASSWORD="your-db-password" \
#     DB_NAME="your-db-name" \
#     DB_HOST="your-db-host.internal" \
#     DB_PORT="5432" \
#     DB_SSLMODE="disable" \
#     WUZAPI_GLOBAL_WEBHOOK="your-webhook-url"
#
# Non-sensitive config is in fly.toml [env] section

name: Fly Deploy
on:
  push:
    branches:
      - main
jobs:
  deploy:
    name: Deploy app
    runs-on: ubuntu-latest
    concurrency: deploy-group    # Ensure only one deployment runs at a time
    steps:
      - uses: actions/checkout@v4
      - uses: superfly/flyctl-actions/setup-flyctl@master
      - run: flyctl deploy --remote-only
        env:
          FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}
```

**Important:** All secrets (database credentials, encryption keys) should be set via `flyctl secrets set`, not in GitHub Actions environment variables.

### 4. Deploy via Git Push

```bash
git add .
git commit -m "Configure Fly.io deployment"
git push origin main

# Deployment will trigger automatically
# View progress at: https://github.com/your-repo/actions
```

## Troubleshooting

### Common Issues and Solutions

#### 1. Database Authentication Failed

**Error:** `FATAL Failed to initialize database error="failed to ping postgres database: pq: password authentication failed for user \"postgres\""`

**Solution:**
```bash
# Reset Postgres password
flyctl postgres connect --app wuzapi-db
ALTER USER postgres WITH PASSWORD 'NewSecurePassword123!';
\q

# Update secret
flyctl secrets set DB_PASSWORD="NewSecurePassword123!" --app wuzapi-shy-river-5375
```

#### 2. Database Does Not Exist

**Error:** `FATAL Failed to initialize database error="failed to ping postgres database: pq: database \"wuzapi\" does not exist"`

**Solution:**
```bash
# Connect and create database
flyctl postgres connect --app wuzapi-db
CREATE DATABASE wuzapi;
\q

# Or update DB_NAME to match existing database
flyctl secrets set DB_NAME="wuzapi_shy_river_5375" --app wuzapi-shy-river-5375
```

#### 3. App Not Listening on Port 8080

**Error:** `App is not listening to the expected port`

**Solution:**
- Verify `fly.toml` has `internal_port = 8080`
- Check `WUZAPI_PORT` is set to `"8080"` in `[env]` section
- Redeploy: `flyctl deploy --app wuzapi-shy-river-5375`

#### 4. Machines Restarting Frequently

**Symptoms:** WhatsApp sessions disconnecting, 502 errors, logs show frequent restarts

**Fly Doctor Output:**
```
Symptom: Machines Restarting a Lot
Your machines are restarting frequently, which indicates a stability problem
with your application.
```

**Solution:**
- Set `auto_stop_machines = 'off'` in `fly.toml`
- Set `min_machines_running = 1` in `fly.toml`
- Increase memory if logs show OOM errors: `memory_mb = 1024`
- Check database connection (most common cause)

#### 5. Lost Encryption Keys

**Problem:** Auto-generated keys not saved, can't decrypt data

**Solution:**
```bash
# Check recent logs for keys (within first few deployments)
flyctl logs --app wuzapi-shy-river-5375 | grep -E "ENCRYPTION_KEY|HMAC_KEY"

# If found, immediately set as secrets
flyctl secrets set \
  WUZAPI_GLOBAL_ENCRYPTION_KEY="key-from-logs" \
  WUZAPI_GLOBAL_HMAC_KEY="key-from-logs" \
  --app wuzapi-shy-river-5375
```

**Prevention:** Always check logs after first deployment and save auto-generated keys.

### Useful Debugging Commands

```bash
# View real-time logs
flyctl logs --app wuzapi-shy-river-5375

# SSH into machine
flyctl ssh console --app wuzapi-shy-river-5375

# Check machine status
flyctl status --app wuzapi-shy-river-5375

# List all machines
flyctl machine list --app wuzapi-shy-river-5375

# Restart machine
flyctl machine restart MACHINE_ID --app wuzapi-shy-river-5375

# Check secrets
flyctl secrets list --app wuzapi-shy-river-5375

# Check database connection
flyctl postgres connect --app wuzapi-db
\l
\dt
\q

# Check DNS propagation
dig wuzapi.gasbridge.nz
nslookup wuzapi.gasbridge.nz

# Check certificate status
flyctl certs check wuzapi.gasbridge.nz --app wuzapi-shy-river-5375
```

## Maintenance

### Updating WuzAPI

```bash
# Pull latest changes
git pull origin main

# Deploy update
flyctl deploy --app wuzapi-shy-river-5375

# Or push to main branch to trigger GitHub Actions
git push origin main
```

### Scaling Resources

#### Increase Memory

Edit `fly.toml`:
```toml
[[vm]]
  memory_mb = 1024  # Increased from 512
```

Then redeploy:
```bash
flyctl deploy --app wuzapi-shy-river-5375
```

#### Scale Horizontally (Multiple Regions)

```bash
# Add machine in another region (e.g., Los Angeles)
flyctl machine clone --region lax --app wuzapi-shy-river-5375

# Note: Requires careful database replication setup
```

### Database Maintenance

```bash
# List databases
flyctl postgres db list --app wuzapi-db

# Backup database
flyctl postgres connect --app wuzapi-db
pg_dump wuzapi > backup-$(date +%Y%m%d).sql
\q

# Restore database
flyctl postgres connect --app wuzapi-db < backup-20251227.sql

# Check database size
flyctl postgres db list --app wuzapi-db
```

### Monitor Application

```bash
# Check app metrics
flyctl dashboard metrics --app wuzapi-shy-river-5375

# Monitor in Fly.io dashboard
# Visit: https://fly.io/apps/wuzapi-shy-river-5375

# Check health endpoint
curl https://wuzapi.gasbridge.nz/health
```

### Cost Optimization

**Current Configuration Costs (Approximate):**
- Shared CPU-1x (512MB RAM): ~$2-3/month
- PostgreSQL (256MB): ~$2-3/month
- Bandwidth (1GB free, then ~$0.02/GB): ~$0-2/month
- **Total: ~$5-10/month**

**Tips:**
- Monitor actual usage in Fly.io dashboard
- Current 512MB memory is optimal for single-user deployments
- PostgreSQL auto-scales storage as needed
- Bandwidth is typically minimal for API usage

## Deployed Configuration Summary

| Setting | Value |
|---------|-------|
| **App Name** | `wuzapi-shy-river-5375` |
| **Region** | `syd` (Sydney, Australia) |
| **Database** | `wuzapi-db` (PostgreSQL) |
| **Database Name** | `wuzapi` |
| **Custom Domain** | `wuzapi.gasbridge.nz` |
| **Memory** | 512MB |
| **CPU** | 1 shared CPU |
| **Auto-stop** | Disabled (always running) |
| **Min Machines** | 1 |
| **Timezone** | Australia/Sydney |
| **Port** | 8080 |

## Comparison with Other Platforms

| Platform | WuzAPI Suitability | Monthly Cost | Notes |
|----------|-------------------|--------------|-------|
| **Fly.io** | ⭐⭐⭐⭐⭐ Excellent | $5-10 | ✅ Deployed here: native WebSocket, stateful apps, regional deployment |
| **Railway** | ⭐⭐⭐⭐ Very Good | $8-15 | Similar to Fly.io, easier UI, slightly more expensive |
| **Render** | ⭐⭐⭐⭐ Very Good | $10-20 | Native Postgres, simple setup |
| **Cloudflare Containers** | ⭐⭐ Poor Fit | Unknown | Beta status, edge-focused (not regional), architectural mismatch |
| **Raspberry Pi 4B** | ⭐⭐⭐⭐ Very Good | $0 | Previous deployment: great for home use, requires stable internet |
| **Hetzner Cloud** | ⭐⭐⭐⭐⭐ Excellent | €4.50 | Best value VPS option, requires manual setup |

## Getting Help

- **Fly.io Community:** https://community.fly.io
- **WuzAPI Issues:** https://github.com/asternic/wuzapi/issues
- **Fly.io Documentation:** https://fly.io/docs
- **This Deployment:** Deployed from https://github.com/dipdapdop/wuzapi

## Security Considerations

1. **Always use HTTPS** - Fly.io provides this automatically ✅
2. **Set strong admin token** - Use `openssl rand -hex 32` ✅
3. **Enable HMAC signing** for webhooks
4. **Keep encryption keys secure** - Never commit to Git ✅
5. **Use Fly secrets** - Never put credentials in `fly.toml` or GitHub Actions ✅
6. **Regular updates** - Keep WuzAPI and dependencies updated
7. **Monitor logs** - Watch for unauthorized access attempts

## Next Steps

After successful deployment:

1. ✅ Access the dashboard: `https://wuzapi.gasbridge.nz/dashboard`
2. Create your first user via `/admin/users` endpoint
3. Scan QR code to connect WhatsApp
4. Configure webhooks for events
5. Review API documentation at `https://wuzapi.gasbridge.nz/api`
6. Set up monitoring and alerting
7. Configure backups for PostgreSQL

---

**Deployment Date:** 2025-12-27
**WuzAPI Version:** 1.0.5
**Fly.io Platform:** Machines v2
**Deployed By:** dipdapdop

For issues or improvements to this guide, please open an issue or PR.