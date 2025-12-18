# Remote Development Environment Solutions

## Overview

A remote development environment allows you to code from anywhere using a web browser, while the actual computing happens on your powerful server.

## Benefits of Remote Development

1. **Access Anywhere**: Code from any device with a browser
2. **Consistent Environment**: Same setup across all devices
3. **Powerful Resources**: Use your server's 32GB RAM and CPU
4. **Persistent Sessions**: Leave processes running when you disconnect
5. **Isolated Projects**: Keep different projects in separate environments
6. **Secure**: Access via VPN, no public exposure needed

## Solution Comparison

### Solution 1: code-server (VS Code in Browser) - Recommended

#### What is code-server?

Open-source VS Code running in your browser. Made by Coder (the company).

#### Pros

- ✅ **Full VS Code**: Exact same experience as desktop VS Code
- ✅ **Extensions**: Install extensions from marketplace
- ✅ **Lightweight**: ~200MB RAM per instance
- ✅ **Easy setup**: Single binary, simple configuration
- ✅ **Multiple instances**: Run different code-servers for different projects
- ✅ **Terminal access**: Built-in terminal
- ✅ **Git integration**: Built-in Git support
- ✅ **Free and open-source**: No licensing costs
- ✅ **Active development**: Regular updates

#### Cons

- ❌ **No user management**: One instance per user (need multiple for team)
- ❌ **Manual setup**: Need to configure each instance
- ❌ **No workspace management**: Can't create/destroy workspaces via UI

#### Use Cases

- Perfect for: Solo developers or small teams
- Best when: You want VS Code experience remotely
- Ideal for: Your use case (personal projects)

#### Resource Usage

- RAM: ~200-500MB per instance
- CPU: Minimal when idle
- Storage: ~500MB for code-server + project files

### Solution 2: Coder (Enterprise Platform)

#### What is Coder?

Enterprise remote development platform. Manages multiple workspaces and users.

#### Pros

- ✅ **Full platform**: Multi-user workspace management
- ✅ **Terraform-based**: Infrastructure as code for workspaces
- ✅ **Multiple IDEs**: VS Code, JetBrains, Jupyter
- ✅ **Workspace templates**: Reusable development environments
- ✅ **SSO integration**: OIDC, SAML support
- ✅ **Metrics and audit logs**: Enterprise features
- ✅ **Auto-shutdown**: Cost-saving features
- ✅ **Git provider integration**: GitHub, GitLab, Bitbucket

#### Cons

- ❌ **More complex**: Requires PostgreSQL, more moving parts
- ❌ **Overkill for solo**: Too much for single user
- ❌ **Learning curve**: Terraform templates to learn
- ❌ **Resource overhead**: Needs more resources to run platform

#### Use Cases

- Perfect for: Teams, organizations
- Best when: You need user/workspace management
- Ideal for: Growing from solo to small team

#### Resource Usage

- RAM: ~500MB for platform + per-workspace resources
- CPU: Moderate
- Storage: Database + workspace volumes

### Solution 3: Gitpod (Self-Hosted)

#### What is Gitpod?

Cloud development environment platform. Can self-host with some limitations.

#### Pros

- ✅ **Git-centric**: Spins up environment from Git repos
- ✅ **Ephemeral workspaces**: Fresh environment each time
- ✅ **Prebuilds**: Faster workspace startup
- ✅ **Browser and desktop**: Access via browser or local VS Code
- ✅ **Collaboration**: Share workspaces with team

#### Cons

- ❌ **Kubernetes required**: Must run on K8s
- ❌ **Complex setup**: Much more complex than code-server
- ❌ **Resource heavy**: Needs substantial resources
- ❌ **Limited self-hosted version**: Some features cloud-only
- ❌ **Overkill**: Way too much for single user

#### Use Cases

- Perfect for: Large teams with K8s experience
- Best when: You want ephemeral, git-based environments
- Not ideal for: Your use case (too complex)

### Solution 4: OpenVSCode Server

#### What is OpenVSCode Server?

Official Microsoft project for VS Code in browser.

#### Pros

- ✅ **Official**: Directly from Microsoft/VS Code team
- ✅ **Lightweight**: Minimal setup
- ✅ **Open source**: MIT license
- ✅ **Compatible**: Works with VS Code extensions

#### Cons

- ❌ **Bare bones**: Very minimal features
- ❌ **No auth built-in**: Need reverse proxy auth
- ❌ **Less features than code-server**: Fewer customizations
- ❌ **Newer project**: Less mature than code-server

#### Use Cases

- Perfect for: Minimalists who want official VS Code
- Best when: You trust Microsoft's direction
- Alternative to: code-server

### Solution 5: JupyterLab (For Data Science)

#### What is JupyterLab?

Interactive notebook environment, great for Python/data science.

#### Pros

- ✅ **Notebook interface**: Great for data exploration
- ✅ **Python-focused**: Excellent Python support
- ✅ **Visualization**: Built-in plotting and charts
- ✅ **Extensions**: Rich ecosystem
- ✅ **Lightweight**: Low resource usage

#### Cons

- ❌ **Not general-purpose**: Best for data science, not web apps
- ❌ **Different paradigm**: Notebook-based, not traditional files
- ❌ **Limited language support**: Primarily Python

#### Use Cases

- Perfect for: Data science, machine learning work
- Best when: Working with notebooks and Python
- Complementary: Use alongside code-server for different workloads

## Comparison Matrix

| Feature | code-server | Coder | Gitpod | OpenVSCode | JupyterLab |
|---------|-------------|-------|---------|-----------|-----------|
| **Setup Time** | 10 min | 30 min | 2+ hours | 10 min | 5 min |
| **Complexity** | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐ |
| **Multi-user** | Manual | ✅ Built-in | ✅ Built-in | Manual | ✅ Built-in |
| **Resource Usage** | Low | Medium | High | Low | Low |
| **VS Code Compat** | ✅ Full | ✅ Full | ✅ Full | ✅ Full | ❌ Different |
| **Auth Built-in** | ✅ Password | ✅ Full | ✅ Full | ❌ Need proxy | ✅ Token |
| **Best For** | Solo dev | Teams | Large teams | Minimalists | Data science |
| **Deployment** | Binary | Docker | Kubernetes | Docker/Binary | Docker/pip |

## Recommendation for Your Setup

### Primary Recommendation: code-server

**Why code-server is perfect for you:**

1. **Familiar**: It's VS Code, which you likely already know
2. **Simple**: Single Docker container or binary
3. **Lightweight**: Won't eat into your 32GB RAM
4. **Can deploy via Coolify**: Fits into your existing infrastructure
5. **Multiple instances**: Run different instances for different projects
6. **VPN-friendly**: Access securely via Tailscale

### Architecture

```
You (anywhere) → VPN (Tailscale) → Caddy → code-server(s) → Project files
```

### Scaling Path

- **Now**: Single code-server instance
- **Later**: Multiple instances for different projects
- **Future**: Upgrade to Coder if you expand to a team

## Implementation Guide: code-server

### Method 1: Deploy with Coolify (Recommended)

#### Step 1: Create code-server in Coolify

1. **Go to Coolify**: Projects → New Resource → Docker Compose
2. **Use this docker-compose.yml**:

```yaml
version: '3.8'

services:
  code-server:
    image: codercom/code-server:latest
    container_name: code-server
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Paris
      - PASSWORD=${CODE_SERVER_PASSWORD}  # Set in Coolify env
      - SUDO_PASSWORD=${SUDO_PASSWORD}    # Optional
    volumes:
      - /home/rainreport/code-projects:/home/coder/projects
      - /home/rainreport/code-config:/home/coder/.local/share/code-server
    ports:
      - "8080:8080"
    restart: unless-stopped
```

3. **Environment Variables** (in Coolify):
   - `CODE_SERVER_PASSWORD`: Your secure password
   - `SUDO_PASSWORD`: Optional, for sudo inside container

4. **Deploy**: Click Deploy and wait for completion

#### Step 2: Configure Caddy

Add to `/etc/caddy/Caddyfile`:

```caddyfile
code.antoinelb.fr {
    # Only accessible via VPN (recommended)
    @notlocal {
        not remote_ip 100.64.0.0/10  # Tailscale IP range
    }
    respond @notlocal 403

    reverse_proxy localhost:8080
}
```

Reload Caddy:
```bash
sudo systemctl reload caddy
```

#### Step 3: Access code-server

1. Connect to VPN (Tailscale)
2. Navigate to: `https://code.antoinelb.fr`
3. Enter your password
4. Start coding!

### Method 2: Manual Installation (Alternative)

#### Step 1: Install code-server Binary

```bash
# Download and install
curl -fsSL https://code-server.dev/install.sh | sh

# This installs to /usr/bin/code-server
```

#### Step 2: Configure code-server

Create config file:

```bash
mkdir -p ~/.config/code-server
nano ~/.config/code-server/config.yaml
```

```yaml
bind-addr: 127.0.0.1:8080
auth: password
password: your-secure-password-here
cert: false  # Caddy handles SSL
```

#### Step 3: Create Systemd Service

```bash
sudo nano /etc/systemd/system/code-server.service
```

```ini
[Unit]
Description=code-server
After=network.target

[Service]
Type=simple
User=rainreport
WorkingDirectory=/home/rainreport
Environment="PASSWORD=your-secure-password"
ExecStart=/usr/bin/code-server --bind-addr 127.0.0.1:8080 /home/rainreport/projects
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

#### Step 4: Enable and Start

```bash
sudo systemctl enable --now code-server
sudo systemctl status code-server
```

#### Step 5: Configure Caddy (Same as Method 1)

### Method 3: Multiple code-server Instances

For different projects with isolated environments:

```bash
# Project 1: Web Development
docker run -d \
  --name code-server-web \
  -p 8081:8080 \
  -e PASSWORD=secure-password-1 \
  -v ~/projects/web:/home/coder/project \
  -v ~/code-config-web:/home/coder/.local/share/code-server \
  codercom/code-server:latest

# Project 2: Python/ML
docker run -d \
  --name code-server-ml \
  -p 8082:8080 \
  -e PASSWORD=secure-password-2 \
  -v ~/projects/ml:/home/coder/project \
  -v ~/code-config-ml:/home/coder/.local/share/code-server \
  codercom/code-server:latest
```

Caddy config:
```caddyfile
web-dev.antoinelb.fr {
    @notlocal {
        not remote_ip 100.64.0.0/10
    }
    respond @notlocal 403
    reverse_proxy localhost:8081
}

ml-dev.antoinelb.fr {
    @notlocal {
        not remote_ip 100.64.0.0/10
    }
    respond @notlocal 403
    reverse_proxy localhost:8082
}
```

## Advanced Configuration

### Installing Extensions

Extensions work just like desktop VS Code:

1. Click Extensions icon in sidebar
2. Search for extension
3. Click Install

**Or via command line**:
```bash
code-server --install-extension ms-python.python
code-server --install-extension golang.go
```

### Syncing Settings

To sync settings from your desktop VS Code:

1. Export settings from desktop VS Code
2. In code-server, go to File → Preferences → Settings
3. Open settings.json and paste your settings

**Or use Settings Sync**:
- Enable Settings Sync in code-server
- Sign in with GitHub
- Sync automatically across instances

### Custom Docker Image

For specific language environments:

```dockerfile
FROM codercom/code-server:latest

USER root

# Install Node.js
RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
RUN apt-get install -y nodejs

# Install Python
RUN apt-get install -y python3 python3-pip

# Install Go
RUN wget https://go.dev/dl/go1.21.0.linux-amd64.tar.gz
RUN tar -C /usr/local -xzf go1.21.0.linux-amd64.tar.gz
ENV PATH=$PATH:/usr/local/go/bin

# Install Docker CLI (for managing server's Docker)
RUN apt-get install -y docker.io

USER coder

# Pre-install extensions
RUN code-server --install-extension ms-python.python
RUN code-server --install-extension golang.go
RUN code-server --install-extension dbaeumer.vscode-eslint
```

Build and use:
```bash
docker build -t custom-code-server .
docker run -d -p 8080:8080 -e PASSWORD=pass custom-code-server
```

### Git Configuration

Inside code-server:

```bash
# Configure Git
git config --global user.name "Your Name"
git config --global user.email "you@antoinelb.fr"

# Set up SSH keys for GitHub
ssh-keygen -t ed25519 -C "you@antoinelb.fr"
cat ~/.ssh/id_ed25519.pub
# Add to GitHub: Settings → SSH Keys

# Or use HTTPS with credential helper
git config --global credential.helper store
```

### Accessing Server Resources

code-server runs on your server, so it can:

1. **Access local files**: Mount directories from your server
2. **Use server's Docker**: Install Docker CLI in code-server
3. **Access databases**: Connect to PostgreSQL/MySQL on localhost
4. **Run servers**: Start web servers that are accessible via your VPN

Example: Developing a Node.js app that connects to your Coolify PostgreSQL:

```javascript
// Inside code-server
const { Pool } = require('pg');
const pool = new Pool({
  host: 'localhost',
  port: 5432,
  database: 'mydb',
  user: 'user',
  password: 'pass'
});

// Works because code-server is on same machine as database!
```

## Integration with Your Infrastructure

### With VPN (Tailscale)

1. code-server only accessible via VPN
2. No public exposure
3. Fast, secure connection from anywhere

### With Coolify

1. Deploy code-server via Coolify
2. Coolify manages the container lifecycle
3. Automatic restarts if code-server crashes
4. Use Coolify's backup for code-server config

### With Git

1. Clone repos directly in code-server
2. Push/pull from GitHub/GitLab
3. Use built-in Git features (staging, commits, etc.)

### With Databases

1. Connect to databases running on same server
2. Use localhost connections
3. No need to expose database ports

## Security Best Practices

### 1. Strong Authentication

```yaml
# Use strong password
environment:
  - PASSWORD=$(openssl rand -base64 32)
```

### 2. VPN-Only Access

```caddyfile
code.antoinelb.fr {
    @notlocal {
        not remote_ip 100.64.0.0/10
    }
    respond @notlocal 403
    reverse_proxy localhost:8080
}
```

### 3. Disable Telemetry

```yaml
environment:
  - TELEMETRY=false
```

### 4. Regular Updates

```bash
# Update code-server
docker pull codercom/code-server:latest
docker restart code-server

# Or with Coolify, redeploy to get latest image
```

### 5. Limit Sudo Access

Don't set `SUDO_PASSWORD` unless needed. If you need sudo:

```yaml
environment:
  - SUDO_PASSWORD=different-strong-password
```

### 6. File Permissions

```bash
# Ensure project files have correct permissions
sudo chown -R 1000:1000 /home/rainreport/code-projects
sudo chmod -R 755 /home/rainreport/code-projects
```

## Troubleshooting

### Can't Access code-server

```bash
# Check if container is running
docker ps | grep code-server

# Check logs
docker logs code-server

# Verify port is listening
sudo netstat -tlnp | grep 8080

# Test locally
curl -I http://localhost:8080
```

### Extensions Won't Install

1. Check internet connection from container:
   ```bash
   docker exec -it code-server curl -I https://marketplace.visualstudio.com
   ```

2. Try manual installation:
   ```bash
   docker exec -it code-server code-server --install-extension <extension-id>
   ```

### Slow Performance

1. **Check resources**:
   ```bash
   docker stats code-server
   ```

2. **Increase memory limit**:
   ```yaml
   services:
     code-server:
       mem_limit: 2g
       mem_reservation: 1g
   ```

3. **Check network latency**: Use faster VPN or reduce VPN overhead

### Git Authentication Issues

```bash
# SSH key permissions
docker exec -it code-server chmod 600 ~/.ssh/id_ed25519

# Or use HTTPS with token
git config --global credential.helper store
# Then enter GitHub personal access token when prompted
```

### Terminal Not Working

1. Check that container has shell:
   ```bash
   docker exec -it code-server bash
   ```

2. Restart code-server container:
   ```bash
   docker restart code-server
   ```

## Performance Optimization

### 1. Use SSD Storage

Ensure code-server volumes are on SSD (you have 2TB - check if SSD):
```bash
lsblk -d -o name,rota
# 0 = SSD, 1 = HDD
```

### 2. Increase Container Resources

```yaml
services:
  code-server:
    mem_limit: 4g
    cpus: '2.0'
```

### 3. Disable Unnecessary Extensions

Remove extensions you don't use to reduce memory usage.

### 4. Use Workspace Folders

Instead of opening entire home directory, open specific project folders:
- File → Open Folder → `/home/coder/projects/specific-project`

## Alternative: Desktop VS Code + Remote SSH

If you prefer desktop VS Code over browser:

### Setup

1. **Install Remote-SSH extension** in desktop VS Code
2. **Configure SSH connection**:
   ```bash
   # In VS Code: CMD+Shift+P → "Remote-SSH: Connect to Host"
   # Add: ssh rainreport@server.tailscale-network.ts.net
   ```

3. **Connect and code** as if local!

### Pros
- Native VS Code performance
- Better keybindings
- Native clipboard integration

### Cons
- Requires VS Code installed locally
- Not accessible from random devices
- More bandwidth usage

### When to Use

- **code-server**: For coding from any device, including tablets
- **Remote-SSH**: For regular development from your main machines

You can use both! Remote-SSH from laptop, code-server from tablet/phone.

## Recommended Setup for You

Based on your infrastructure:

```
1. Deploy code-server via Coolify (easiest management)
2. Use Docker Compose with custom image (includes your tools)
3. Accessible only via Tailscale VPN
4. Caddy provides HTTPS and VPN restriction
5. Mount your projects directory
6. Install your favorite extensions
7. Optionally: Remote-SSH from main laptop
```

This gives you:
- ✅ Code from anywhere (browser)
- ✅ Code from laptop (Remote-SSH)
- ✅ Secure (VPN-only)
- ✅ Integrated with your infrastructure
- ✅ Easy to manage (Coolify)

## Example: Complete Setup

### docker-compose.yml for Coolify

```yaml
version: '3.8'

services:
  code-server:
    image: codercom/code-server:latest
    container_name: code-server-main
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Paris
      - PASSWORD=${CODE_SERVER_PASSWORD}
      - TELEMETRY=false
    volumes:
      # Projects
      - /home/rainreport/code-projects:/home/coder/projects
      # Config persistence
      - /home/rainreport/code-config:/home/coder/.local/share/code-server
      # Docker socket (to manage containers from code-server)
      - /var/run/docker.sock:/var/run/docker.sock
    ports:
      - "8080:8080"
    restart: unless-stopped
    mem_limit: 2g
    cpus: '2.0'
```

### Caddy Configuration

```caddyfile
code.antoinelb.fr {
    @notlocal {
        not remote_ip 100.64.0.0/10
    }
    respond @notlocal 403

    reverse_proxy localhost:8080 {
        # WebSocket support (important for code-server)
        header_up Upgrade {http.request.header.Upgrade}
        header_up Connection {http.request.header.Connection}
    }
}
```

### First-Time Setup

```bash
# 1. Create directories
mkdir -p /home/rainreport/code-projects
mkdir -p /home/rainreport/code-config

# 2. Set permissions
sudo chown -R 1000:1000 /home/rainreport/code-projects /home/rainreport/code-config

# 3. Deploy via Coolify with above docker-compose.yml

# 4. Add Caddy config and reload

# 5. Connect via VPN and access https://code.antoinelb.fr
```

## Next Steps

1. Choose deployment method (Coolify recommended)
2. Deploy code-server
3. Configure Caddy
4. Test access via VPN
5. Install your favorite extensions
6. Clone your first project
7. Start coding!

## Resources

- **code-server**: https://github.com/coder/code-server
- **Coder**: https://coder.com
- **VS Code**: https://code.visualstudio.com/docs/remote/remote-overview
- **JupyterLab**: https://jupyterlab.readthedocs.io/
