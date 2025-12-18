# Security Hardening Guide for Fedora 43 Server

## Overview

This guide covers security best practices for your Fedora 43 server to protect against common threats and vulnerabilities.

## Security Philosophy

**Defense in Depth**: Multiple layers of security, so if one layer fails, others protect you.

Layers:
1. **Network**: Firewall, VPN, limited exposure
2. **System**: Hardened OS, regular updates
3. **Application**: Secure configurations, limited permissions
4. **Data**: Encryption, backups
5. **Monitoring**: Detect and respond to threats

## Initial Security Setup

### Step 1: Update System

**Always keep your system updated!**

```bash
# Update all packages
sudo dnf update -y

# Enable automatic security updates
sudo dnf install -y dnf-automatic

# Configure automatic updates
sudo nano /etc/dnf/automatic.conf
```

Edit `/etc/dnf/automatic.conf`:
```ini
[commands]
upgrade_type = security
apply_updates = yes
download_updates = yes

[emitters]
emit_via = stdio

[email]
email_from = root@antoinelb.fr
email_to = you@antoinelb.fr
```

Enable automatic updates:
```bash
sudo systemctl enable --now dnf-automatic.timer
sudo systemctl status dnf-automatic.timer
```

### Step 2: Configure Firewall

Fedora uses `firewalld` by default.

```bash
# Check firewall status
sudo firewall-cmd --state

# Get active zones
sudo firewall-cmd --get-active-zones

# List current rules
sudo firewall-cmd --list-all
```

#### Recommended Firewall Configuration

```bash
# Set default zone to drop (deny everything by default)
sudo firewall-cmd --set-default-zone=drop

# Create a custom zone for your services
sudo firewall-cmd --permanent --new-zone=services
sudo firewall-cmd --permanent --zone=services --add-source=0.0.0.0/0

# Allow SSH (temporarily, will restrict to VPN later)
sudo firewall-cmd --permanent --zone=services --add-service=ssh

# Allow HTTP and HTTPS (for Caddy)
sudo firewall-cmd --permanent --zone=services --add-service=http
sudo firewall-cmd --permanent --zone=services --add-service=https

# Allow Tailscale VPN (UDP port 41641)
sudo firewall-cmd --permanent --zone=services --add-port=41641/udp

# Reload firewall
sudo firewall-cmd --reload

# Verify
sudo firewall-cmd --zone=services --list-all
```

#### After VPN is Set Up: Restrict SSH to VPN Only

```bash
# Remove SSH from public zone
sudo firewall-cmd --permanent --zone=services --remove-service=ssh

# Create VPN zone (Tailscale IP range: 100.64.0.0/10)
sudo firewall-cmd --permanent --new-zone=vpn
sudo firewall-cmd --permanent --zone=vpn --add-source=100.64.0.0/10
sudo firewall-cmd --permanent --zone=vpn --add-service=ssh

# Reload
sudo firewall-cmd --reload
```

Now SSH is only accessible via VPN!

### Step 3: Secure SSH

#### Create Non-Root User (If Not Done)

```bash
# Create user
sudo useradd -m -G wheel antoine

# Set password
sudo passwd antoine

# Test sudo access
su - antoine
sudo whoami  # Should output: root
```

#### Harden SSH Configuration

Edit `/etc/ssh/sshd_config`:

```bash
sudo nano /etc/ssh/sshd_config
```

Recommended settings:
```
# Disable root login
PermitRootLogin no

# Use SSH keys only (disable password auth)
PasswordAuthentication no
PubkeyAuthentication yes

# Disable empty passwords
PermitEmptyPasswords no

# Disable X11 forwarding (if not needed)
X11Forwarding no

# Use strong ciphers only
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr,aes192-ctr,aes128-ctr

# Use strong MACs only
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,hmac-sha2-512,hmac-sha2-256

# Limit authentication attempts
MaxAuthTries 3

# Enable strict mode
StrictModes yes

# Set client timeout (disconnect idle clients)
ClientAliveInterval 300
ClientAliveCountMax 2

# Disable unused authentication methods
ChallengeResponseAuthentication no
KerberosAuthentication no
GSSAPIAuthentication no
```

Restart SSH:
```bash
sudo systemctl restart sshd
```

**Important**: Test SSH access in a new terminal before closing your current session!

#### Set Up SSH Key Authentication

On your **local machine**:
```bash
# Generate SSH key (if you don't have one)
ssh-keygen -t ed25519 -C "you@antoinelb.fr"

# Copy public key to server
ssh-copy-id antoine@your-server-ip
```

On the **server**:
```bash
# Verify key was added
cat ~/.ssh/authorized_keys

# Set correct permissions
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

Test SSH key login:
```bash
ssh antoine@your-server-ip
# Should log in without password
```

Once confirmed working, disable password authentication (see sshd_config above).

### Step 4: Install and Configure Fail2Ban

Fail2Ban bans IPs with repeated failed login attempts.

```bash
# Install
sudo dnf install -y fail2ban fail2ban-firewalld

# Enable and start
sudo systemctl enable --now fail2ban
```

Configure Fail2Ban:

```bash
sudo nano /etc/fail2ban/jail.local
```

```ini
[DEFAULT]
# Ban for 1 hour
bantime = 3600
# Check for 5 failures
maxretry = 5
# Within 10 minutes
findtime = 600
# Use firewalld
banaction = firewallcmd-rich-rules

[sshd]
enabled = true
port = 22
logpath = /var/log/secure
```

Restart Fail2Ban:
```bash
sudo systemctl restart fail2ban

# Check status
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

### Step 5: Enable SELinux

Fedora has SELinux enabled by default. Verify:

```bash
# Check status
sestatus

# Should show: SELinux status: enabled
# Mode: enforcing
```

If disabled, enable it:
```bash
sudo nano /etc/selinux/config
```

Set:
```
SELINUX=enforcing
SELINUXTYPE=targeted
```

Reboot:
```bash
sudo reboot
```

#### Common SELinux Issues and Solutions

If services fail due to SELinux:

```bash
# Check SELinux denials
sudo ausearch -m avc -ts recent

# Generate and apply policy module
sudo ausearch -m avc -ts recent | audit2allow -M mypolicy
sudo semodule -i mypolicy.pp

# Or for Caddy/Docker (common issues):
sudo setsebool -P httpd_can_network_connect on
```

For Docker:
```bash
# Allow Docker to manage network
sudo setsebool -P container_manage_cgroup on
```

### Step 6: Harden System Limits

Edit `/etc/security/limits.conf`:

```bash
sudo nano /etc/security/limits.conf
```

Add:
```
# Limit core dumps
* hard core 0

# Limit number of processes
* soft nproc 1024
* hard nproc 2048

# Limit open files
* soft nofile 65536
* hard nofile 65536
```

### Step 7: Kernel Hardening

Edit `/etc/sysctl.d/99-security.conf`:

```bash
sudo nano /etc/sysctl.d/99-security.conf
```

Add:
```conf
# IP forwarding (disable unless needed for VPN/routing)
net.ipv4.ip_forward = 0
net.ipv6.conf.all.forwarding = 0

# Disable ICMP redirect acceptance
net.ipv4.conf.all.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0

# Disable ICMP redirect sending
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0

# Enable SYN cookies (DDoS protection)
net.ipv4.tcp_syncookies = 1

# Disable source packet routing
net.ipv4.conf.all.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0

# Log martian packets
net.ipv4.conf.all.log_martians = 1

# Ignore ICMP ping requests (optional, can break some monitoring)
# net.ipv4.icmp_echo_ignore_all = 1

# Enable bad error message protection
net.ipv4.icmp_ignore_bogus_error_responses = 1

# Enable reverse path filtering
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# Disable IPv6 (if not using it)
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1

# Randomize kernel addresses (KASLR)
kernel.randomize_va_space = 2

# Restrict kernel pointer access
kernel.kptr_restrict = 2

# Restrict dmesg access
kernel.dmesg_restrict = 1

# Disable ptrace for non-root
kernel.yama.ptrace_scope = 2
```

Apply settings:
```bash
sudo sysctl -p /etc/sysctl.d/99-security.conf
```

### Step 8: Secure Shared Memory

Prevent privilege escalation via shared memory:

```bash
sudo nano /etc/fstab
```

Add:
```
tmpfs /run/shm tmpfs defaults,noexec,nodev,nosuid 0 0
```

Remount:
```bash
sudo mount -o remount /run/shm
```

### Step 9: Install Security Tools

```bash
# Install security audit tools
sudo dnf install -y lynis aide rkhunter

# Lynis - security auditing tool
sudo lynis audit system

# AIDE - file integrity checker
sudo aide --init
sudo mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz

# Check for changes
sudo aide --check

# RKHunter - rootkit detection
sudo rkhunter --update
sudo rkhunter --check
```

Schedule regular checks:
```bash
# Add to crontab
sudo crontab -e
```

```
# Run AIDE daily at 2 AM
0 2 * * * /usr/sbin/aide --check | mail -s "AIDE Report" you@antoinelb.fr

# Run RKHunter weekly
0 3 * * 0 /usr/bin/rkhunter --check --skip-keypress | mail -s "RKHunter Report" you@antoinelb.fr
```

## Docker Security

### Step 1: Docker Daemon Hardening

Create `/etc/docker/daemon.json`:

```json
{
  "icc": false,
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "userland-proxy": false,
  "no-new-privileges": true,
  "live-restore": true,
  "userns-remap": "default"
}
```

Restart Docker:
```bash
sudo systemctl restart docker
```

### Step 2: Run Containers with Limited Privileges

```bash
# Always use these flags when running containers:

# Drop capabilities
--cap-drop=ALL --cap-add=NET_BIND_SERVICE

# Read-only root filesystem (when possible)
--read-only

# No new privileges
--security-opt=no-new-privileges

# Limited resources
--memory="512m" --cpus="1.0"

# Non-root user
--user 1000:1000
```

Example:
```bash
docker run -d \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --security-opt=no-new-privileges \
  --memory="512m" \
  --cpus="1.0" \
  --read-only \
  -v /tmp:/tmp \
  your-image
```

### Step 3: Scan Docker Images

```bash
# Install Trivy (vulnerability scanner)
sudo dnf install -y trivy

# Scan image for vulnerabilities
trivy image your-image:tag

# Only show HIGH and CRITICAL
trivy image --severity HIGH,CRITICAL your-image:tag
```

### Step 4: Use Docker Secrets

Never put secrets in environment variables or Dockerfiles!

```bash
# Create secret
echo "my-secret-value" | docker secret create my_secret -

# Use in docker-compose.yml
version: '3.8'
services:
  app:
    image: myapp
    secrets:
      - my_secret

secrets:
  my_secret:
    external: true
```

## Application Security

### Step 1: Caddy Security Headers

Add security headers to all sites in Caddyfile:

```caddyfile
(security_headers) {
    header {
        # Enable HSTS
        Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"

        # Prevent clickjacking
        X-Frame-Options "DENY"

        # Prevent MIME sniffing
        X-Content-Type-Options "nosniff"

        # XSS protection
        X-XSS-Protection "1; mode=block"

        # Referrer policy
        Referrer-Policy "strict-origin-when-cross-origin"

        # Content Security Policy
        Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self'; connect-src 'self'; frame-ancestors 'none';"

        # Permissions policy
        Permissions-Policy "geolocation=(), microphone=(), camera=()"

        # Remove server header
        -Server
    }
}

# Use in your sites
coolify.antoinelb.fr {
    import security_headers
    reverse_proxy localhost:3000
}
```

### Step 2: Rate Limiting

Protect against brute force and DDoS:

```caddyfile
coolify.antoinelb.fr {
    # Rate limit login endpoints
    @login {
        path /login /api/auth/*
    }

    # Max 5 requests per minute per IP
    rate_limit @login {
        zone dynamic
        rate 5/m
        window 1m
    }

    reverse_proxy localhost:3000
}
```

### Step 3: Database Security

#### PostgreSQL

```bash
# Connect to PostgreSQL
docker exec -it postgres-container psql -U postgres

# In psql:
-- Don't use default 'postgres' user for apps
CREATE USER myapp WITH ENCRYPTED PASSWORD 'strong-password';
CREATE DATABASE myapp_db OWNER myapp;

-- Grant minimal privileges
GRANT CONNECT ON DATABASE myapp_db TO myapp;
GRANT USAGE ON SCHEMA public TO myapp;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO myapp;

-- Never grant SUPERUSER to app users!
```

Configure PostgreSQL (`postgresql.conf`):
```
# Listen only on localhost (unless needed elsewhere)
listen_addresses = 'localhost'

# Enable SSL
ssl = on
ssl_cert_file = '/path/to/cert.pem'
ssl_key_file = '/path/to/key.pem'

# Require SSL connections
ssl_min_protocol_version = 'TLSv1.2'
```

#### MySQL

```sql
-- Create user with limited privileges
CREATE USER 'myapp'@'localhost' IDENTIFIED BY 'strong-password';
CREATE DATABASE myapp_db;
GRANT SELECT, INSERT, UPDATE, DELETE ON myapp_db.* TO 'myapp'@'localhost';
FLUSH PRIVILEGES;

-- Never use root for applications!
```

Configure MySQL (`/etc/mysql/my.cnf`):
```ini
[mysqld]
# Bind to localhost only
bind-address = 127.0.0.1

# Disable local infile
local-infile = 0

# Enable SSL
require_secure_transport = ON
```

## Monitoring and Auditing

### Step 1: Enable Audit Logging

```bash
# Install auditd
sudo dnf install -y audit

# Enable and start
sudo systemctl enable --now auditd
```

Configure audit rules (`/etc/audit/rules.d/audit.rules`):

```bash
sudo nano /etc/audit/rules.d/audit.rules
```

```
# Monitor authentication events
-w /var/log/lastlog -p wa -k logins
-w /var/run/faillock/ -p wa -k logins

# Monitor password changes
-w /etc/passwd -p wa -k passwd_changes
-w /etc/shadow -p wa -k passwd_changes
-w /etc/group -p wa -k group_changes
-w /etc/gshadow -p wa -k group_changes

# Monitor sudo usage
-w /etc/sudoers -p wa -k sudoers_changes
-w /etc/sudoers.d/ -p wa -k sudoers_changes

# Monitor SSH
-w /etc/ssh/sshd_config -p wa -k sshd_config_changes

# Monitor important binaries
-w /usr/bin/docker -p x -k docker_execution
-w /usr/sbin/iptables -p x -k firewall_changes

# Monitor file deletions
-a always,exit -F arch=b64 -S unlink -S unlinkat -S rename -S renameat -k delete
```

Load rules:
```bash
sudo augenrules --load
sudo systemctl restart auditd
```

View audit logs:
```bash
# Search for events
sudo ausearch -k passwd_changes
sudo ausearch -k logins -ts recent

# Generate report
sudo aureport --auth
sudo aureport --failed
```

### Step 2: Log Monitoring

Integrate logs with your monitoring stack (see monitoring guide).

Important logs to monitor:
- `/var/log/secure` - SSH, auth events
- `/var/log/audit/audit.log` - Audit events
- `/var/log/firewalld` - Firewall logs
- Docker logs: `docker logs <container>`

Set up alerts for:
- Failed SSH login attempts
- Sudo usage
- Firewall blocks
- Service failures
- Unusual disk/CPU/memory usage

## Secrets Management

### Step 1: Never Store Secrets in Code

Bad:
```bash
# .env file in Git
DATABASE_PASSWORD=mysecretpassword
```

Good:
```bash
# Use environment variables set outside of Git
# Or use proper secrets management
```

### Step 2: Use Docker Secrets or Environment Files

```yaml
# docker-compose.yml
version: '3.8'
services:
  app:
    image: myapp
    env_file:
      - .env.production  # Not in Git!
    secrets:
      - db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt  # Not in Git!
```

`.gitignore`:
```
.env*
secrets/
*.key
*.pem
```

### Step 3: Rotate Secrets Regularly

```bash
# Generate new password
NEW_PASS=$(openssl rand -base64 32)

# Update database password
docker exec postgres psql -U postgres -c "ALTER USER myapp WITH PASSWORD '$NEW_PASS';"

# Update application config
echo "DATABASE_PASSWORD=$NEW_PASS" > .env.production

# Restart application
docker compose restart app
```

Schedule password rotation quarterly or after any security incident.

## Incident Response Plan

### Detection

1. **Monitor alerts** from Grafana/Prometheus
2. **Check logs** regularly for anomalies
3. **Review Fail2Ban** bans
4. **Audit system** with Lynis monthly

### Response

If you detect suspicious activity:

```bash
# 1. Identify the threat
sudo ausearch -ts recent | grep suspicious-activity
sudo journalctl -xe

# 2. Contain (if confirmed breach)
# Block attacker IP
sudo firewall-cmd --add-rich-rule="rule family='ipv4' source address='ATTACKER_IP' reject"

# Or disconnect server from network temporarily
sudo systemctl stop network

# 3. Eradicate
# Kill suspicious processes
sudo kill -9 <PID>

# Remove malicious files
sudo rm /path/to/malicious/file

# 4. Recover
# Restore from clean backup
restic restore latest --target /restore

# 5. Post-incident
# Change all passwords
# Review and update security policies
# Document incident
```

### Backup and Recovery

See the monitoring-backups guide. Ensure you can:
1. Restore from backup quickly
2. Have offline backup copies
3. Test restores regularly

## Regular Maintenance Tasks

### Daily (Automated)
- Security updates applied
- Backups run
- Logs rotated

### Weekly (Manual)
- Review failed login attempts
- Check Fail2Ban status
- Review firewall logs

### Monthly
- Run Lynis audit: `sudo lynis audit system`
- Run RKHunter: `sudo rkhunter --check`
- Review user accounts and permissions
- Update Docker images
- Test backup restoration

### Quarterly
- Rotate secrets and passwords
- Review and update firewall rules
- Review and update security policies
- Security assessment of all services

## Security Checklist

Use this checklist when setting up your server:

- [ ] System updated and automatic updates enabled
- [ ] Firewall configured (only essential ports open)
- [ ] SSH hardened (key-only, no root login)
- [ ] VPN set up for remote access
- [ ] Fail2Ban installed and configured
- [ ] SELinux enabled and enforcing
- [ ] Kernel hardened with sysctl settings
- [ ] Non-root user created with sudo access
- [ ] Docker daemon hardened
- [ ] Security headers configured in Caddy
- [ ] Databases secured (no root access for apps)
- [ ] Audit logging enabled
- [ ] Monitoring and alerting set up
- [ ] Backup system configured and tested
- [ ] Secrets not stored in code/Git
- [ ] All services accessible only via VPN (where appropriate)
- [ ] SSL certificates configured for all services
- [ ] Password policy enforced
- [ ] Log rotation configured
- [ ] Intrusion detection (AIDE, RKHunter) installed
- [ ] Regular security audits scheduled

## Advanced: Two-Factor Authentication

### For SSH

```bash
# Install Google Authenticator
sudo dnf install -y google-authenticator

# Set up for your user
google-authenticator

# Follow prompts, save backup codes!

# Configure SSH PAM
sudo nano /etc/pam.d/sshd
```

Add:
```
auth required pam_google_authenticator.so
```

Edit `/etc/ssh/sshd_config`:
```
ChallengeResponseAuthentication yes
AuthenticationMethods publickey,keyboard-interactive
```

Restart SSH:
```bash
sudo systemctl restart sshd
```

Now SSH requires both key AND TOTP code!

### For Web Services

Use Caddy's authentication plugins or integrate OAuth/OIDC:

```bash
# Install Caddy with authentication module
caddy add-package github.com/greenpau/caddy-security
```

## Resources

- **Fedora Security Guide**: https://docs.fedoraproject.org/en-US/fedora-server/administration/security/
- **CIS Benchmarks**: https://www.cisecurity.org/cis-benchmarks/
- **OWASP**: https://owasp.org/
- **Docker Security**: https://docs.docker.com/engine/security/
- **Lynis**: https://cisofy.com/lynis/
- **Fail2Ban**: https://www.fail2ban.org/

## Next Steps

1. Work through security checklist
2. Run initial Lynis audit
3. Configure firewall rules
4. Set up Fail2Ban
5. Harden SSH
6. Configure audit logging
7. Set up VPN (see VPN guide)
8. Restrict services to VPN-only access
9. Regular security audits

## Important Reminder

**Security is a process, not a product.**

- Stay updated on security news
- Regularly review and update configurations
- Test your backups and incident response
- Learn from security incidents (yours and others')
- Assume you will be breached, plan accordingly

**The most important security measures for your setup:**
1. Keep everything updated
2. Use VPN for all admin access (SSH, Coolify, monitoring)
3. Strong, unique passwords (use a password manager)
4. Regular backups with tested restoration
5. Monitor logs and set up alerts

Follow these principles and you'll be more secure than 90% of self-hosted servers!
