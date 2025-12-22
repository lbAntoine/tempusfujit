# Step-by-Step Security Hardening Guide

**Difficulty**: Intermediate
**Prerequisites**:

- Fedora 43 Server with SSH key authentication already working
- User with sudo privileges (fedora)

---

## ⚠️ Safety Rules

1. **Keep 2 SSH terminals open** at all times
2. **Test SSH in a new terminal** after each service restart
3. **Don't skip verification steps**

---

## Current Working State

Before we start, verify your current setup:

```bash
# 1. You're logged in as fedora user
whoami
# Output: fedora

# 2. You have sudo access
sudo whoami
# Output: root

# 3. SSH is working (obviously, since you're connected)
sudo systemctl status sshd | grep Active
# Output: Active: active (running)

# 4. Check current SSH config
sudo sshd -T | grep -E 'passwordauth|pubkeyauth|permitroot'
# Note the current values
```

---

## Step 1: System Updates

```bash
# Update system
sudo dnf update -y

# Install automatic security updates
sudo dnf install -y dnf-automatic

# Configure for security updates only
sudo sed -i 's/^upgrade_type = .*/upgrade_type = security/' /etc/dnf/automatic.conf
sudo sed -i 's/^apply_updates = .*/apply_updates = yes/' /etc/dnf/automatic.conf

# Enable automatic updates
sudo systemctl enable --now dnf-automatic.timer

# Verify
systemctl is-active dnf-automatic.timer
```

✅ **Expected**: `active`

---

## Step 2: Firewall Configuration

**Current situation**: Firewall might be blocking or misconfigured.

### Check Current Firewall State

```bash
# Check firewall status
sudo firewall-cmd --state
# Should output: running

# Check default zone and services
sudo firewall-cmd --get-default-zone
sudo firewall-cmd --list-all
```

### Simple Working Configuration

```bash
# Use the default zone (usually public or FedoraServer)
DEFAULT_ZONE=$(sudo firewall-cmd --get-default-zone)

# Ensure SSH is allowed in default zone
sudo firewall-cmd --permanent --zone=$DEFAULT_ZONE --add-service=ssh

# Add HTTP and HTTPS for Caddy (later)
sudo firewall-cmd --permanent --zone=$DEFAULT_ZONE --add-service=http
sudo firewall-cmd --permanent --zone=$DEFAULT_ZONE --add-service=https

# Add Tailscale port (for VPN later)
sudo firewall-cmd --permanent --zone=$DEFAULT_ZONE --add-port=41641/udp

# Reload firewall
sudo firewall-cmd --reload
```

### Verification

```bash
# Check services are allowed
sudo firewall-cmd --list-all

# Your current SSH connection should still work
echo "SSH still working!"
```

**⚠️ CRITICAL TEST**: Open a **NEW terminal** and test SSH:

```bash
ssh fedora@YOUR_SERVER_IP
```

✅ **Must work before continuing!**

---

## Step 3: SSH Configuration (Careful!)

**You already have working SSH with keys. We're just adding security settings.**

### Backup Current Config

```bash
# Backup
sudo cp -r /etc/ssh /etc/ssh.backup.$(date +%Y%m%d-%H%M%S)

# Verify backup exists
ls -ld /etc/ssh.backup.*
```

### Check What's Already There

```bash
# Check included configs
ls -la /etc/ssh/sshd_config.d/

# View their contents
sudo cat /etc/ssh/sshd_config.d/*.conf
```

### Add Safe Hardening Settings

```bash
# Create hardening config
sudo tee /etc/ssh/sshd_config.d/99-hardening.conf > /dev/null << 'EOF'
# Additional SSH hardening
# Password auth already disabled by cloud-init

# Disable root login
PermitRootLogin no

# Limit auth attempts
MaxAuthTries 3

# Set timeouts
ClientAliveInterval 300
ClientAliveCountMax 2

# Disable unused auth methods
ChallengeResponseAuthentication no
KerberosAuthentication no
GSSAPIAuthentication no

# Strong ciphers only
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr,aes192-ctr,aes128-ctr
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,hmac-sha2-512,hmac-sha2-256
EOF
```

### Test Config Before Applying

```bash
# Test for syntax errors
sudo sshd -t

# Should show NO output (no errors)
```

**If errors shown**: Fix them before continuing!

### Apply Config (Carefully!)

```bash
# Restart SSH
sudo systemctl restart sshd

# Check it's running
sudo systemctl status sshd | grep Active
# Should show: Active: active (running)
```

### ⚠️ CRITICAL TEST

**In a NEW terminal** (don't close your current one):

```bash
ssh fedora@YOUR_SERVER_IP
```

**Did it work?**

- ✅ **YES**: Continue
- ❌ **NO**: Use your existing terminal to restore backup:

  ```bash
  sudo rm /etc/ssh/sshd_config.d/99-hardening.conf
  sudo systemctl restart sshd
  # Try connecting again
  ```

---

## Step 4: SELinux (OPTIONAL - Skip for Now)

**Note**: SELinux provides additional security through mandatory access controls, but it adds complexity and can break SSH if misconfigured. **You can safely skip this step** and come back to it later once you're comfortable with the other security measures and VPN setup.

<details>
<summary>Click here if you want to enable SELinux (Advanced)</summary>

**Problem**: SELinux contexts not set correctly can break SSH on reboot.

### Check Current Status

```bash
getenforce
```

**If Enforcing**: Already enabled, just verify contexts below.
**If Permissive/Disabled**: We'll enable it carefully.

### Fix SSH File Contexts (Critical!)

```bash
# Fix all SSH contexts
sudo restorecon -R -v /root/.ssh 2>/dev/null || true
sudo restorecon -R -v /home/fedora/.ssh
sudo restorecon -R -v /etc/ssh
sudo restorecon -v /usr/sbin/sshd

# Verify authorized_keys context
ls -lZ ~/.ssh/authorized_keys
# Should show: ssh_home_t
```

### Enable SELinux (If Not Already)

```bash
# Check current mode
getenforce

# If not Enforcing:
sudo setenforce 1

# Verify
getenforce
# Should output: Enforcing
```

### Test SSH with SELinux Enforcing

**In a NEW terminal**:

```bash
ssh fedora@YOUR_SERVER_IP
```

**Did it work?**

- ✅ **YES**: SELinux properly configured
- ❌ **NO**: Revert immediately:

  ```bash
  sudo setenforce 0
  sudo restorecon -R -v ~/.ssh
  # Try again
  ```

### Make Permanent

```bash
# Edit config
sudo sed -i 's/^SELINUX=.*/SELINUX=enforcing/' /etc/selinux/config

# Verify
grep ^SELINUX= /etc/selinux/config
# Should show: SELINUX=enforcing
```

✅ **Success**: SELinux enabled, SSH still works

</details>

**Skipping SELinux?** Just continue to Step 5 below.

---

## Step 5: Install CrowdSec

**Problem before**: CrowdSec might ban your IP if configured wrong.

### Installation

```bash
# Install CrowdSec (official installer)
curl -s https://install.crowdsec.net | sudo bash

# Install firewall bouncer
sudo dnf install -y crowdsec-firewall-bouncer-iptables

# Start services
sudo systemctl enable --now crowdsec
sudo systemctl enable --now crowdsec-firewall-bouncer
```

### Install Collections

```bash
# Install protection collections
sudo cscli collections install crowdsecurity/linux
sudo cscli collections install crowdsecurity/sshd
sudo cscli collections install crowdsecurity/http-cve

# Reload CrowdSec
sudo systemctl reload crowdsec
```

### Whitelist Your IP (Important!)

```bash
# Get your current IP
MY_IP=$(echo $SSH_CLIENT | awk '{print $1}')
echo "Your IP: $MY_IP"

# Whitelist it
sudo cscli decisions add --ip $MY_IP --duration 999h --type captcha --reason "My IP - never ban"

# Verify
sudo cscli decisions list
```

### Configure Bouncer

```bash
# Check bouncer config
sudo cat /etc/crowdsec/bouncers/crowdsec-firewall-bouncer.yaml | grep -E "mode:|deny_action:"

# Should show:
# mode: nftables (or iptables)
# deny_action: DROP
```

### Verification

```bash
# Check services running
systemctl is-active crowdsec crowdsec-firewall-bouncer

# View metrics
sudo cscli metrics

# Check scenarios
sudo cscli scenarios list | grep enabled
```

✅ **Success**: CrowdSec installed and your IP whitelisted

---

## Step 6: Kernel Hardening

```bash
# Create security config
sudo tee /etc/sysctl.d/99-security.conf > /dev/null << 'EOF'
# Network security
net.ipv4.ip_forward = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.all.log_martians = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# Kernel security
kernel.randomize_va_space = 2
kernel.kptr_restrict = 2
kernel.dmesg_restrict = 1
kernel.yama.ptrace_scope = 2

# Disable IPv6 if not needed
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
EOF

# Apply
sudo sysctl -p /etc/sysctl.d/99-security.conf

# Verify
sysctl net.ipv4.tcp_syncookies
```

✅ **Success**: Kernel hardened

---

## Step 7: Secure Shared Memory

```bash
# Backup fstab
sudo cp /etc/fstab /etc/fstab.backup

# Add secure mount
echo "tmpfs /run/shm tmpfs defaults,noexec,nodev,nosuid 0 0" | sudo tee -a /etc/fstab

# Remount
sudo mount -o remount /run/shm

# Verify
mount | grep /run/shm | grep -o 'noexec,nodev,nosuid'
```

✅ **Success**: Shared memory secured

---

## Step 8: Security Tools

```bash
# Install
sudo dnf install -y lynis aide rkhunter

# Initialize AIDE
sudo aide --init
sudo mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz

# Update RKHunter
sudo rkhunter --update

# Run Lynis (optional - takes a few minutes)
sudo lynis audit system --quick
```

✅ **Success**: Security tools installed

---

## Step 9: Audit Logging

```bash
# Install
sudo dnf install -y audit

# Enable
sudo systemctl enable --now auditd

# Create rules
sudo tee /etc/audit/rules.d/audit.rules > /dev/null << 'EOF'
# Monitor authentication
-w /var/log/lastlog -p wa -k logins
-w /var/run/faillock/ -p wa -k logins

# Monitor password changes
-w /etc/passwd -p wa -k passwd_changes
-w /etc/shadow -p wa -k passwd_changes

# Monitor sudo usage
-w /etc/sudoers -p wa -k sudoers_changes
-w /etc/sudoers.d/ -p wa -k sudoers_changes

# Monitor SSH config
-w /etc/ssh/sshd_config -p wa -k sshd_config

# Monitor firewall
-w /etc/firewalld/ -p wa -k firewall_changes

# Monitor file deletions
-a always,exit -F arch=b64 -S unlink -S unlinkat -S rename -S renameat -k delete
EOF

# Load rules
sudo augenrules --load

# Restart auditd
sudo service auditd restart

# Verify
sudo auditctl -l | head -5
```

✅ **Success**: Audit logging enabled

---

## Step 10: System Limits

```bash
# Add limits
sudo tee -a /etc/security/limits.conf > /dev/null << 'EOF'

# Security limits
* hard core 0
* soft nproc 1024
* hard nproc 2048
* soft nofile 65536
* hard nofile 65536
EOF

# Verify
tail -5 /etc/security/limits.conf
```

✅ **Success**: System limits configured

---

## Final Verification

```bash
# Create verification script
tee ~/security-check.sh > /dev/null << 'EOF'
#!/bin/bash
echo "=== Security Status ==="
echo ""
echo "System Updates:"
systemctl is-active dnf-automatic.timer && echo "✓ Automatic updates active" || echo "✗ Inactive"
echo ""
echo "Firewall:"
sudo firewall-cmd --list-all | grep -q ssh && echo "✓ SSH allowed" || echo "✗ SSH not allowed"
echo ""
echo "SSH Service:"
systemctl is-active sshd && echo "✓ SSH active" || echo "✗ SSH inactive"
echo ""
echo "CrowdSec:"
systemctl is-active crowdsec && echo "✓ CrowdSec active" || echo "✗ Inactive"
systemctl is-active crowdsec-firewall-bouncer && echo "✓ Bouncer active" || echo "✗ Inactive"
echo ""
echo "SELinux (optional):"
[ "$(getenforce)" = "Enforcing" ] && echo "✓ SELinux enforcing" || echo "○ SELinux not enforcing (skipped)"
echo ""
echo "Auditd:"
systemctl is-active auditd && echo "✓ Auditd active" || echo "✗ Inactive"
echo ""
EOF

chmod +x ~/security-check.sh
~/security-check.sh
```

**All items should show ✓**

---

## Final SSH Test

**Before finishing, test SSH one more time in a NEW terminal**:

```bash
ssh fedora@YOUR_SERVER_IP
```

✅ **Works? You're done!**

---

## If Something Breaks

### SSH Not Working After Config Change

```bash
# In existing terminal (still open):
sudo rm /etc/ssh/sshd_config.d/99-hardening.conf
sudo systemctl restart sshd
# Try connecting again
```

### CrowdSec Banned Your IP

```bash
# Check if you're banned
sudo cscli decisions list

# Remove ban
sudo cscli decisions delete --ip YOUR_IP

# Add whitelist
sudo cscli decisions add --ip YOUR_IP --duration 999h --type captcha --reason "My IP"
```

### Firewall Blocking SSH

```bash
# Check SSH is allowed
sudo firewall-cmd --list-all | grep ssh

# Add SSH if missing
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
```

### SELinux Blocking SSH (If You Enabled It)

```bash
# Temporarily disable
sudo setenforce 0

# Fix contexts
sudo restorecon -R -v ~/.ssh

# Re-enable
sudo setenforce 1
```

---

## Summary

✅ **System**: Updated, automatic updates enabled
✅ **Firewall**: Configured, SSH allowed
✅ **SSH**: Hardened, still working
✅ **CrowdSec**: Protecting, your IP whitelisted
✅ **Monitoring**: Audit logging enabled
✅ **Tools**: Security tools installed
○ **SELinux**: Optional (you can enable later if desired)

**Most Important**: You tested SSH in a new terminal after each critical change!

---

## Next Steps

1. Review security check: `~/security-check.sh`
2. Optional: Reboot to test everything survives restart
3. Proceed to VPN setup guide (next) - learn to manage VPN and non-VPN access

**Before rebooting (optional test)**:

```bash
# Reboot to verify all services start correctly
sudo reboot

# Wait 2-3 minutes for system to restart
# Then test SSH again
ssh fedora@YOUR_SERVER_IP
```

Only reboot if you want to test. It's not required - everything is already working!

**What you'll learn in the VPN guide**: How to set up Tailscale VPN for secure remote access, configure firewall rules to manage VPN vs non-VPN traffic, and control which services are accessible from the public internet vs only through the VPN.
