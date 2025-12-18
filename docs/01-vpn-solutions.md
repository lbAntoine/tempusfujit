# VPN Solutions: Tailscale vs Headscale

## Overview

A VPN (Virtual Private Network) solution will allow you to securely access your server and services from anywhere, as well as create a private network between multiple devices.

## Understanding Your Needs

Based on your use case (both access and networking), you need:

1. **Secure Remote Access**: Connect to your server from anywhere without exposing services to the public internet
2. **Service Networking**: Allow different services to communicate securely
3. **Multi-Device**: Connect your laptop, phone, and other devices to access your infrastructure
4. **Exit Node** (optional): Route your internet traffic through your server when away from home

## Solution 1: Tailscale (Recommended for Beginners)

### What is Tailscale?

Tailscale is a managed WireGuard VPN service. It handles all the complexity of creating a secure mesh network between your devices.

### Pros

- ✅ **Zero configuration**: Works out of the box in 5 minutes
- ✅ **Free tier**: Up to 100 devices for personal use
- ✅ **Magic DNS**: Access devices by name (e.g., `server.tailscale-network.ts.net`)
- ✅ **Mobile apps**: Native iOS and Android apps
- ✅ **Excellent documentation**: Comprehensive guides and support
- ✅ **ACLs**: Fine-grained access control through web UI
- ✅ **MagicDNS**: Automatic DNS resolution for your devices
- ✅ **Exit Nodes**: Easy to configure
- ✅ **Subnet routing**: Expose entire subnets through your server
- ✅ **SSH**: Built-in SSH access without exposing port 22
- ✅ **Regular updates**: Maintained by a well-funded company

### Cons

- ❌ **Third-party dependency**: Tailscale controls the coordination server
- ❌ **Privacy considerations**: Metadata (not traffic) goes through Tailscale servers
- ❌ **Account required**: Need to create a Tailscale account
- ❌ **Free tier limits**: 1 user, 100 devices (sufficient for most personal use)

### Use Cases

- Perfect for: Personal projects, small teams, getting started quickly
- Best when: You trust Tailscale with coordination metadata
- Ideal for: Beginners who want something that "just works"

### Cost

- **Free**: 1 user, 100 devices, 3 users who can share devices
- **Personal Pro**: $6/month - 100 devices, unlimited users
- **Team**: $18/user/month - More features

## Solution 2: Headscale (Self-Hosted Alternative)

### What is Headscale?

Headscale is an open-source, self-hosted implementation of the Tailscale control server. You get all the benefits of Tailscale but run your own coordination server.

### Pros

- ✅ **Full control**: You own and control everything
- ✅ **No external dependencies**: Completely self-hosted
- ✅ **Privacy**: All metadata stays on your server
- ✅ **No costs**: 100% free and open source
- ✅ **Compatible with Tailscale clients**: Use official Tailscale apps
- ✅ **No account limits**: Unlimited users and devices
- ✅ **API available**: Programmatic management

### Cons

- ❌ **More complex setup**: Requires more technical knowledge
- ❌ **Self-maintenance**: You're responsible for updates and security
- ❌ **No web UI by default**: Command-line based (third-party UIs available)
- ❌ **Feature lag**: May not have all latest Tailscale features
- ❌ **Less documentation**: Smaller community compared to Tailscale
- ❌ **Coordination server needs**: Requires another service to maintain
- ❌ **Manual ACL management**: Edit configuration files instead of web UI

### Use Cases

- Perfect for: Privacy-conscious users, learning experience
- Best when: You want zero external dependencies
- Ideal for: Those comfortable with server administration

### Requirements

- A server to run Headscale (can be your main server)
- Domain name or public IP
- Basic understanding of DNS and networking

## Comparison Matrix

| Feature | Tailscale | Headscale |
|---------|-----------|-----------|
| Setup Time | 5 minutes | 30-60 minutes |
| Maintenance | None | Regular updates needed |
| Privacy | Metadata to Tailscale | 100% self-hosted |
| Mobile Apps | Official apps | Use Tailscale apps |
| Web UI | Yes | Third-party options |
| Documentation | Excellent | Good |
| Cost | Free tier / Paid | Free |
| Ease of Use | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| Control | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Features | All features | Most features |

## Recommendation

### For Your Use Case

Since you're learning and want both access and networking capabilities, I recommend:

**Start with Tailscale, migrate to Headscale later if needed**

**Why this approach?**

1. **Immediate functionality**: Get your VPN working in minutes, unblock other infrastructure work
2. **Learn the concepts**: Understand how mesh VPNs work without configuration complexity
3. **Free tier is sufficient**: 100 devices is more than enough for personal use
4. **Easy migration path**: If you later want full control, you can migrate to Headscale
5. **Stable foundation**: Build other services (Coolify, monitoring) on a working VPN first

### When to Consider Headscale

Migrate to Headscale when:
- You're comfortable with your server setup and want full control
- You have specific privacy requirements
- You want a learning project
- You've outgrown Tailscale's free tier

## Implementation Guides

### Quick Start: Tailscale

1. **Install on Fedora server**:
   ```bash
   sudo dnf config-manager --add-repo https://pkgs.tailscale.com/stable/fedora/tailscale.repo
   sudo dnf install tailscale
   sudo systemctl enable --now tailscaled
   ```

2. **Authenticate**:
   ```bash
   sudo tailscale up
   ```
   Follow the URL to authenticate with your Tailscale account.

3. **Enable features**:
   ```bash
   # Enable subnet routing (to access other services on your server)
   sudo tailscale up --advertise-routes=192.168.1.0/24 --accept-routes

   # Enable as exit node (route internet traffic through server)
   sudo tailscale up --advertise-exit-node
   ```

4. **Install on other devices**:
   - Download Tailscale app for your OS
   - Login with same account
   - Devices automatically see each other

### Advanced Setup: Headscale

See dedicated setup guide in this document below.

## Integration with Your Infrastructure

### Using VPN with Coolify

Once VPN is set up:
1. Coolify admin interface is only accessible via VPN (not exposed to internet)
2. SSH to server only through VPN (disable public SSH)
3. Database ports only accessible via VPN

### Using VPN with Remote Coding

Your remote development environment:
1. Runs on your server
2. Accessible only via VPN
3. Can access all other services through VPN mesh

### Security Benefits

1. **No public ports**: Only VPN port exposed (WireGuard uses UDP 41641)
2. **Encrypted traffic**: All communication encrypted with WireGuard
3. **Device authentication**: Only authorized devices can connect
4. **No passwords**: Key-based authentication
5. **Reduced attack surface**: Services hidden from public internet

## Detailed Headscale Setup Guide

*Follow this only if you choose Headscale instead of Tailscale*

### Prerequisites

- Server with public IP or domain
- Docker (or can install natively)

### Step 1: Install Headscale

```bash
# Using Docker (recommended)
sudo mkdir -p /opt/headscale/config
sudo mkdir -p /opt/headscale/data

# Download Headscale
cd /opt/headscale
sudo wget https://github.com/juanfont/headscale/releases/latest/download/headscale_*_linux_amd64 -O headscale
sudo chmod +x headscale

# Generate configuration
sudo ./headscale config generate > config/config.yaml
```

### Step 2: Configure Headscale

Edit `/opt/headscale/config/config.yaml`:

```yaml
server_url: https://headscale.antoinelb.fr:443
listen_addr: 0.0.0.0:8080
metrics_listen_addr: 127.0.0.1:9090

# Database
db_type: sqlite3
db_path: /var/lib/headscale/db.sqlite

# DNS
magic_dns: true
base_domain: antoinelb.fr
dns_config:
  nameservers:
    - 1.1.1.1
```

### Step 3: Create Systemd Service

```bash
sudo nano /etc/systemd/system/headscale.service
```

```ini
[Unit]
Description=Headscale Control Server
After=network.target

[Service]
Type=simple
User=headscale
Group=headscale
ExecStart=/opt/headscale/headscale serve
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### Step 4: Setup Reverse Proxy

Your reverse proxy (Caddy/Traefik) should forward:
- `headscale.antoinelb.fr` → `localhost:8080`
- Handle SSL certificates

### Step 5: Create Users and Register Devices

```bash
# Create a user
sudo headscale users create antoine

# On your client device, install Tailscale client
# Then connect to your Headscale server:
tailscale up --login-server https://headscale.antoinelb.fr

# Back on server, register the device
sudo headscale nodes register --user antoine --key <KEY_FROM_CLIENT>
```

### Step 6: Verify Connection

```bash
# List nodes
sudo headscale nodes list

# Check routes
sudo headscale routes list
```

## Next Steps

1. Choose your VPN solution (Tailscale recommended to start)
2. Install and configure VPN
3. Test connectivity from another device
4. Move on to setting up reverse proxy
5. Configure Coolify to be accessible only via VPN

## Resources

### Tailscale
- Official Website: https://tailscale.com
- Documentation: https://tailscale.com/kb
- Pricing: https://tailscale.com/pricing

### Headscale
- GitHub: https://github.com/juanfont/headscale
- Documentation: https://headscale.net
- Third-party Web UI: https://github.com/gurucomputing/headscale-ui

### WireGuard
- Official Website: https://wireguard.com
- Whitepaper: https://wireguard.com/papers/wireguard.pdf
