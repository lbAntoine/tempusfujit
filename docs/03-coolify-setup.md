# Coolify Setup and Configuration

## Overview

Coolify is an open-source, self-hostable alternative to Heroku/Netlify/Vercel. It allows you to deploy applications, databases, and services with a user-friendly interface.

## What is Coolify?

Coolify provides:
- **Application deployment**: Node.js, Python, PHP, Go, Rust, static sites, Docker images
- **Database management**: PostgreSQL, MySQL, MongoDB, Redis, and more
- **Automatic SSL**: Built-in Let's Encrypt integration
- **Git integration**: Deploy from GitHub, GitLab, Bitbucket
- **Environment management**: Multiple environments per application
- **Backup system**: Database backups and restores
- **Resource monitoring**: CPU, memory, disk usage

## Prerequisites

- ✅ Fedora 43 Server (you have this)
- ✅ Root SSH access (required by Coolify)
- ✅ Minimum 2GB RAM (you have 32GB - plenty)
- ✅ Docker installed
- ✅ Domain names configured (antoinelb.fr, vaultofsuraya.com)
- ✅ Reverse proxy (Caddy recommended - see previous guide)

## Architecture Options

### Option 1: Coolify with Caddy (Recommended)

```
Internet → Caddy (ports 80/443) → Coolify & Apps (localhost ports)
```

**Pros**:
- Single reverse proxy (simpler)
- Consistent SSL management
- Full control over routing
- Works well with VPN

**Cons**:
- Manual Caddy config for each app
- Coolify won't auto-configure domains

### Option 2: Coolify with Built-in Proxy

```
Internet → Coolify Proxy (ports 80/443) → Apps
```

**Pros**:
- Coolify manages everything
- Automatic domain configuration
- No extra proxy needed

**Cons**:
- Two proxy layers if you want Caddy too
- Less flexibility

**Recommendation**: Use Option 1 (Caddy + Coolify without built-in proxy) for consistency with your other services.

## Installation Guide

### Step 1: Prepare the System

```bash
# Update system
sudo dnf update -y

# Install required packages
sudo dnf install -y curl wget git

# Install Docker (if not already installed)
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Enable and start Docker
sudo systemctl enable --now docker

# Verify Docker installation
sudo docker --version
sudo docker compose version
```

### Step 2: Configure Docker for Non-Root (Optional but Recommended)

```bash
# Create docker group and add your user
sudo groupadd -f docker
sudo usermod -aG docker $USER

# Apply group changes (or log out and back in)
newgrp docker

# Test Docker without sudo
docker ps
```

### Step 3: Install Coolify

```bash
# Run the official installation script
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash

# The script will:
# 1. Install required dependencies
# 2. Download and set up Coolify
# 3. Start Coolify services
# 4. Display the access URL
```

**Installation takes ~5-10 minutes.**

### Step 4: Initial Access

Once installation completes, Coolify will be accessible at:
```
http://YOUR_SERVER_IP:8000
```

**First-time setup**:
1. Create admin account (username, email, password)
2. Set up your server connection
3. Configure your first project

### Step 5: Configure Reverse Proxy (Caddy)

Add Coolify to your Caddyfile:

```bash
sudo nano /etc/caddy/Caddyfile
```

```caddyfile
# Coolify admin panel
coolify.antoinelb.fr {
    reverse_proxy localhost:8000

    # Optional: Only accessible via VPN
    # @notlocal {
    #     not remote_ip 100.64.0.0/10  # Tailscale IP range
    # }
    # respond @notlocal 403
}
```

Reload Caddy:
```bash
sudo systemctl reload caddy
```

### Step 6: Disable Coolify's Built-in Proxy (Optional)

If using Caddy as your main proxy:

1. Navigate to Coolify UI: Settings → Configuration
2. Disable "Use Coolify Proxy"
3. This frees up ports 80/443 for Caddy

**Note**: You'll need to manually add Caddy entries for each app you deploy.

### Step 7: Configure SSH Access

Coolify needs SSH access to manage Docker and services.

#### Option A: Root SSH (Simplest)

Coolify documentation recommends root SSH access.

```bash
# Ensure root can SSH (already should be enabled on your setup)
# Coolify will use the same machine, so it's localhost SSH
```

Coolify will SSH to `localhost` as root to manage containers.

#### Option B: Non-Root with Docker Access (More Secure)

Create a dedicated user:

```bash
# Create coolify user
sudo useradd -m -G docker coolify

# Set up SSH key for the user
sudo -u coolify ssh-keygen -t ed25519 -C "coolify@server" -f /home/coolify/.ssh/id_ed25519 -N ""

# Add to authorized_keys
sudo -u coolify bash -c 'cat /home/coolify/.ssh/id_ed25519.pub >> /home/coolify/.ssh/authorized_keys'
sudo chmod 600 /home/coolify/.ssh/authorized_keys

# Test SSH
sudo -u coolify ssh coolify@localhost
```

In Coolify, configure server connection with:
- Host: `localhost`
- Port: `22`
- User: `coolify`
- Private key: Contents of `/home/coolify/.ssh/id_ed25519`

## Coolify Configuration

### Server Settings

1. **Go to**: Settings → Servers
2. **Add Server**:
   - Name: `Production`
   - IP: `localhost` or `127.0.0.1`
   - Port: `22`
   - User: `root` or `coolify`
   - SSH Key: Paste your private key or use existing

3. **Validate Connection**: Click "Validate" to ensure Coolify can connect

### Resource Settings

Configure Docker networks and default settings:

1. **Go to**: Settings → Configuration
2. **Docker Network**: Leave as default or customize
3. **Cleanup Settings**:
   - Enable automatic cleanup of old images
   - Set retention period (e.g., 30 days)

### Notification Settings

Set up notifications for deployment events:

1. **Go to**: Settings → Notifications
2. **Add**: Email, Discord, Slack, or Telegram
3. **Configure**: Deployment success/failure notifications

## Deploying Your First Application

### Example 1: Deploy a Node.js Application

1. **Create Project**:
   - Go to Projects → New Project
   - Name: `My Node App`
   - Description: `Production Node.js application`

2. **Add Application**:
   - Click "New Resource" → "Application"
   - Choose "Public Repository" (or connect GitHub)
   - Repository: `https://github.com/your-username/your-app`
   - Branch: `main`

3. **Configure Build**:
   - **Build Pack**: `nixpacks` (auto-detects Node.js)
   - **Port**: `3000` (or your app's port)
   - **Start Command**: `npm start` (auto-detected)
   - **Install Command**: `npm install` (auto-detected)

4. **Environment Variables**:
   - Click "Environment" tab
   - Add variables: `NODE_ENV=production`, `PORT=3000`, etc.

5. **Domains**:
   - If using Coolify proxy: Add `app.antoinelb.fr`
   - If using Caddy: Leave empty, configure Caddy manually

6. **Deploy**:
   - Click "Deploy"
   - Watch build logs in real-time
   - Once complete, app runs on configured port

7. **Add to Caddy** (if using Caddy):
   ```caddyfile
   app.antoinelb.fr {
       reverse_proxy localhost:3000
   }
   ```

### Example 2: Deploy a Static Site

1. **Create Application**:
   - New Resource → Application
   - Repository: `https://github.com/your-username/portfolio`

2. **Configure**:
   - **Build Pack**: `static`
   - **Build Command**: `npm run build`
   - **Publish Directory**: `dist` or `build`
   - **Port**: `80` (Coolify serves static files)

3. **Deploy and Configure Caddy**:
   ```caddyfile
   portfolio.antoinelb.fr {
       reverse_proxy localhost:8001  # Check Coolify-assigned port
   }
   ```

### Example 3: Deploy a Database

1. **New Resource → Database**
2. **Choose Type**:
   - PostgreSQL
   - MySQL
   - MongoDB
   - Redis
   - MariaDB

3. **Configure**:
   - Name: `production-db`
   - Version: Latest or specific version
   - Root password: Auto-generated or custom
   - Port: Default or custom

4. **Create Database**:
   - Coolify provisions the database
   - Provides connection details
   - Automatic daily backups (configurable)

5. **Connect from Application**:
   ```env
   DATABASE_URL=postgresql://user:password@localhost:5432/dbname
   ```

## Docker Compose Deployments

Coolify supports Docker Compose for complex applications.

### Example: Deploy with Docker Compose

1. **New Resource → Docker Compose**
2. **Paste your `docker-compose.yml`**:
   ```yaml
   version: '3.8'
   services:
     web:
       image: nginx:alpine
       ports:
         - "8080:80"
     app:
       build: .
       ports:
         - "3000:3000"
       environment:
         - NODE_ENV=production
   ```

3. **Deploy**:
   - Coolify builds and runs all services
   - Manages container lifecycle
   - Provides logs for each service

## Integration with Git

### GitHub Integration

1. **Go to**: Settings → GitHub
2. **Install GitHub App**:
   - Click "Install GitHub App"
   - Authorize Coolify to access your repos
   - Select repositories

3. **Deploy from Private Repo**:
   - Choose "Private Repository"
   - Select from connected repos
   - Choose branch
   - Deploy!

### Automatic Deployments (Webhooks)

1. **In Application Settings**: Enable "Automatic Deploy on Push"
2. **Coolify provides webhook URL**
3. **Add to GitHub**:
   - Repo Settings → Webhooks → Add webhook
   - Paste Coolify webhook URL
   - Select "Just the push event"
   - Save

Now every push to main triggers automatic deployment!

### Deploy Keys

For SSH-based deployments:
1. Coolify generates an SSH deploy key
2. Add to GitHub: Repo → Settings → Deploy keys
3. Paste public key from Coolify

## Database Backups

### Automatic Backups

1. **Go to**: Database → Settings → Backups
2. **Configure**:
   - **Frequency**: Daily, Weekly, or Custom cron
   - **Retention**: Keep last X backups
   - **S3 Integration** (optional): Backup to S3-compatible storage

3. **Example Cron**:
   - `0 2 * * *` - Daily at 2 AM
   - `0 2 * * 0` - Weekly on Sunday at 2 AM

### Manual Backups

1. Go to Database → Backups
2. Click "Backup Now"
3. Download or store remotely

### Restore from Backup

1. Go to Database → Backups
2. Select backup
3. Click "Restore"
4. Confirm (this will overwrite current data)

## Advanced Configuration

### Environment Templates

Create reusable environment configurations:

1. **Go to**: Settings → Environment Templates
2. **Create Template**:
   ```env
   NODE_ENV=production
   LOG_LEVEL=info
   DATABASE_URL=${DATABASE_URL}
   REDIS_URL=${REDIS_URL}
   ```
3. **Use in Applications**: Select template when creating new apps

### Build Packs

Coolify uses Nixpacks by default (auto-detects language):
- Node.js
- Python
- Go
- Rust
- PHP
- Static

For custom builds:
- Use Dockerfile
- Or Docker Compose

### Resource Limits

Set CPU and memory limits:

1. **Go to**: Application → Settings → Resources
2. **Configure**:
   - Memory limit: `512M`, `1G`, `2G`
   - CPU shares: Relative weight

### Health Checks

Configure application health checks:

1. **Go to**: Application → Settings → Health Check
2. **Configure**:
   - Health check URL: `/health` or `/api/health`
   - Interval: 30s
   - Timeout: 5s
   - Retries: 3

Coolify automatically restarts unhealthy containers.

## Security Best Practices

### 1. Limit Coolify Access

**Via VPN only**:
```caddyfile
coolify.antoinelb.fr {
    @notlocal {
        not remote_ip 100.64.0.0/10  # Tailscale range
    }
    respond @notlocal 403

    reverse_proxy localhost:8000
}
```

### 2. Use Strong Passwords

- Enable 2FA in Coolify (if available)
- Use password manager for database credentials
- Rotate secrets regularly

### 3. Isolate Databases

- Run databases in separate Docker networks
- Don't expose database ports publicly
- Use Coolify's internal networking

### 4. Secrets Management

Store secrets in Coolify:
1. Go to Project → Secrets
2. Add secret: `API_KEY=secret_value`
3. Reference in apps: `${API_KEY}`

### 5. Regular Updates

```bash
# Update Coolify
curl -fsSL https://cdn.coollabs.io/coolify/upgrade.sh | bash

# Update every month or when security patches released
```

### 6. Audit Logs

Check Coolify logs regularly:
```bash
docker logs -f coolify
```

## Integration with Your Infrastructure

### With Tailscale VPN

1. Install Tailscale on server (see VPN guide)
2. Configure Coolify to listen only on Tailscale IP
3. Access Coolify via: `http://server-tailscale-ip:8000`
4. Or use Caddy with VPN restrictions

### With Monitoring

Coolify provides basic metrics. For advanced monitoring:
- Export Docker metrics to Prometheus
- Monitor Coolify container: `coolify`
- Alert on deployment failures
- Track resource usage trends

### With Backups

Backup strategy:
1. **Coolify Database**: Daily backup of Coolify's own database
   ```bash
   docker exec coolify-db pg_dump -U coolify > coolify-backup.sql
   ```

2. **Application Databases**: Use Coolify's built-in backup feature

3. **Docker Volumes**: Regular snapshots
   ```bash
   rsync -av /var/lib/docker/volumes/ /backup/docker-volumes/
   ```

## Troubleshooting

### Coolify Won't Start

```bash
# Check Docker status
sudo systemctl status docker

# Check Coolify containers
docker ps -a | grep coolify

# View Coolify logs
docker logs coolify

# Restart Coolify
docker restart coolify
```

### Deployment Failed

1. **Check Build Logs**: Coolify UI → Application → Logs
2. **Common Issues**:
   - Missing environment variables
   - Build command errors
   - Port conflicts
   - Out of memory

### Can't Access Application

```bash
# Check if container is running
docker ps | grep your-app-name

# Check container logs
docker logs <container-id>

# Check port binding
sudo netstat -tlnp | grep <port>

# Verify Caddy configuration
sudo caddy validate --config /etc/caddy/Caddyfile
```

### Database Connection Issues

1. **Verify database is running**: `docker ps | grep postgres`
2. **Check connection string**: Ensure correct host, port, credentials
3. **Network issues**: Verify containers are on same Docker network
4. **Firewall**: Ensure localhost connections allowed

### Out of Disk Space

```bash
# Clean up old Docker images
docker system prune -a

# Check disk usage
df -h

# Clean up old Coolify deployments
# Via Coolify UI: Settings → Cleanup
```

## Maintenance Tasks

### Weekly
- Check deployment logs for failures
- Review resource usage
- Verify backups are running

### Monthly
- Update Coolify: `curl -fsSL https://cdn.coollabs.io/coolify/upgrade.sh | bash`
- Clean up old Docker images: `docker system prune -a`
- Review and rotate secrets

### Quarterly
- Audit user access
- Review and update SSL certificates (auto, but verify)
- Test disaster recovery (restore from backup)

## Coolify CLI

Coolify provides a CLI for automation:

```bash
# Install CLI
npm install -g @coollabs/coolify-cli

# Login
coolify login

# Deploy application
coolify deploy --project myapp --branch main

# View logs
coolify logs --project myapp

# List resources
coolify list projects
coolify list applications
```

## Migration to Coolify

### From Heroku

1. Export Heroku environment variables
2. Create application in Coolify
3. Import environment variables
4. Connect same Git repository
5. Deploy!

### From Manual Docker Setup

1. Convert your `docker run` commands to Coolify config
2. Or use Docker Compose mode
3. Migrate databases using dump/restore
4. Test thoroughly before switching

## Scaling Considerations

### Vertical Scaling (Current Server)

Your server (32GB RAM, 2TB storage) can handle:
- 10-20 small applications
- 5-10 medium applications (Node.js/Python with databases)
- Multiple databases
- Still have resources for monitoring, backups

### Horizontal Scaling (Future)

When you outgrow one server:
1. Add another server to Coolify
2. Distribute applications across servers
3. Use load balancing (Caddy can do this)

## Resources and Documentation

- **Official Docs**: https://coolify.io/docs
- **GitHub**: https://github.com/coollabs/coolify
- **Discord**: Active community support
- **YouTube**: Coolify tutorials and walkthroughs

## Next Steps

1. Install and configure Coolify
2. Deploy a test application
3. Set up database backups
4. Configure Caddy for your apps
5. Move on to remote development environment setup

## Example Deployment Workflow

Here's a complete workflow for deploying a full-stack app:

```bash
# 1. Create PostgreSQL database in Coolify
# Via UI: New Resource → Database → PostgreSQL

# 2. Create backend application
# Via UI: New Resource → Application
# Repository: github.com/you/backend
# Port: 3000
# Environment:
DATABASE_URL=postgresql://user:pass@localhost:5432/mydb
NODE_ENV=production

# 3. Deploy backend
# Click Deploy, wait for completion

# 4. Add to Caddy
```
```caddyfile
api.antoinelb.fr {
    reverse_proxy localhost:3000
}
```

```bash
# 5. Create frontend application
# Via UI: New Resource → Application
# Repository: github.com/you/frontend
# Build command: npm run build
# Environment:
VITE_API_URL=https://api.antoinelb.fr

# 6. Deploy frontend

# 7. Add to Caddy
```
```caddyfile
app.antoinelb.fr {
    reverse_proxy localhost:3001
}
```

```bash
# 8. Test
curl https://api.antoinelb.fr/health
curl https://app.antoinelb.fr

# 9. Set up auto-deploy webhooks
# 10. Monitor and enjoy!
```

That's it! You now have a complete Coolify deployment pipeline.
