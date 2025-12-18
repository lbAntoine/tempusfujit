# Reverse Proxy Solutions

## Overview

A reverse proxy sits in front of your services and handles:
- **SSL/TLS certificates**: Automatic HTTPS with Let's Encrypt
- **Domain routing**: Route different domains to different services
- **Load balancing**: Distribute traffic (if needed in future)
- **Security**: Hide internal services, add authentication

## Why You Need This

With multiple services (Coolify, remote coding, monitoring) and two domains, you need something to:
1. Route `coolify.antoinelb.fr` → Coolify instance
2. Route `dev.antoinelb.fr` → Remote coding environment
3. Route `monitoring.vaultofsuraya.com` → Grafana dashboard
4. Automatically manage SSL certificates for all domains
5. Renew certificates before expiry

## Solution Comparison

### Solution 1: Caddy (Recommended)

#### What is Caddy?

Modern web server with automatic HTTPS built-in. Simplest configuration of all reverse proxies.

#### Pros

- ✅ **Automatic HTTPS**: Zero configuration for Let's Encrypt certificates
- ✅ **Simple config**: Most human-readable configuration (Caddyfile)
- ✅ **Zero dependencies**: Single binary, no external requirements
- ✅ **HTTP/3 support**: Latest protocol support out of the box
- ✅ **Automatic cert renewal**: Never worry about expired certificates
- ✅ **Security by default**: Secure TLS defaults
- ✅ **API available**: Can configure via API if needed
- ✅ **Active development**: Modern, well-maintained project
- ✅ **Low resource usage**: Efficient and lightweight
- ✅ **Easy debugging**: Clear error messages

#### Cons

- ❌ **Less ecosystem**: Smaller plugin ecosystem vs Nginx
- ❌ **Less enterprise adoption**: Newer than Nginx/Traefik
- ❌ **Learning curve**: If you know Nginx, need to learn Caddy syntax

#### Configuration Example

```caddyfile
# /etc/caddy/Caddyfile

coolify.antoinelb.fr {
    reverse_proxy localhost:3000
}

dev.antoinelb.fr {
    reverse_proxy localhost:8080
}

monitoring.vaultofsuraya.com {
    reverse_proxy localhost:3001
}
```

That's it! Caddy automatically:
- Gets SSL certificates from Let's Encrypt
- Handles HTTP → HTTPS redirect
- Renews certificates before expiry
- Sets secure TLS defaults

#### Use Cases

- Perfect for: Straightforward reverse proxy needs
- Best when: You want automatic HTTPS with minimal config
- Ideal for: Your use case (multiple services, automatic SSL)

#### Resource Usage

- RAM: ~20-50MB
- CPU: Minimal

### Solution 2: Traefik

#### What is Traefik?

Cloud-native reverse proxy designed for containers and microservices. Automatically discovers services.

#### Pros

- ✅ **Container-first**: Automatic service discovery with Docker
- ✅ **Dynamic configuration**: No restart needed for new services
- ✅ **Dashboard**: Built-in web UI for monitoring
- ✅ **Let's Encrypt**: Automatic SSL certificates
- ✅ **Middlewares**: Easy to add authentication, rate limiting, etc.
- ✅ **Metrics**: Prometheus metrics built-in
- ✅ **Multiple providers**: Kubernetes, Docker, file-based config
- ✅ **Load balancing**: Advanced load balancing features
- ✅ **Cloudflare integration**: Works well with Cloudflare DNS

#### Cons

- ❌ **Complexity**: Steeper learning curve than Caddy
- ❌ **Configuration can be verbose**: Especially for simple use cases
- ❌ **Memory usage**: Uses more resources than Caddy
- ❌ **Documentation**: Can be overwhelming for beginners
- ❌ **Debugging**: Harder to debug than simple configs

#### Configuration Example

```yaml
# /etc/traefik/traefik.yml
entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
  websecure:
    address: ":443"

certificatesResolvers:
  letsencrypt:
    acme:
      email: you@antoinelb.fr
      storage: /letsencrypt/acme.json
      httpChallenge:
        entryPoint: web

providers:
  docker:
    exposedByDefault: false
  file:
    directory: /etc/traefik/dynamic
```

```yaml
# /etc/traefik/dynamic/services.yml
http:
  routers:
    coolify:
      rule: "Host(`coolify.antoinelb.fr`)"
      service: coolify
      tls:
        certResolver: letsencrypt

  services:
    coolify:
      loadBalancer:
        servers:
          - url: "http://localhost:3000"
```

#### Use Cases

- Perfect for: Container-heavy infrastructure, microservices
- Best when: You want automatic service discovery
- Ideal for: Complex routing needs, Kubernetes environments

#### Resource Usage

- RAM: ~50-100MB
- CPU: Low to moderate

### Solution 3: Nginx Proxy Manager (NPM)

#### What is Nginx Proxy Manager?

Nginx with a friendly web UI. Combines Nginx power with point-and-click configuration.

#### Pros

- ✅ **Web UI**: No need to edit config files
- ✅ **User-friendly**: Visual interface for everything
- ✅ **Nginx underneath**: Battle-tested Nginx engine
- ✅ **SSL management**: Click to enable Let's Encrypt
- ✅ **Access lists**: Built-in basic authentication
- ✅ **Stream support**: TCP/UDP proxying available
- ✅ **Backup/restore**: Easy configuration backup
- ✅ **Good documentation**: Clear guides for common tasks

#### Cons

- ❌ **Docker required**: Runs as a Docker container (with MySQL)
- ❌ **Resource overhead**: Uses more resources (needs database)
- ❌ **Less flexible**: Limited to what UI offers
- ❌ **Update lag**: May lag behind Nginx updates
- ❌ **Dependency**: Adds complexity (container + database)
- ❌ **UI dependency**: If container fails, can't quick-edit configs

#### Configuration

No config files! Everything through web UI:
1. Navigate to http://your-server:81
2. Click "Add Proxy Host"
3. Enter domain, target URL, enable SSL
4. Done!

#### Use Cases

- Perfect for: Users who prefer GUIs over config files
- Best when: You want visual management
- Ideal for: Beginners uncomfortable with text configs

#### Resource Usage

- RAM: ~200-300MB (includes Nginx + MySQL + Node.js)
- CPU: Moderate

### Solution 4: Native Nginx

#### What is Nginx?

Industry-standard web server and reverse proxy. Powers a huge portion of the internet.

#### Pros

- ✅ **Mature and stable**: Battle-tested in production
- ✅ **High performance**: Extremely efficient
- ✅ **Extensive documentation**: Huge community and resources
- ✅ **Flexible**: Can do almost anything with proper config
- ✅ **Low resource usage**: Very lightweight
- ✅ **Widely known**: Easy to find help and examples

#### Cons

- ❌ **Manual SSL**: Need to configure certbot separately
- ❌ **Complex config**: Steeper learning curve than Caddy
- ❌ **Manual cert renewal**: Need to set up cron jobs
- ❌ **Verbose**: More lines of config for same result
- ❌ **No automatic HTTPS**: Everything manual

#### Configuration Example

```nginx
# /etc/nginx/sites-available/coolify.antoinelb.fr
server {
    listen 80;
    server_name coolify.antoinelb.fr;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name coolify.antoinelb.fr;

    ssl_certificate /etc/letsencrypt/live/coolify.antoinelb.fr/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/coolify.antoinelb.fr/privkey.pem;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Plus you need to set up certbot:
```bash
sudo certbot --nginx -d coolify.antoinelb.fr
```

#### Use Cases

- Perfect for: Those familiar with Nginx already
- Best when: You need maximum performance and control
- Ideal for: Production environments with specific needs

#### Resource Usage

- RAM: ~10-20MB
- CPU: Minimal

## Comparison Matrix

| Feature | Caddy | Traefik | NPM | Nginx |
|---------|-------|---------|-----|-------|
| **Setup Time** | 5 min | 20 min | 10 min | 30 min |
| **Config Style** | File (simple) | File (YAML) | Web UI | File (complex) |
| **Auto HTTPS** | ✅ Built-in | ✅ Built-in | ✅ One-click | ❌ Manual |
| **Cert Renewal** | ✅ Automatic | ✅ Automatic | ✅ Automatic | ⚠️ Cron job |
| **Docker Integration** | Good | Excellent | Good | Manual |
| **Resource Usage** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Learning Curve** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Flexibility** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Documentation** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Best For** | Simplicity | Containers | GUI lovers | Performance |

## Recommendation for Your Setup

### Primary Recommendation: Caddy

**Why Caddy is perfect for your use case:**

1. **Automatic HTTPS**: Your two domains will automatically get SSL certificates
2. **Simple config**: 3 lines of config per service
3. **Low maintenance**: Set it and forget it
4. **Cloudflare compatible**: Works great with Cloudflare DNS
5. **Modern**: HTTP/3, security defaults
6. **Resource efficient**: Won't eat into your 32GB RAM
7. **Easy to modify**: When you add services, just add 3 more lines

### Alternative: Traefik

**Consider Traefik if:**
- You plan to use lots of Docker containers (Traefik auto-discovers them)
- You want a dashboard to see all your services
- Coolify already uses Docker heavily (synergy)

### Not Recommended: NPM or Nginx

**Why skip these:**
- **NPM**: Adds unnecessary Docker complexity + database overhead
- **Nginx**: More manual work with certbot, no benefit for your use case

## Implementation Guide: Caddy

### Step 1: Install Caddy on Fedora 43

```bash
# Add Caddy repository
sudo dnf install -y 'dnf-command(copr)'
sudo dnf copr enable @caddy/caddy -y

# Install Caddy
sudo dnf install caddy -y

# Enable and start service
sudo systemctl enable --now caddy
```

### Step 2: Configure Firewall

```bash
# Allow HTTP and HTTPS
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

### Step 3: Configure DNS (Cloudflare)

For each domain you want to use:
1. Go to Cloudflare dashboard
2. Navigate to DNS settings
3. Add A records pointing to your server's public IP:
   - `coolify.antoinelb.fr` → `YOUR_SERVER_IP`
   - `dev.antoinelb.fr` → `YOUR_SERVER_IP`
   - `monitoring.vaultofsuraya.com` → `YOUR_SERVER_IP`
   - etc.

**Important Cloudflare Settings:**
- **Proxy status**: Can be proxied (orange cloud) or DNS only (gray cloud)
  - Orange cloud: Cloudflare proxies traffic (hides your IP, adds DDoS protection)
  - Gray cloud: Direct connection to your server (faster, but IP exposed)
- **SSL/TLS mode**: Set to "Full" or "Full (strict)" (NOT "Flexible")

### Step 4: Create Caddyfile

```bash
sudo nano /etc/caddy/Caddyfile
```

```caddyfile
# Caddy configuration
# Each block automatically gets HTTPS via Let's Encrypt

# Coolify application
coolify.antoinelb.fr {
    reverse_proxy localhost:3000
}

# Remote development environment
dev.antoinelb.fr {
    reverse_proxy localhost:8080
    # Optional: Add basic auth
    # basicauth {
    #     antoine $2a$14$hashed_password_here
    # }
}

# Monitoring dashboard
monitoring.vaultofsuraya.com {
    reverse_proxy localhost:3001
}

# Main domain - could host a landing page
antoinelb.fr, www.antoinelb.fr {
    reverse_proxy localhost:8000
    # Or serve static files:
    # root * /var/www/antoinelb.fr
    # file_server
}

vaultofsuraya.com, www.vaultofsuraya.com {
    reverse_proxy localhost:8001
}
```

### Step 5: Test and Reload

```bash
# Test configuration
sudo caddy validate --config /etc/caddy/Caddyfile

# Reload Caddy (no downtime)
sudo systemctl reload caddy

# Check status
sudo systemctl status caddy

# View logs
sudo journalctl -u caddy -f
```

### Step 6: Verify HTTPS

1. Visit your domains in a browser
2. Check for the lock icon (SSL certificate)
3. Certificate should be issued by "Let's Encrypt"
4. Valid for 90 days (auto-renews at 60 days)

## Caddy Advanced Features

### Adding Basic Authentication

```caddyfile
dev.antoinelb.fr {
    basicauth {
        antoine JDJhJDE0JEhOV...  # Generate with: caddy hash-password
    }
    reverse_proxy localhost:8080
}
```

Generate password hash:
```bash
caddy hash-password --plaintext 'your-password'
```

### Adding Request Logging

```caddyfile
coolify.antoinelb.fr {
    log {
        output file /var/log/caddy/coolify.log
        format json
    }
    reverse_proxy localhost:3000
}
```

### Adding Custom Headers

```caddyfile
dev.antoinelb.fr {
    header {
        # Security headers
        Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
        X-Frame-Options "DENY"
        X-Content-Type-Options "nosniff"
        Referrer-Policy "strict-origin-when-cross-origin"
    }
    reverse_proxy localhost:8080
}
```

### WebSocket Support

Caddy automatically supports WebSockets! No special config needed.

```caddyfile
app.antoinelb.fr {
    reverse_proxy localhost:3000
    # WebSockets work automatically
}
```

### Handling Multiple Backends (Load Balancing)

```caddyfile
app.antoinelb.fr {
    reverse_proxy localhost:3000 localhost:3001 localhost:3002 {
        lb_policy round_robin
        health_path /health
        health_interval 10s
    }
}
```

## Integration with Coolify

Coolify has its own internal proxy system. You have two options:

### Option 1: Caddy as Main Proxy (Recommended)

1. Disable Coolify's proxy feature
2. Use Caddy to route to Coolify services
3. Configure Coolify apps to listen on localhost:PORT
4. Add Caddy entries for each app

**Pros**: Single proxy, consistent config, easier troubleshooting
**Cons**: More manual configuration for new apps

### Option 2: Caddy + Coolify Proxy

1. Caddy handles external domains
2. Coolify proxy handles internal routing
3. Caddy forwards to Coolify proxy

**Pros**: Let Coolify manage its own apps
**Cons**: Two proxy layers (added complexity)

Recommendation: Start with Option 1 for simplicity.

## Cloudflare Integration

### With Cloudflare Proxy (Orange Cloud)

Cloudflare sits between users and your server:
```
User → Cloudflare → Your Server (Caddy) → Service
```

**Benefits**:
- DDoS protection
- Global CDN
- Hides your server IP
- Additional firewall rules

**Configuration**:
- Use SSL/TLS mode "Full (strict)"
- Caddy will get certificates from Let's Encrypt
- Cloudflare verifies your certificate

### Without Cloudflare Proxy (Gray Cloud)

Direct connection:
```
User → Your Server (Caddy) → Service
```

**Benefits**:
- Lower latency (direct connection)
- Simpler troubleshooting
- True end-to-end encryption visibility

**Configuration**:
- Use DNS only mode
- Caddy still handles SSL

## Troubleshooting

### Certificate Issues

```bash
# Check Caddy logs
sudo journalctl -u caddy -n 50

# Common issues:
# 1. Port 80/443 not open
sudo firewall-cmd --list-all

# 2. DNS not pointing to server
dig +short coolify.antoinelb.fr

# 3. Cloudflare SSL mode wrong
# Ensure it's "Full" or "Full (strict)", not "Flexible"
```

### Service Not Accessible

```bash
# Check if backend service is running
sudo netstat -tlnp | grep :3000

# Check Caddy can reach backend
curl -I localhost:3000

# Check Caddy configuration
sudo caddy validate --config /etc/caddy/Caddyfile
```

### Certificate Renewal

Caddy automatically renews certificates. Check renewal:

```bash
# Certificates stored in:
sudo ls -la /var/lib/caddy/.local/share/caddy/certificates/

# Force manual renewal (if needed):
sudo systemctl restart caddy
```

## Next Steps

1. Install and configure Caddy
2. Set up DNS records in Cloudflare
3. Create basic Caddyfile with one test domain
4. Verify HTTPS works
5. Add remaining services as you deploy them
6. Move on to Coolify setup

## Resources

- **Caddy Documentation**: https://caddyserver.com/docs/
- **Caddyfile Reference**: https://caddyserver.com/docs/caddyfile
- **Let's Encrypt**: https://letsencrypt.org/
- **Cloudflare SSL Guide**: https://developers.cloudflare.com/ssl/
