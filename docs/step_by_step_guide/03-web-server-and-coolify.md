# Step-by-Step Application Platform Setup (Coolify)

**Difficulty**: Intermediate
**Prerequisites**:

- Completed guide 01 (Security Hardening)
- Completed guide 02 (VPN Setup)
- Domain name pointing to your server (for public apps with HTTPS)
- Fedora 43 Server with SSH working

---

## What You'll Learn

This guide teaches you how to set up Coolify, a self-hosted Platform as a Service (PaaS) for deploying and managing all your applications:

1. **Install Coolify** with its built-in reverse proxy (Traefik)
2. **Deploy public applications** (accessible from internet with HTTPS)
3. **Deploy VPN-only applications** (only accessible via Tailscale)
4. **Manage databases** securely
5. **Understand traffic routing** and security

### What is Coolify?

**Coolify** is a self-hosted alternative to Heroku, Netlify, Vercel, and Railway:
- Web UI for deploying applications
- Built-in reverse proxy (Traefik) with automatic HTTPS
- Deploy from Git (GitHub, GitLab, Bitbucket)
- One-click services (WordPress, Ghost, databases, monitoring)
- Docker-based
- Free and open source

**What you can deploy**:
- Static websites (HTML, React, Vue, Next.js)
- Backend apps (Node.js, Python, Go, PHP, Ruby)
- Databases (PostgreSQL, MySQL, MongoDB, Redis)
- Pre-configured services (WordPress, Plausible, Uptime Kuma, etc.)

---

## ⚠️ Safety Rules

1. **Keep SSH access working** (test after firewall changes)
2. **Access Coolify UI via VPN only** (never expose publicly)
3. **Understand public vs VPN-only** before deploying services
4. **Test deployments** before going to production

---

## Current Working State

Verify your starting point:

```bash
# 1. Firewall is running
sudo firewall-cmd --state
# Output: running

# 2. HTTP/HTTPS ports are allowed (from guide 01)
sudo firewall-cmd --zone=public --list-services | grep -E "http|https"
# Should show: http https

# 3. VPN is working
tailscale status
# Should show your devices

# 4. Get your Tailscale IP (you'll need this)
TAILSCALE_IP=$(tailscale ip -4)
echo "Your Tailscale IP: $TAILSCALE_IP"
# Save this IP!
```

---

## Architecture Overview

### How Everything Works Together

```
┌─────────────────────────────────────────────────────────────┐
│                        INTERNET                              │
│                    (Public Access)                           │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ Port 80/443 (HTTP/HTTPS)
                     │
                     ▼
           ┌─────────────────────┐
           │   Your Server       │
           │   Firewall          │
           └─────────┬───────────┘
                     │
                     ▼
         ┌───────────────────────────┐
         │  Coolify's Traefik        │
         │  (Reverse Proxy)          │
         └───────┬───────────────────┘
                 │
                 ├──► Public Apps (with domains)
                 │    example.com → App on port 3000
                 │    api.example.com → App on port 4000
                 │
                 └──► (VPN-only apps NOT proxied here)


┌─────────────────────────────────────────────────────────────┐
│                    TAILSCALE VPN                             │
│                (Private Access - 100.x.x.x)                  │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
           ┌─────────────────────┐
           │   Your Server       │
           │   Trusted Zone      │
           └─────────┬───────────┘
                     │
                     ├──► Coolify UI (port 8000)
                     │    http://100.x.x.x:8000
                     │
                     ├──► VPN-only Apps (no domain)
                     │    http://100.x.x.x:3001
                     │
                     └──► Databases
                          postgresql://100.x.x.x:5432
```

### Key Principles

1. **Coolify UI**: Always VPN-only (100.x.x.x:8000)
2. **Public apps**: Get domains (example.com) → Traefik handles HTTPS
3. **VPN-only apps**: No domains → Access via 100.x.x.x:port
4. **Databases**: VPN-only by default
5. **No manual reverse proxy needed**: Coolify's Traefik handles everything

---

## Step 1: Install Docker

Coolify requires Docker for container management.

```bash
# Remove old Docker if installed
sudo dnf remove -y docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine

# Add Docker repository
sudo dnf config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo

# Install Docker
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Start and enable Docker
sudo systemctl enable --now docker

# Verify Docker is running
sudo systemctl status docker | grep Active
# Should show: Active: active (running)

# Test Docker
sudo docker run hello-world
# Should download and run test container
```

✅ **Expected**: "Hello from Docker!" message

### Add User to Docker Group

```bash
# Add fedora user to docker group
sudo usermod -aG docker fedora

# Apply group changes (or logout/login)
newgrp docker

# Test Docker without sudo
docker ps
# Should work without permission error
```

---

## Step 2: Install Coolify

```bash
# Download and run Coolify installer
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
```

This script will:
1. Install required dependencies
2. Set up Coolify Docker containers
3. Configure Traefik (reverse proxy)
4. Start Coolify services

**Wait 2-5 minutes** for installation to complete.

### Verify Installation

```bash
# Check Coolify containers are running
docker ps | grep coolify

# Should see several containers:
# - coolify
# - coolify-proxy (Traefik)
# - coolify-db (PostgreSQL)
# - coolify-redis

# Check Coolify is accessible locally
curl http://localhost:8000
# Should return HTML (Coolify UI)
```

✅ **Success**: Coolify is installed!

---

## Step 3: Secure Coolify UI (VPN-Only Access)

**CRITICAL SECURITY STEP**: Keep Coolify UI accessible only via VPN.

### Why VPN-Only?

- **Coolify UI** = Full control of your server
- If exposed publicly = Anyone can try to access
- Via VPN only = Only you can access (maximum security)
- Your public apps can still be deployed and accessible

### Verify Port 8000 is NOT Public

```bash
# Check public zone ports
sudo firewall-cmd --zone=public --list-ports

# Should NOT show 8000
# If it shows 8000, remove it:
sudo firewall-cmd --permanent --remove-port=8000/tcp
sudo firewall-cmd --reload
```

### Access Coolify via VPN

```bash
# Get your Tailscale IP (if you forgot)
tailscale ip -4

# Access Coolify from your local machine:
# http://100.x.x.x:8000
# (Replace 100.x.x.x with your server's Tailscale IP)
```

**From your local machine** (with Tailscale connected):
1. Open browser
2. Visit: `http://100.x.x.x:8000` (your server's Tailscale IP)
3. Should load Coolify setup screen

✅ **Critical Test**: Can you access Coolify via VPN? If not, troubleshoot before continuing.

---

## Step 4: Initial Coolify Setup

### First-Time Configuration

1. **Access Coolify**: `http://100.x.x.x:8000` (via VPN)

2. **Create Admin Account**:
   - Email: your email
   - Password: **strong password** (use password manager)
   - Name: your name

3. **Configure Server**:
   - Server name: `production` (or your choice)
   - Should auto-detect Docker
   - Click "Validate & Continue"

4. **Complete Setup**:
   - Review settings
   - Click "Finish Setup"

### Verify Dashboard

You should see:
- Coolify dashboard
- Server status (green/online)
- Resources: 0 projects, 0 applications
- System metrics (CPU, memory, disk)

✅ **Success**: Coolify UI accessible and configured!

---

## Step 5: Understanding Public vs VPN-Only Deployments

Before deploying anything, understand the two approaches:

### Public Application

**When to use**: Website, API, or service that should be accessible from the internet

**Configuration**:
- Set a domain: `app.example.com`
- Coolify's Traefik will:
  - Listen on ports 80/443 (allowed by firewall)
  - Get HTTPS certificate from Let's Encrypt
  - Route `app.example.com` → your app

**Access**: Anyone on internet can visit `https://app.example.com`

### VPN-Only Application

**When to use**: Admin panels, internal tools, development apps

**Configuration**:
- **Do NOT set a domain**
- Coolify will assign a port (e.g., 3001)
- Access via Tailscale IP

**Access**: Only you via VPN: `http://100.x.x.x:3001`

### Comparison

| Aspect | Public | VPN-Only |
|--------|--------|----------|
| Domain | Yes (app.example.com) | No |
| HTTPS | Yes (automatic) | No (HTTP only) |
| Firewall | Port 80/443 (allowed) | Blocked from internet |
| Access | Anyone | VPN only |
| Use Cases | Websites, APIs, public services | Admin panels, databases, internal tools |

---

## Step 6: Deploy Your First Public Application

Let's deploy a public website that anyone can access.

### Create New Project

1. **In Coolify dashboard**: Click "New Project"
2. **Project name**: "My Website"
3. **Click**: "Create Project"

### Add Public Application

1. **Click**: "Add Resource" → "Application"
2. **Source**: Choose "Public Repository"
3. **Repository URL**: Use a test repo or your own
   - Example: `https://github.com/vercel/next.js/tree/canary/examples/hello-world`
   - Or any static HTML site repo

4. **Configuration**:
   - **Name**: `hello-world`
   - **Branch**: `main` or `master`
   - **Build Pack**: Auto-detect (or select manually)
   - **Port**: 3000 (or your app's port)

5. **Domain Configuration** (CRITICAL for public access):
   - **Click**: "Domains" tab
   - **Add domain**: `hello.example.com` (use your actual domain)
   - **Enable HTTPS**: Yes (default)
   - **Save**

6. **Deploy**:
   - Click "Deploy"
   - Watch build logs
   - Wait for "Application is running" (2-5 minutes)

### Configure DNS

**Before accessing**, configure DNS:

```bash
# From your local machine, add DNS A record:
# Hostname: hello.example.com
# Type: A
# Value: YOUR_SERVER_PUBLIC_IP
# TTL: 300 (5 minutes)

# Verify DNS after a few minutes:
dig +short hello.example.com
# Should show your server's public IP
```

### Test Public Access

**From any device** (not VPN required):
1. Visit: `https://hello.example.com`
2. Should load your application
3. Should have valid HTTPS certificate (green padlock)

✅ **Success**: Public application deployed and accessible!

### What Happened Behind the Scenes

1. Coolify built your app in a Docker container
2. App started on port 3000 (inside container)
3. Traefik detected the domain configuration
4. Traefik requested HTTPS cert from Let's Encrypt
5. Traefik routes `hello.example.com` → your container
6. Public traffic flows: Internet → Firewall (80/443) → Traefik → Your App

---

## Step 7: Deploy a VPN-Only Application

Now let's deploy an admin panel that should NOT be public.

### Add VPN-Only Application

1. **In same project**: Click "Add Resource" → "Application"
2. **Source**: Public Repository (or your private repo)
3. **Repository**: Your admin panel code
4. **Configuration**:
   - **Name**: `admin-panel`
   - **Branch**: `main`
   - **Port**: 3000 (or your app's port)

5. **Domain Configuration** (KEY DIFFERENCE):
   - **Do NOT add any domain**
   - Leave domains section empty
   - Save

6. **Deploy**:
   - Click "Deploy"
   - Wait for deployment
   - Note the assigned port (e.g., 3001)

### Find Access Information

After deployment:
1. Go to application details
2. Find "Network" or "Ports" section
3. Note the mapped port (e.g., 3001)

### Access VPN-Only Application

**From your local machine** (with Tailscale):
```bash
# Access via Tailscale IP and port
# http://100.x.x.x:3001

# Open in browser:
# http://100.x.x.x:3001
```

✅ **Success**: Admin panel accessible via VPN, not from public internet!

### Test It's Actually Private

**From your local machine** (disable Tailscale temporarily):
```bash
# Try accessing via public IP:
curl http://YOUR_PUBLIC_IP:3001
# Should timeout or refuse connection

# Try accessing via domain (if you accidentally added one):
curl https://admin.example.com
# Should not resolve or fail
```

✅ **Verification**: Not accessible from public internet!

---

## Step 8: Deploy a Database

Databases should ALWAYS be VPN-only for security.

### Add Database

1. **In your project**: Click "Add Resource" → "Database"
2. **Database Type**: PostgreSQL (or choose MySQL, MongoDB, Redis, etc.)
3. **Configuration**:
   - **Name**: `my-postgres`
   - **Version**: Latest stable
   - **Database Name**: `myapp`
   - **Username**: Auto-generated (or set your own)
   - **Password**: Strong password (Coolify generates one)

4. **Deploy**:
   - Click "Create"
   - Wait for container to start (30-60 seconds)

### Get Connection Details

1. Go to database details
2. Find "Connection" section
3. Copy connection details:

```
Internal URL: postgresql://user:pass@postgres:5432/myapp
External URL: postgresql://100.x.x.x:5432/myapp
```

### Important: Database Security

**By default**:
- Database is NOT exposed on port 5432 to public internet
- Only accessible from:
  - Other containers on same Docker network (internal URL)
  - Via Tailscale VPN (external URL with VPN IP)

**Verify it's private**:
```bash
# Check public ports
sudo firewall-cmd --zone=public --list-ports
# Should NOT show 5432

# Check what's listening on 5432
sudo ss -tlnp | grep 5432
# Shows it's listening, but firewall blocks external access
```

✅ **Success**: Database deployed and secured!

### Connect Application to Database

**For public apps**:
1. Go to application settings
2. Click "Environment Variables"
3. Add variable:
   ```
   DATABASE_URL=postgresql://user:pass@postgres:5432/myapp
   ```
   Use the **internal URL** (postgres hostname works within Docker network)
4. Redeploy application

**For VPN-only apps**:
- Can use either internal or external URL
- External URL useful for accessing from your dev machine via VPN

---

## Step 9: One-Click Services

Coolify provides pre-configured services you can deploy instantly.

### Available Services

Common one-click services:
- **WordPress**: CMS/blogging platform
- **Ghost**: Modern blogging platform
- **Plausible**: Privacy-friendly analytics
- **Umami**: Alternative analytics
- **Uptime Kuma**: Uptime monitoring
- **n8n**: Workflow automation
- **Vikunja**: Project management
- **And many more...**

### Example: Deploy Uptime Kuma (VPN-Only)

**Use case**: Monitor your services, but dashboard should be private

1. **Add Resource**: Click "Add Resource" → "Service"
2. **Search**: Type "uptime kuma"
3. **Select**: Uptime Kuma
4. **Configuration**:
   - **Name**: `monitoring`
   - **Do NOT add domain** (keep VPN-only)
5. **Deploy**: Click "Deploy"
6. **Access**: `http://100.x.x.x:3002` (note the assigned port)

### Example: Deploy WordPress (Public)

**Use case**: Public blog that anyone can read

1. **Add Resource**: Click "Add Resource" → "Service"
2. **Select**: WordPress
3. **Configuration**:
   - **Name**: `blog`
   - **Domain**: `blog.example.com`
   - **Database**: Coolify creates one automatically
4. **Deploy**: Click "Deploy"
5. **Configure DNS**: Point `blog.example.com` to your server
6. **Access**: `https://blog.example.com`

---

## Step 10: Advanced - VPN-Only Services with Domains and HTTPS

**Problem**: Using `http://100.x.x.x:3001` for VPN-only services is not user-friendly and lacks HTTPS.

**Solution**: Use domains with IP-based access restrictions - you get nice URLs with HTTPS, but only accessible via VPN!

### Approach: Public DNS + IP Restrictions

1. **Create DNS record** pointing to your server (public)
2. **Deploy app with domain** in Coolify
3. **Add IP whitelist** (allow only Tailscale IPs)
4. **Result**: `https://admin.example.com` works with HTTPS, but blocked for non-VPN users

### Example: VPN-Only Admin Panel with Domain

#### Step 1: Deploy App with Domain

1. In Coolify, deploy your admin app
2. Add domain: `admin.example.com`
3. Deploy normally

#### Step 2: Create DNS Record

```bash
# Add DNS A record:
# Hostname: admin.example.com
# Type: A
# Value: YOUR_SERVER_PUBLIC_IP

# This makes the domain resolve publicly (needed for Let's Encrypt)
```

#### Step 3: Add IP Restriction in Coolify

**Method A: Via Coolify UI** (if supported in your version):
1. Go to application → Settings
2. Find "Middlewares" or "Security" section
3. Add IP whitelist: `100.64.0.0/10` (Tailscale IP range)

**Method B: Via Docker Labels** (more universal):

Add custom Docker labels to restrict access:

1. In Coolify, go to application settings
2. Find "Custom Docker Labels" or "Advanced" section
3. Add these labels:

```yaml
traefik.http.middlewares.vpn-whitelist.ipwhitelist.sourcerange=100.64.0.0/10
traefik.http.routers.YOUR_APP_NAME.middlewares=vpn-whitelist
```

Replace `YOUR_APP_NAME` with your actual app name (check in Coolify or `docker ps`).

4. Redeploy application

#### Step 4: Test Access

**Via VPN (should work)**:
```bash
# From local machine with Tailscale connected:
curl https://admin.example.com
# ✅ Should load
```

**Without VPN (should fail)**:
```bash
# Disable Tailscale, then:
curl https://admin.example.com --connect-timeout 5
# ❌ Should get 403 Forbidden or timeout
```

### Understanding What Happened

```
Public User (not on VPN):
  https://admin.example.com
         ↓
  DNS resolves to server
         ↓
  Reaches Traefik (port 443)
         ↓
  Traefik checks IP: Not in 100.64.0.0/10
         ↓
  ❌ 403 Forbidden

VPN User (on Tailscale):
  https://admin.example.com
         ↓
  DNS resolves to server
         ↓
  Reaches Traefik (port 443)
         ↓
  Traefik checks IP: Is in 100.64.0.0/10 ✓
         ↓
  Routes to your app
         ↓
  ✅ App loads
```

**Key point**: The domain resolves publicly (needed for HTTPS cert), but Traefik blocks non-VPN IPs at the application level.

### Alternative: Use Tailscale MagicDNS

If you don't want public DNS records at all:

1. **Enable MagicDNS** in Tailscale (from guide 02)
2. **Use Tailscale hostname**: `http://fedora-server:3001`
3. **For HTTPS**: Need internal certificate authority (more complex)

**Pros**: No public DNS at all
**Cons**: No automatic HTTPS, more complex setup

### Recommended Approach

**For most VPN-only services**: Use IP restrictions (Method above)
- ✅ Automatic HTTPS
- ✅ Nice domain names
- ✅ Easy to manage in Coolify
- ⚠️ DNS record is public (but service is blocked)

**For truly sensitive services**: Use direct IP:port access
- ✅ No public DNS footprint
- ✅ Simpler configuration
- ❌ No HTTPS (unless you set up internal CA)
- ❌ Less user-friendly URLs

### Complete Example: Admin Dashboard

```bash
# 1. Deploy in Coolify
# Name: admin-dashboard
# Domain: admin.example.com
# Add custom labels:
traefik.http.middlewares.vpn-only.ipwhitelist.sourcerange=100.64.0.0/10
traefik.http.routers.admin-dashboard.middlewares=vpn-only

# 2. Create DNS
# admin.example.com → YOUR_SERVER_PUBLIC_IP

# 3. Access
# https://admin.example.com (works only via VPN)
```

### Multiple VPN-Only Apps

For multiple apps, you can reuse the middleware:

```yaml
# App 1: admin.example.com
traefik.http.routers.admin.middlewares=vpn-only

# App 2: monitoring.example.com
traefik.http.routers.monitoring.middlewares=vpn-only

# App 3: internal.example.com
traefik.http.routers.internal.middlewares=vpn-only

# Shared middleware (define once)
traefik.http.middlewares.vpn-only.ipwhitelist.sourcerange=100.64.0.0/10
```

All use the same `vpn-only` middleware with Tailscale IP whitelist.

### Troubleshooting IP Restrictions

**Test if restriction is working**:
```bash
# Check Traefik logs
docker logs coolify-proxy | grep -i ipwhitelist

# Test from VPN
curl -v https://admin.example.com
# Should see 200 OK

# Test without VPN (should fail)
curl -v https://admin.example.com
# Should see 403 Forbidden
```

**If restriction not working**:
```bash
# Check middleware is applied
docker inspect YOUR_CONTAINER | grep -A 5 middleware

# Check Traefik configuration
docker exec coolify-proxy cat /etc/traefik/traefik.yml

# Verify Tailscale IP range
# All Tailscale IPs are in: 100.64.0.0/10
```

### Summary: Three Deployment Options

| Option | URL | HTTPS | Access | Use Case |
|--------|-----|-------|--------|----------|
| **Public** | https://app.example.com | ✅ | Anyone | Public websites, APIs |
| **VPN-Only with Domain** | https://admin.example.com | ✅ | VPN only (IP restricted) | Admin panels, internal tools (user-friendly) |
| **VPN-Only IP:Port** | http://100.x.x.x:3001 | ❌ | VPN only (firewall) | Simple internal tools, databases |

Choose based on your needs!

---

## Step 11: Understanding Traefik Routing

Coolify uses Traefik as its reverse proxy. Let's understand how it works.

### How Traefik Routes Traffic

```
Public Request: https://app.example.com
          ↓
    [Firewall allows 80/443]
          ↓
      [Traefik]
          ↓
    Check domain: app.example.com
          ↓
    Find matching container (has label: traefik.http.routers.app.rule=Host(`app.example.com`))
          ↓
    Route to container port 3000
          ↓
      [Your App]
```

### Traefik Labels

Coolify automatically adds labels to containers:
```yaml
traefik.enable=true
traefik.http.routers.myapp.rule=Host(`app.example.com`)
traefik.http.routers.myapp.tls=true
traefik.http.routers.myapp.tls.certresolver=letsencrypt
```

These labels tell Traefik:
- Enable routing for this container
- Route requests for `app.example.com` to this container
- Enable HTTPS
- Use Let's Encrypt for certificate

### View Traefik Dashboard (Optional)

Traefik has a dashboard (if enabled):
```bash
# Access Traefik dashboard via VPN
# http://100.x.x.x:8080

# Shows all routes, backends, certificates
```

---

## Step 11: Environment Variables and Secrets

Securely manage configuration and secrets.

### Adding Environment Variables

For any application:
1. Go to application settings
2. Click "Environment Variables"
3. Add variables:
   ```
   DATABASE_URL=postgresql://...
   API_KEY=your-secret-key
   NODE_ENV=production
   ```
4. Click "Save"
5. Redeploy application (environment updated on restart)

### Best Practices

1. **Never commit secrets to Git**
2. **Store secrets in Coolify environment variables**
3. **Use strong random values** for API keys
4. **Rotate secrets regularly**
5. **Different secrets for dev/staging/production**

---

## Step 12: Auto-Deployment from Git

Set up automatic deployments when you push to Git.

### Enable Auto-Deploy

1. **Go to application settings**
2. **Find "Auto Deploy" section**
3. **Enable**: "Deploy on Push"
4. **Save**

### Configure Git Webhook

1. **In Coolify**: Copy webhook URL
   - Example: `https://yourserver/webhooks/deploy/abc123`

2. **In GitHub** (or GitLab/Bitbucket):
   - Go to repository settings
   - Click "Webhooks"
   - Add webhook:
     - URL: Paste Coolify webhook URL
     - Content type: application/json
     - Events: "Just the push event"
     - Active: Yes

3. **Test**: Push to your repository
   - Should trigger automatic deployment in Coolify
   - Watch deployment logs in Coolify UI

✅ **Success**: Now `git push` automatically deploys!

---

## Step 13: Resource Limits and Monitoring

Prevent applications from consuming all resources.

### Set Resource Limits

For each application:
1. Go to application settings
2. Find "Resources" section
3. Set limits:
   ```
   Memory Limit: 512MB
   Memory Reservation: 256MB
   CPU Limit: 1.0 (1 core)
   ```
4. Save and redeploy

### Monitor Resources

```bash
# View all containers and resource usage
docker stats

# View specific container
docker stats CONTAINER_NAME

# View Coolify's resource usage
docker ps | grep coolify
docker stats coolify
```

### In Coolify Dashboard

- View overall server metrics (CPU, memory, disk)
- View per-application metrics
- Set up alerts (if available in your Coolify version)

---

## Step 14: Backups

Protect your data with regular backups.

### Database Backups

**In Coolify**:
1. Go to database resource
2. Click "Backups" tab
3. Configure backup:
   - Frequency: Daily
   - Retention: 7 days
   - Destination: Local or S3

**Manual backup**:
```bash
# Get database container name
docker ps | grep postgres

# Create backup
docker exec CONTAINER_NAME pg_dump -U user dbname > backup-$(date +%Y%m%d).sql

# Store backup securely offsite
scp backup-*.sql your-backup-location/
```

### Application Backups

**Git-based apps**: Code is in Git (already backed up)
**Persistent data**: Configure volume backups in Coolify

### Coolify Configuration Backup

```bash
# Backup Coolify database (contains all your configs)
docker exec coolify-db pg_dump -U postgres coolify > coolify-backup.sql

# Store securely
```

---

## Step 15: SSL/TLS Certificates

Understanding HTTPS certificate management.

### Automatic HTTPS

For public apps with domains:
1. Add domain in Coolify
2. Ensure DNS points to your server
3. Coolify/Traefik automatically:
   - Contacts Let's Encrypt
   - Proves domain ownership (HTTP challenge)
   - Gets certificate
   - Installs certificate
   - Auto-renews before expiration

### Troubleshooting Certificates

**Certificate not obtained**:
```bash
# Check DNS resolves correctly
dig +short yourdomain.com
# Must show your server IP

# Check port 80 is accessible (needed for verification)
curl -I http://yourdomain.com
# Should connect

# Check Traefik logs
docker logs coolify-proxy | grep -i certificate

# Common issues:
# - DNS not pointing to server
# - Port 80 blocked
# - Domain already has certificate elsewhere
# - Rate limit hit (wait and retry)
```

### View Certificates

```bash
# Check Traefik certificate storage
docker exec coolify-proxy ls -la /letsencrypt/acme.json

# View certificate expiration
echo | openssl s_client -servername yourdomain.com -connect yourdomain.com:443 2>/dev/null | openssl x509 -noout -dates
```

---

## Final Verification

Create a comprehensive check script:

```bash
# Create verification script
tee ~/coolify-check.sh > /dev/null << 'EOF'
#!/bin/bash
echo "=== Coolify Stack Status ==="
echo ""

echo "Docker:"
systemctl is-active docker && echo "✓ Docker active" || echo "✗ Inactive"
echo ""

echo "Coolify Containers:"
docker ps --filter "name=coolify" --format "{{.Names}}: {{.Status}}" | while read line; do
    echo "✓ $line"
done
echo ""

echo "Application Containers:"
COOL_COUNT=$(docker ps --filter "name=coolify" --format "{{.Names}}" | wc -l)
APP_COUNT=$(docker ps | grep -v CONTAINER | grep -v coolify | wc -l)
echo "Coolify containers: $COOL_COUNT"
echo "Application containers: $APP_COUNT"
echo ""

echo "Firewall - Public Zone:"
sudo firewall-cmd --zone=public --list-services | grep -q "http\|https" && echo "✓ HTTP/HTTPS allowed" || echo "✗ Not allowed"
sudo firewall-cmd --zone=public --list-ports | grep -q 8000 && echo "⚠ Port 8000 PUBLIC (should be VPN-only!)" || echo "✓ Port 8000 not public"
echo ""

echo "VPN Access:"
tailscale status >/dev/null 2>&1 && echo "✓ Tailscale active" || echo "✗ Tailscale inactive"
echo "Coolify UI: http://$(tailscale ip -4):8000"
echo ""

echo "Disk Usage:"
df -h / | tail -1 | awk '{print "Used: "$5" of "$2}'
echo ""

echo "Memory Usage:"
free -h | grep Mem | awk '{print "Used: "$3" of "$2}'
echo ""

echo "Public Applications (with domains):"
docker ps --format "{{.Names}}" | while read container; do
    DOMAIN=$(docker inspect $container 2>/dev/null | grep -oP 'traefik.http.routers.\w+.rule=Host\(`\K[^`]+' | head -1)
    if [ ! -z "$DOMAIN" ]; then
        echo "  - $container → https://$DOMAIN"
    fi
done
echo ""

echo "VPN-Only Applications (no domains):"
docker ps --format "{{.Names}}" | grep -v coolify | while read container; do
    DOMAIN=$(docker inspect $container 2>/dev/null | grep -oP 'traefik.http.routers.\w+.rule=Host\(`\K[^`]+' | head -1)
    if [ -z "$DOMAIN" ]; then
        PORT=$(docker port $container 2>/dev/null | head -1 | cut -d':' -f2)
        if [ ! -z "$PORT" ]; then
            echo "  - $container → http://$(tailscale ip -4):$PORT"
        fi
    fi
done
EOF

chmod +x ~/coolify-check.sh
~/coolify-check.sh
```

**Expected output**:
- ✓ Docker active
- ✓ Coolify containers running
- ✓ HTTP/HTTPS allowed
- ✓ Port 8000 not public
- ✓ Tailscale active
- List of public apps
- List of VPN-only apps

---

## Testing Checklist

### Test 1: Coolify UI Access (VPN)
```bash
# From local machine with Tailscale:
curl http://100.x.x.x:8000
# ✅ Should load Coolify UI
```

### Test 2: Coolify UI NOT Public
```bash
# From local machine (disable Tailscale):
curl http://YOUR_PUBLIC_IP:8000 --connect-timeout 5
# ❌ Should timeout or refuse connection
```

### Test 3: Public App Accessible
```bash
# From anywhere (no VPN needed):
curl https://hello.example.com
# ✅ Should load your public app
```

### Test 4: VPN-Only App NOT Public
```bash
# From anywhere (no VPN):
curl http://YOUR_PUBLIC_IP:3001 --connect-timeout 5
# ❌ Should timeout or refuse connection
```

### Test 5: VPN-Only App Via VPN
```bash
# From local machine with Tailscale:
curl http://100.x.x.x:3001
# ✅ Should load your VPN-only app
```

### Test 6: SSH Still Works
```bash
# Via VPN or public (depending on guide 02 config)
ssh fedora@100.x.x.x  # VPN
# or
ssh fedora@YOUR_PUBLIC_IP  # Public (if not restricted)
# ✅ Must always work!
```

---

## Troubleshooting

### Coolify Not Accessible

```bash
# Check Coolify containers
docker ps | grep coolify

# If containers not running:
cd /data/coolify/source
docker compose up -d

# Check logs
docker logs coolify

# Check what's using port 8000
sudo ss -tlnp | grep :8000

# Access via local machine with VPN:
curl http://100.x.x.x:8000
```

### Public App Not Accessible

**Check DNS**:
```bash
dig +short yourdomain.com
# Must show your server IP
```

**Check Traefik routing**:
```bash
# View Traefik logs
docker logs coolify-proxy

# Check container labels
docker inspect YOUR_APP_CONTAINER | grep traefik
```

**Check firewall**:
```bash
sudo firewall-cmd --zone=public --list-services
# Must show: http https
```

**Check certificate**:
```bash
# View certificate logs
docker logs coolify-proxy | grep -i certificate

# Common issues:
# - DNS not pointing to server (wait for propagation)
# - Port 80 blocked (needed for Let's Encrypt)
# - Domain already has certificate elsewhere
```

### VPN-Only App Not Accessible

```bash
# Check Tailscale is connected
tailscale status

# Check app container is running
docker ps | grep YOUR_APP_NAME

# Check app port mapping
docker port YOUR_APP_CONTAINER

# Try accessing from server itself first
curl http://localhost:3001

# Then try via Tailscale IP
curl http://100.x.x.x:3001
```

### Database Connection Failed

```bash
# Check database container is running
docker ps | grep postgres  # or mysql, mongo, redis

# Check database logs
docker logs YOUR_DB_CONTAINER

# Test connection from server
docker exec YOUR_DB_CONTAINER psql -U user -d dbname -c "SELECT 1;"

# For apps: use internal Docker network hostname
# Not: postgresql://100.x.x.x:5432/db
# Use: postgresql://postgres:5432/db
```

### Port Conflicts

```bash
# Check what's using a port
sudo ss -tlnp | grep :PORT_NUMBER

# Stop conflicting service
sudo systemctl stop SERVICE_NAME

# Or in Coolify: change app port in settings
```

### Out of Disk Space

```bash
# Check disk usage
df -h

# Clean Docker
docker system prune -a --volumes
# WARNING: This removes unused containers, images, volumes

# Check large directories
du -sh /var/lib/docker/*
du -sh /data/coolify/*

# Clean old logs
sudo journalctl --vacuum-time=7d
```

### Out of Memory

```bash
# Check memory
free -h

# Check container memory usage
docker stats --no-stream

# Set memory limits in Coolify
# Go to app → Resources → Set memory limit

# Restart high-memory containers
docker restart CONTAINER_NAME
```

---

## Understanding Your Setup

### Complete Architecture

```
                         INTERNET
                            |
                            | Port 80/443 (Firewall allows)
                            |
                            ▼
                    ┌───────────────┐
                    │   Traefik     │
                    │ Reverse Proxy │
                    └───────┬───────┘
                            |
        ┌───────────────────┼───────────────────┐
        |                   |                   |
        ▼                   ▼                   ▼
  [Public App 1]      [Public App 2]      [WordPress]
  hello.example.com   api.example.com     blog.example.com
  Port 3000           Port 4000           Port 80
        |                   |                   |
        └───────────────────┴───────────────────┘
                            |
                    [Docker Network]


                      TAILSCALE VPN
                    (100.x.x.x network)
                            |
        ┌───────────────────┼───────────────────┐
        |                   |                   |
        ▼                   ▼                   ▼
  [Coolify UI]      [Admin Panel]        [Database]
  Port 8000         Port 3001            Port 5432
  VPN-only          VPN-only             VPN-only
```

### Traffic Flow Examples

**Public Website**:
1. User visits `https://hello.example.com`
2. DNS resolves to your server public IP
3. Request hits firewall (port 443 allowed)
4. Traefik receives request
5. Traefik checks: "Host is hello.example.com"
6. Traefik routes to container with matching label
7. Container responds
8. Traefik sends response back

**VPN-Only Admin Panel**:
1. You connect to Tailscale VPN
2. You visit `http://100.x.x.x:3001`
3. Request goes through VPN tunnel
4. Reaches server on trusted zone (all ports allowed)
5. Directly connects to container port 3001
6. No Traefik involved
7. Not accessible from internet

**Coolify UI**:
1. You connect to Tailscale VPN
2. You visit `http://100.x.x.x:8000`
3. Request goes through VPN tunnel
4. Directly connects to Coolify container
5. Coolify serves web interface
6. Completely hidden from internet

---

## Summary

✅ **Docker**: Container platform installed
✅ **Coolify**: PaaS platform installed and configured
✅ **Coolify UI**: VPN-only access (secure)
✅ **Public Apps**: Deploy with domains, automatic HTTPS
✅ **VPN-Only Apps**: Deploy without domains, VPN access
✅ **Databases**: VPN-only by default
✅ **Traefik**: Automatic routing and SSL
✅ **Auto-Deploy**: Git webhook integration
✅ **Backups**: Database and config backup strategies

**Key Concepts Learned**:
1. **Single platform**: Coolify handles everything (no separate Caddy needed)
2. **Built-in reverse proxy**: Traefik routes public traffic
3. **VPN-only by default**: Only public apps with domains are exposed
4. **Automatic HTTPS**: Let's Encrypt integration
5. **Container isolation**: Each app in its own container
6. **Resource management**: Set limits per app
7. **Security model**: Clear separation of public vs private

---

## Best Practices Summary

### Security

1. ✅ **Coolify UI**: Always VPN-only (never public)
2. ✅ **Databases**: Always VPN-only (never public)
3. ✅ **Admin panels**: VPN-only (no domains)
4. ✅ **Public apps**: Only those that need to be public
5. ✅ **Secrets**: Store in Coolify environment variables
6. ✅ **Strong passwords**: For Coolify and databases
7. ✅ **Regular updates**: Check for Coolify updates

### Operations

1. ✅ **Auto-deploy**: Enable for Git-based apps
2. ✅ **Resource limits**: Set per application
3. ✅ **Monitoring**: Check resource usage regularly
4. ✅ **Backups**: Automate database backups
5. ✅ **Logs**: Monitor application and system logs
6. ✅ **Testing**: Test deployments in dev before production
7. ✅ **Documentation**: Document your deployments

### Performance

1. ✅ **Resource limits**: Prevent resource hogging
2. ✅ **Image optimization**: Use slim base images
3. ✅ **Cleanup**: Regular `docker system prune`
4. ✅ **Monitoring**: Track CPU, memory, disk usage
5. ✅ **Scaling**: Deploy multiple instances if needed

---

## Next Steps

### Deploy Your Applications

Now you can deploy:
- **Static websites**: HTML, React, Vue, Next.js
- **Backend APIs**: Node.js, Python, Go, PHP
- **Full-stack apps**: With databases
- **One-click services**: WordPress, Ghost, etc.

### Recommended Next Deployments

1. **Uptime monitoring** (VPN-only):
   - Deploy Uptime Kuma
   - Monitor all your services
   - Get alerts when services go down

2. **Analytics** (public):
   - Deploy Plausible or Umami
   - Privacy-friendly analytics for your sites

3. **Your actual projects**:
   - Public websites/apps with domains
   - Admin panels VPN-only
   - Databases for your apps

### Future Guides

1. **Guide 04**: Monitoring and Observability
   - Log aggregation
   - Metrics and dashboards
   - Alerting

2. **Guide 05**: CI/CD Pipelines
   - GitHub Actions integration
   - Automated testing
   - Advanced deployment strategies

3. **Guide 06**: Advanced Docker
   - Custom Dockerfiles
   - Multi-stage builds
   - Optimization techniques

---

## Quick Reference

### Coolify Access
```bash
# Get Tailscale IP
tailscale ip -4

# Access Coolify UI
http://100.x.x.x:8000
```

### Docker Commands
```bash
# List all containers
docker ps -a

# View container logs
docker logs CONTAINER_NAME
docker logs -f CONTAINER_NAME  # Follow

# Restart container
docker restart CONTAINER_NAME

# View resource usage
docker stats

# Clean up
docker system prune -a
```

### Firewall Commands
```bash
# Check public services
sudo firewall-cmd --zone=public --list-all

# Verify port 8000 not public
sudo firewall-cmd --zone=public --list-ports | grep 8000
# Should show nothing
```

### Coolify Operations
```bash
# Restart Coolify
cd /data/coolify/source
docker compose restart

# View Coolify logs
docker logs coolify

# Backup Coolify database
docker exec coolify-db pg_dump -U postgres coolify > coolify-backup.sql

# Update Coolify (check their docs for latest method)
```

### Traefik
```bash
# View Traefik logs
docker logs coolify-proxy

# View Traefik config
docker exec coolify-proxy cat /etc/traefik/traefik.yml

# Check certificates
docker exec coolify-proxy ls -la /letsencrypt/
```

---

## Additional Resources

- **Coolify Documentation**: https://coolify.io/docs
- **Coolify GitHub**: https://github.com/coollabsio/coolify
- **Coolify Discord**: https://coolify.io/discord
- **Docker Documentation**: https://docs.docker.com/
- **Traefik Documentation**: https://doc.traefik.io/traefik/
- **Let's Encrypt**: https://letsencrypt.org/

---

**Congratulations!** 🎉

You now have a complete, production-ready application platform:
- ✅ Deploy apps in minutes via web UI
- ✅ Automatic HTTPS for public apps
- ✅ Secure VPN-only access for admin interfaces
- ✅ Database management
- ✅ Resource monitoring and limits
- ✅ Auto-deployment from Git

Your server is ready to host any application you can think of, with a clear security model: **public when needed, private by default**!
