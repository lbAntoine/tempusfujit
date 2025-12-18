# System Architecture Overview

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        External Access                          │
│                                                                 │
│  User's Devices (Laptop, Phone, Tablet)                       │
│                         │                                       │
│                         ↓                                       │
│              ┌──────────────────────┐                          │
│              │   Tailscale VPN      │                          │
│              │  (100.64.0.0/10)     │                          │
│              └──────────────────────┘                          │
└─────────────────────────┬───────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Your Server (Fedora 43)                      │
│                  antoinelb.fr / vaultofsuraya.com               │
│                   32GB RAM | 2TB Storage                        │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                    Firewall Layer                          │ │
│  │  - firewalld                                               │ │
│  │  - Only VPN, HTTP, HTTPS exposed                           │ │
│  │  - Fail2Ban for intrusion prevention                       │ │
│  └────────────────────────┬──────────────────────────────────┘ │
│                           │                                      │
│                           ↓                                      │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                   Reverse Proxy (Caddy)                    │ │
│  │  - Automatic HTTPS (Let's Encrypt)                         │ │
│  │  - Domain routing                                          │ │
│  │  - Security headers                                        │ │
│  │  - VPN access control                                      │ │
│  └────────────┬──────────────────────────────────────────────┘ │
│               │                                                  │
│               ↓                                                  │
│  ┌────────────────────────────────────────────────────────────┐│
│  │                    Applications Layer                       ││
│  │                                                             ││
│  │  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐ ││
│  │  │   Coolify    │  │ code-server  │  │   Monitoring    │ ││
│  │  │              │  │              │  │  - Prometheus   │ ││
│  │  │ :8000        │  │ :8080        │  │  - Grafana      │ ││
│  │  └──────┬───────┘  └──────────────┘  │  - Alertmanager │ ││
│  │         │                             └─────────────────┘ ││
│  │         ↓                                                  ││
│  │  ┌────────────────────────────────────────────────────┐  ││
│  │  │          Coolify-Managed Containers                 │  ││
│  │  │                                                     │  ││
│  │  │  ┌──────────┐ ┌──────────┐ ┌───────────────────┐  │  ││
│  │  │  │ Web App  │ │ Web App  │ │    Databases      │  │  ││
│  │  │  │ (Node.js)│ │ (Python) │ │ - PostgreSQL      │  │  ││
│  │  │  │ :3000    │ │ :3001    │ │ - MySQL           │  │  ││
│  │  │  └──────────┘ └──────────┘ │ - MongoDB         │  │  ││
│  │  │                             │ - Redis           │  │  ││
│  │  │                             └───────────────────┘  │  ││
│  │  └────────────────────────────────────────────────────┘  ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                   Storage & Backups                        │ │
│  │                                                             │ │
│  │  Local Storage:                                            │ │
│  │  - /home/rainreport/code-projects (Development)            │ │
│  │  - /var/lib/docker/volumes (Container data)                │ │
│  │  - /backup (Local backup staging)                          │ │
│  │                                                             │ │
│  │  Remote Backup: Backblaze B2 / S3                          │ │
│  │  - Restic (encrypted, deduplicated)                        │ │
│  └───────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

## Network Flow

### External Access Flow

```
User → Tailscale VPN → Server's Tailscale IP → Caddy → Service
```

### Domain Routing

```
coolify.antoinelb.fr        → Caddy → localhost:8000 (Coolify UI)
code.antoinelb.fr           → Caddy → localhost:8080 (code-server)
monitoring.antoinelb.fr     → Caddy → localhost:3001 (Grafana)
app1.antoinelb.fr           → Caddy → localhost:3000 (Your App 1)
app2.vaultofsuraya.com      → Caddy → localhost:3001 (Your App 2)
```

### VPN-Protected Access

All admin interfaces only accessible via VPN:
- Coolify dashboard
- Grafana monitoring
- code-server
- SSH access
- Database ports

Public access (optional):
- Your deployed applications (if they're meant to be public)
- Static websites

## Component Details

### Layer 1: Security & Access

**Purpose**: Control who can access the server

**Components**:
- **Tailscale VPN**: Secure mesh network for remote access
- **firewalld**: Firewall with restrictive default rules
- **Fail2Ban**: Intrusion prevention
- **SELinux**: Mandatory access control

**Configuration**:
- SSH only via VPN
- Admin services only via VPN
- Public HTTP/HTTPS for Caddy
- Tailscale UDP port for VPN

### Layer 2: Reverse Proxy

**Purpose**: Route traffic to correct services, handle SSL

**Component**: Caddy

**Responsibilities**:
- Automatic HTTPS via Let's Encrypt
- Domain-based routing
- SSL termination
- Security headers
- VPN access enforcement
- Rate limiting

**Example Configuration**:
```caddyfile
coolify.antoinelb.fr {
    @notlocal {
        not remote_ip 100.64.0.0/10  # Tailscale only
    }
    respond @notlocal 403
    reverse_proxy localhost:8000
}
```

### Layer 3: Orchestration

**Purpose**: Manage application deployments

**Component**: Coolify

**Features**:
- Git-based deployments
- Database management
- Environment variables
- Automatic deployments via webhooks
- Built-in backups

**Manages**:
- Your web applications
- Databases (PostgreSQL, MySQL, etc.)
- Static sites
- Docker Compose stacks

### Layer 4: Applications

**Your Applications**:
- Node.js applications
- Python applications
- Static sites
- APIs and microservices

**System Applications**:
- code-server (remote development)
- Monitoring stack (Prometheus, Grafana)
- Uptime Kuma (status monitoring)

### Layer 5: Data

**Databases** (managed by Coolify):
- PostgreSQL
- MySQL/MariaDB
- MongoDB
- Redis

**Storage**:
- Docker volumes (persistent container data)
- Project files (code-server)
- Configuration files

**Backups**:
- Local: `/backup` directory
- Remote: Backblaze B2 via Restic
- Frequency: Daily with 7-day retention

### Layer 6: Monitoring

**Purpose**: Observe system health, detect issues

**Components**:
- **Prometheus**: Metrics collection
- **Grafana**: Visualization dashboards
- **Node Exporter**: System metrics
- **cAdvisor**: Container metrics
- **Alertmanager**: Alert notifications
- **Uptime Kuma**: Uptime monitoring

**What's Monitored**:
- CPU, memory, disk usage
- Container resource usage
- Service uptime
- Application metrics
- Database performance

## Data Flow Examples

### Example 1: Deploying a New Application

```
1. Push code to GitHub
   ↓
2. GitHub webhook triggers Coolify
   ↓
3. Coolify pulls code, builds container
   ↓
4. Container starts on localhost:PORT
   ↓
5. Add Caddy configuration: app.antoinelb.fr → localhost:PORT
   ↓
6. Caddy gets Let's Encrypt certificate
   ↓
7. Application accessible at https://app.antoinelb.fr
```

### Example 2: Remote Development Session

```
1. Connect to Tailscale VPN from laptop
   ↓
2. Access https://code.antoinelb.fr (via VPN)
   ↓
3. Caddy verifies VPN IP, forwards to code-server
   ↓
4. code-server authenticates with password
   ↓
5. VS Code interface in browser
   ↓
6. Code runs on server (32GB RAM, fast CPU)
   ↓
7. Can access databases on localhost
   ↓
8. Push code to GitHub when done
   ↓
9. Coolify auto-deploys to production
```

### Example 3: Monitoring Alert

```
1. Prometheus scrapes metrics (every 15s)
   ↓
2. CPU usage > 80% for 5 minutes
   ↓
3. Prometheus evaluates alert rule
   ↓
4. Alert fires, sent to Alertmanager
   ↓
5. Alertmanager sends notification
   ↓
6. Email/Discord/Slack notification received
   ↓
7. Check Grafana dashboard via VPN
   ↓
8. Investigate and resolve issue
```

### Example 4: Disaster Recovery

```
1. Server failure detected
   ↓
2. Provision new server (same specs)
   ↓
3. Install base OS (Fedora 43)
   ↓
4. Clone this documentation repo
   ↓
5. Follow setup guides in order:
   - Security hardening
   - VPN setup
   - Reverse proxy (Caddy)
   - Monitoring
   ↓
6. Restore backups from Backblaze B2:
   restic restore latest --target /
   ↓
7. Install Coolify
   ↓
8. Restore Coolify database
   ↓
9. Redeploy applications via Coolify
   ↓
10. Verify all services operational
```

## Port Mapping

### External Ports (Exposed to Internet)

| Port | Protocol | Service | Purpose |
|------|----------|---------|---------|
| 80 | TCP | Caddy | HTTP (redirects to HTTPS) |
| 443 | TCP | Caddy | HTTPS (all web traffic) |
| 41641 | UDP | Tailscale | VPN connection |

### Internal Ports (Localhost only)

| Port | Service | Purpose |
|------|---------|---------|
| 22 | SSH | Remote administration (VPN only) |
| 8000 | Coolify | Deployment platform UI |
| 8080 | code-server | Remote development environment |
| 3001 | Grafana | Monitoring dashboards |
| 9090 | Prometheus | Metrics database |
| 9093 | Alertmanager | Alert routing |
| 9100 | Node Exporter | System metrics |
| 8082 | cAdvisor | Container metrics |
| 3002 | Uptime Kuma | Uptime monitoring |
| 3000+ | Applications | Your deployed apps (via Coolify) |
| 5432 | PostgreSQL | Database (if running) |
| 3306 | MySQL | Database (if running) |
| 27017 | MongoDB | Database (if running) |
| 6379 | Redis | Cache/database (if running) |

## Security Layers

### Layer 1: Network Security
- Firewall allows only necessary ports
- VPN required for admin access
- Fail2Ban blocks brute force attempts

### Layer 2: Transport Security
- All traffic encrypted (HTTPS via Let's Encrypt)
- VPN uses WireGuard encryption
- SSH uses key-based authentication only

### Layer 3: Application Security
- Caddy adds security headers
- Services run with minimal privileges
- Docker containers isolated
- SELinux enforces mandatory access control

### Layer 4: Data Security
- Databases not exposed publicly
- Backups encrypted (Restic)
- Secrets not stored in code
- Regular security updates

### Layer 5: Monitoring & Response
- Real-time alerts for anomalies
- Audit logs for all admin actions
- Regular security scans (Lynis, RKHunter)
- Incident response plan

## Scalability Considerations

### Current Capacity

With 32GB RAM and 2TB storage, you can run:
- 10-20 lightweight applications
- 5-10 medium applications (with databases)
- Multiple databases
- Full monitoring stack
- Remote development environment

**Rough estimates**:
- Coolify: 500MB RAM
- Caddy: 50MB RAM
- Monitoring stack: 1GB RAM
- code-server: 500MB RAM
- Each small app: 200-500MB RAM
- Each database: 200MB-1GB RAM

**Available for apps**: ~25-28GB RAM

### Vertical Scaling (Same Server)

If you need more resources:
- Upgrade RAM (already high at 32GB)
- Add more storage (you have 2TB)
- Optimize applications (caching, CDN)

### Horizontal Scaling (Multiple Servers)

When one server isn't enough:

1. **Add Application Servers**:
   - Deploy apps on multiple servers
   - Use Caddy load balancing
   - Shared database on main server

2. **Add Database Server**:
   - Dedicated server for databases
   - Connect via Tailscale VPN
   - Replicas for high availability

3. **Add Monitoring Server**:
   - Separate monitoring stack
   - Monitor all servers from one place

**With Coolify**:
- Add servers to Coolify
- Deploy apps across servers
- Manage from single interface

## Backup & Disaster Recovery

### What's Backed Up

**Critical (Daily backups)**:
- Application databases
- Docker volumes
- Configuration files
- SSL certificates
- Code-server projects

**Semi-critical (Weekly backups)**:
- Coolify database
- Monitoring configuration
- System configurations

**Not backed up** (can be recreated):
- Docker images
- System packages
- Cached data

### Backup Strategy

```
Daily (3 AM) → Restic → Backblaze B2
             ↓
        Local copy (/backup)
             ↓
        Retention:
        - Daily: 7 days
        - Weekly: 4 weeks
        - Monthly: 6 months
```

### Recovery Time Objectives (RTO)

- **Minor issue** (single service): 5-15 minutes
- **Major issue** (server crash): 1-2 hours
- **Disaster** (complete rebuild): 4-6 hours

### Recovery Point Objectives (RPO)

- **Databases**: < 24 hours (daily backups)
- **Files**: < 24 hours (daily backups)
- **Code**: 0 (in Git)

## Maintenance Windows

### Automated Maintenance

**Daily (3 AM)**:
- Security updates applied
- Backups run
- Log rotation

**Weekly (Sunday 2 AM)**:
- Docker image cleanup
- Database optimization
- Full backup integrity check

### Manual Maintenance

**Weekly**:
- Review monitoring dashboards
- Check backup status
- Review security alerts

**Monthly**:
- Update Coolify
- Update monitoring stack
- Security audit (Lynis)
- Test backup restoration

**Quarterly**:
- Rotate secrets
- Review and update documentation
- Capacity planning

## Technology Stack Summary

| Layer | Technology | Why? |
|-------|------------|------|
| **OS** | Fedora 43 Server | Modern, secure, well-supported |
| **VPN** | Tailscale | Zero-config, secure mesh network |
| **Firewall** | firewalld | Built-in, flexible |
| **Reverse Proxy** | Caddy | Automatic HTTPS, simple config |
| **Orchestration** | Coolify | Self-hosted PaaS, easy deployments |
| **Containerization** | Docker | Industry standard |
| **Monitoring** | Prometheus + Grafana | Best-in-class monitoring |
| **Backups** | Restic | Modern, encrypted, deduplicated |
| **Remote Dev** | code-server | VS Code in browser |
| **Security** | SELinux, Fail2Ban | Defense in depth |

## Implementation Order

Follow this order for smooth setup:

1. **Security Hardening** (Day 1)
   - Update system
   - Configure firewall
   - Harden SSH
   - Enable SELinux

2. **VPN Setup** (Day 1)
   - Install Tailscale
   - Connect devices
   - Test connectivity

3. **Reverse Proxy** (Day 1-2)
   - Install Caddy
   - Configure DNS in Cloudflare
   - Test HTTPS

4. **Monitoring** (Day 2)
   - Deploy Prometheus + Grafana
   - Configure exporters
   - Set up alerts

5. **Backups** (Day 2)
   - Set up Restic with B2
   - Configure backup scripts
   - Test restoration

6. **Coolify** (Day 3)
   - Install Coolify
   - Configure server connection
   - Deploy first test app

7. **Remote Development** (Day 3)
   - Deploy code-server
   - Configure access via VPN
   - Install extensions

8. **Testing & Validation** (Day 4)
   - Test all services
   - Verify backups
   - Simulate failure scenarios
   - Document any custom configurations

## Estimated Costs

### One-Time Costs
- Domain names: ~$15/year × 2 = $30/year
- Server: Already owned ($0)

### Recurring Costs
- **Cloudflare**: $0 (free tier sufficient)
- **Tailscale**: $0 (free tier, 100 devices)
- **Backblaze B2**: ~$0.50/month (for ~100GB backups)
- **All software**: $0 (100% open source)

**Total monthly cost: < $1**

Compare to managed alternatives:
- Heroku: $25-100/month
- Vercel: $20-50/month
- AWS/GCP: $50-200/month
- GitHub Codespaces: $18+/month

**Your setup: < $1/month!**

## Questions & Decisions Log

Keep track of architectural decisions:

| Date | Decision | Reasoning | Impact |
|------|----------|-----------|--------|
| 2024-XX-XX | Use Tailscale over Headscale | Easier setup, can migrate later | Faster deployment |
| 2024-XX-XX | Use Caddy over Nginx | Automatic HTTPS, simpler config | Easier maintenance |
| 2024-XX-XX | Use code-server over Coder | Solo developer, don't need multi-user | Lower complexity |
| 2024-XX-XX | Backblaze B2 for backups | Cost-effective, S3-compatible | ~$0.50/month |

## Resources

- This documentation repo
- Vendor documentation (links in each guide)
- Your notes and customizations

## Next Steps

1. Read through all documentation
2. Follow implementation order above
3. Customize for your specific needs
4. Document any changes you make
5. Keep this repo updated as your setup evolves

---

**Remember**: This architecture is designed to be secure, reliable, and cost-effective for a solo developer or small team. As your needs grow, you can scale horizontally by adding more servers while keeping the same architecture.
