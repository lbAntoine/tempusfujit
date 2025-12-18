# Step-by-Step Security Hardening Guide (Safe Version)

**Estimated Time**: 2-3 hours
**Difficulty**: Intermediate
**Prerequisites**: Root or sudo access to Fedora 43 Server

---

## 🛡️ CRITICAL SAFETY RULES

**READ THIS BEFORE STARTING:**

1. **ALWAYS keep 2-3 terminal windows open** to your server
2. **NEVER close all sessions** until you've tested the new one works
3. **NEVER reboot** until SSH is verified working in multiple new terminals
4. **Test every change** in a new terminal before making it permanent
5. **Have console/IPMI access ready** as backup
6. **Take a snapshot** before starting (if your provider supports it)

**If anything goes wrong**: Keep one terminal open, undo the change, restart the service, test again.

---

## Pre-Flight Checklist

Before we begin, verify you have:

```bash
# 1. You're logged in via SSH
echo "Current connection: SSH"

# 2. You have sudo/root access
sudo whoami
# Should output: root

# 3. Check your current user
whoami
# Remember this username - you'll need it!

# 4. Note your server IP
ip addr show | grep "inet " | grep -v 127.0.0.1
# Write down your public IP address

# 5. Verify internet connectivity
ping -c 3 1.1.1.1
# Should show replies
```

✅ **All checks passed? Continue. Any failed? Fix before proceeding.**

---

## Step 1: System Update

### What We're Doing
Update all packages to get security fixes.

### Commands

```bash
# 1.1: Update all packages
sudo dnf update -y
```

**Expected**: Package list, download progress, "Complete!"

### Install Automatic Updates

```bash
# 1.2: Install automatic update tool
sudo dnf install -y dnf-automatic
```

```bash
# 1.3: Configure automatic security updates
sudo nano /etc/dnf/automatic.conf
```

**Change these lines**:
```ini
[commands]
upgrade_type = security
apply_updates = yes
download_updates = yes
```

**Save**: `Ctrl+X`, `Y`, `Enter`

```bash
# 1.4: Enable automatic updates
sudo systemctl enable --now dnf-automatic.timer
```

### Verification

```bash
# Check no pending updates
sudo dnf check-update

# Check timer is active
systemctl status dnf-automatic.timer | grep Active
# Should show: Active: active (waiting)
```

✅ **Success**: System updated, automatic updates enabled

---

## Step 2: Firewall Configuration

### What We're Doing
Configure firewall to only allow necessary services.

### ⚠️ SAFETY NOTE
Firewall changes are SAFE - they don't affect existing SSH connections. Your current session will stay connected.

### Check Current Status

```bash
# 2.1: Verify firewall is running
sudo firewall-cmd --state
# Should output: running

# 2.2: View current configuration
sudo firewall-cmd --list-all
```

### Configure Firewall

```bash
# 2.3: Create custom services zone
sudo firewall-cmd --permanent --new-zone=services

# 2.4: Allow connections from anywhere (this is correct and safe)
sudo firewall-cmd --permanent --zone=services --add-source=0.0.0.0/0

# 2.5: Allow SSH (CRITICAL - don't skip this!)
sudo firewall-cmd --permanent --zone=services --add-service=ssh

# 2.6: Allow HTTP and HTTPS (for Caddy later)
sudo firewall-cmd --permanent --zone=services --add-service=http
sudo firewall-cmd --permanent --zone=services --add-service=https

# 2.7: Allow Tailscale VPN port (for later)
sudo firewall-cmd --permanent --zone=services --add-port=41641/udp

# 2.8: Apply changes
sudo firewall-cmd --reload
```

### Verification

```bash
# Verify services zone
sudo firewall-cmd --zone=services --list-all

# Should show:
# - sources: 0.0.0.0/0
# - services: ssh http https
# - ports: 41641/udp
```

**Test your current SSH connection**:
```bash
# Your current terminal should still work (type something)
echo "SSH still works!"
```

✅ **Success**: Firewall configured, SSH still working

---

## Step 3: SSH Key Setup (SAFE METHOD)

### What We're Doing
Set up SSH key authentication WITHOUT disabling password auth yet.

### ⚠️ SAFETY APPROACH
We will:
1. Add your SSH key
2. Test key authentication works
3. Verify key login works in multiple new terminals
4. ONLY THEN disable password authentication

**We will NOT disable password auth until Step 3D - after thorough testing!**

---

### Step 3A: Prepare Your Local Machine

**On your LOCAL machine** (laptop/desktop):

```bash
# Check if you have an SSH key
ls -la ~/.ssh/id_*.pub

# If you see files, you have keys. Note the name (id_ed25519.pub or id_rsa.pub)
```

**If no keys exist**, create one:
```bash
ssh-keygen -t ed25519 -C "rainreport.dev@proton.me"
# Press Enter for default location
# Choose a passphrase (optional but recommended)
```

```bash
# Display your public key
cat ~/.ssh/id_ed25519.pub
# Or: cat ~/.ssh/id_rsa.pub

# Copy the ENTIRE line (starts with ssh-ed25519 or ssh-rsa)
```

---

### Step 3B: Add Key to Server (Manual Method)

**Back on the SERVER**:

```bash
# 3.1: Verify your username
whoami
# Remember this! (probably 'fedora' or 'rainreport')

# 3.2: Create .ssh directory
mkdir -p ~/.ssh
chmod 700 ~/.ssh

# 3.3: Add your public key
nano ~/.ssh/authorized_keys
```

**Paste your public key**:
- Paste the ENTIRE line you copied from local machine
- Should be one long line starting with `ssh-ed25519` or `ssh-rsa`
- Save: `Ctrl+X`, `Y`, `Enter`

```bash
# 3.4: Set correct permissions
chmod 600 ~/.ssh/authorized_keys

# 3.5: Verify file contents
cat ~/.ssh/authorized_keys
# Should show your public key

# 3.6: Check permissions
ls -la ~/.ssh/
# Should show:
# drwx------ (700) for .ssh
# -rw------- (600) for authorized_keys
```

---

### Step 3C: Test SSH Key Authentication (CRITICAL!)

**⚠️ DO NOT CLOSE YOUR CURRENT TERMINAL ⚠️**

**On your LOCAL machine**, open a **NEW terminal window**:

```bash
# Test SSH with key authentication
ssh YOUR_USERNAME@YOUR_SERVER_IP

# Should connect WITHOUT asking for password!
# (might ask for SSH key passphrase if you set one)
```

**Did it work?**

- ✅ **YES, connected without password**: Continue to verification below
- ❌ **NO, still asking for password**: See troubleshooting section, DO NOT CONTINUE

**If it worked, test again** in another new terminal:

```bash
# Second test in another new terminal
ssh YOUR_USERNAME@YOUR_SERVER_IP

# Should also connect without password
```

**Keep these test terminals open!**

---

### Verification Checklist

Run these on the **SERVER** (in your original terminal):

```bash
# 1. Verify authorized_keys exists and has content
[ -f ~/.ssh/authorized_keys ] && echo "✓ File exists" || echo "✗ File missing"

# 2. Verify permissions
[ "$(stat -c %a ~/.ssh)" = "700" ] && echo "✓ .ssh permissions correct" || echo "✗ Wrong permissions"
[ "$(stat -c %a ~/.ssh/authorized_keys)" = "600" ] && echo "✓ authorized_keys permissions correct" || echo "✗ Wrong permissions"

# 3. Verify key is present
[ "$(wc -l < ~/.ssh/authorized_keys)" -gt 0 ] && echo "✓ Key present" || echo "✗ No key found"

# 4. Count how many SSH sessions you have open
who | grep $(whoami) | wc -l
# Should be at least 3 (original + 2 test terminals)
```

**All checks passed AND you have 3+ terminals open?**
- ✅ **YES**: Continue to Step 3D
- ❌ **NO**: Fix issues before continuing

---

### Step 3D: SSH Troubleshooting (If Key Auth Didn't Work)

**If SSH key authentication didn't work, try these:**

#### Fix 1: Remove Cloud-Init SSH Restrictions

```bash
# On server, check for cloud-init config blocking you
cat /etc/ssh/sshd_config.d/50-cloud-init.conf

# If it shows "PasswordAuthentication no", remove it:
sudo mv /etc/ssh/sshd_config.d/50-cloud-init.conf /etc/ssh/sshd_config.d/50-cloud-init.conf.disabled

# Restart SSH
sudo systemctl restart sshd

# Test again from local machine
```

#### Fix 2: Enable PubkeyAuthentication

```bash
# On server, create override file
sudo nano /etc/ssh/sshd_config.d/60-enable-keys.conf
```

**Add this content**:
```
# Enable public key authentication
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
```

**Save**: `Ctrl+X`, `Y`, `Enter`

```bash
# Restart SSH
sudo systemctl restart sshd

# Test again from local machine
```

#### Fix 3: Check SSH Logs

```bash
# On server, watch SSH logs
sudo tail -20 /var/log/secure | grep -i sshd

# Look for errors like:
# - "Authentication refused: bad ownership"
# - "Could not open authorized keys"
# - Permission denied errors
```

**Common fixes**:
```bash
# Fix ownership
sudo chown -R $(whoami):$(whoami) ~/.ssh

# Fix SELinux contexts
sudo restorecon -R -v ~/.ssh

# Restart SSH
sudo systemctl restart sshd
```

**After fixing, test again from local machine. DO NOT CONTINUE until key auth works!**

---

### Step 3E: Harden SSH Config (ONLY AFTER KEY AUTH WORKS!)

**⚠️ CRITICAL SAFETY CHECK ⚠️**

Before proceeding, verify:
- [ ] You have at least 3 SSH terminals open
- [ ] At least 2 of them connected using KEY authentication (no password)
- [ ] You tested key auth in multiple new terminals
- [ ] All test connections worked

**ALL boxes checked? Continue. Any unchecked? DO NOT CONTINUE!**

---

```bash
# 3.7: Backup SSH config
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup.$(date +%Y%m%d-%H%M%S)

# Verify backup exists
ls -la /etc/ssh/sshd_config.backup.*
```

```bash
# 3.8: Create SSH hardening config
sudo nano /etc/ssh/sshd_config.d/99-hardening.conf
```

**Add this content**:
```conf
# SSH Hardening Configuration

# Disable root login
PermitRootLogin no

# Disable password authentication (KEY AUTH MUST WORK FIRST!)
PasswordAuthentication no

# Enable public key authentication
PubkeyAuthentication yes

# Disable empty passwords
PermitEmptyPasswords no

# Limit authentication attempts
MaxAuthTries 3

# Set client timeout (5 minutes)
ClientAliveInterval 300
ClientAliveCountMax 2

# Disable unused authentication
ChallengeResponseAuthentication no
KerberosAuthentication no
GSSAPIAuthentication no
```

**Save**: `Ctrl+X`, `Y`, `Enter`

```bash
# 3.9: Test SSH config for syntax errors
sudo sshd -t

# NO output = config is valid
# If errors shown, fix them before continuing!
```

---

### ⚠️ CRITICAL TEST BEFORE RESTARTING SSH ⚠️

**DO THIS IN A NEW TERMINAL ON LOCAL MACHINE**:

```bash
# Test connection BEFORE restarting SSH
ssh YOUR_USERNAME@YOUR_SERVER_IP

# Should connect with key (no password)
```

**Did it work?**
- ✅ **YES**: Continue below
- ❌ **NO**: DO NOT restart SSH! Fix the issue first!

---

### Restart SSH (Point of No Return)

**Only do this if key auth is working in multiple terminals!**

```bash
# 3.10: Restart SSH service
sudo systemctl restart sshd

# Check status
sudo systemctl status sshd
# Should show: Active: active (running)
```

**Your existing terminals will stay connected**, but new connections will require key auth.

---

### Final Verification

**Open a NEW terminal on local machine**:

```bash
# Test 1: Connect with key (should work)
ssh YOUR_USERNAME@YOUR_SERVER_IP
# Should connect!

# Test 2: Try to use password (should fail)
ssh -o PreferredAuthentications=password YOUR_USERNAME@YOUR_SERVER_IP
# Should show: Permission denied
```

✅ **Success**: SSH hardened, key-only authentication working!

**If Test 1 failed**: You still have your other terminals open. Use them to:
1. Check SSH service: `sudo systemctl status sshd`
2. Restore backup: `sudo cp /etc/ssh/sshd_config.backup.* /etc/ssh/sshd_config`
3. Remove hardening config: `sudo rm /etc/ssh/sshd_config.d/99-hardening.conf`
4. Restart SSH: `sudo systemctl restart sshd`
5. Debug and try again

---

## Step 4: Install CrowdSec

### What We're Doing
Installing CrowdSec for intrusion detection and prevention.

### Installation

```bash
# 4.1: Add CrowdSec repository
curl -s https://packagecloud.io/install/repositories/crowdsec/crowdsec/script.rpm.sh | sudo bash

# 4.2: Install CrowdSec
sudo dnf install -y crowdsec

# 4.3: Enable and start
sudo systemctl enable --now crowdsec
```

### Install Firewall Bouncer

```bash
# 4.4: Install bouncer
sudo dnf install -y crowdsec-firewall-bouncer-iptables

# 4.5: Enable and start bouncer
sudo systemctl enable --now crowdsec-firewall-bouncer
```

### Configure Collections

```bash
# 4.6: Install protection collections
sudo cscli collections install crowdsecurity/linux
sudo cscli collections install crowdsecurity/sshd
sudo cscli collections install crowdsecurity/http-cve

# 4.7: Reload CrowdSec
sudo systemctl reload crowdsec
```

### Verification

```bash
# Check services running
systemctl is-active crowdsec crowdsec-firewall-bouncer
# Both should output: active

# View installed collections
sudo cscli collections list

# View metrics
sudo cscli metrics
```

✅ **Success**: CrowdSec installed and protecting your server

---

## Step 5: SELinux (SAFE METHOD)

### What We're Doing
Enabling SELinux WITHOUT breaking SSH.

### ⚠️ CRITICAL: This section caused your lockouts before!

**New approach**: Fix all file contexts BEFORE enabling enforcing mode.

---

### Check SELinux Status

```bash
# 5.1: Check current status
getenforce

# Outputs: Enforcing, Permissive, or Disabled
```

**If already Enforcing**: Skip to verification section.

**If Disabled or Permissive**: Continue below.

---

### Step 5A: Fix All File Contexts FIRST

**This prevents SSH lockout!**

```bash
# 5.2: Fix SSH contexts (CRITICAL!)
sudo restorecon -R -v /root/.ssh
sudo restorecon -R -v /home/*/.ssh
sudo restorecon -R -v /etc/ssh
sudo restorecon -v /usr/sbin/sshd

# Expected: Shows files being relabeled with correct contexts
```

```bash
# 5.3: Verify your authorized_keys has correct context
ls -lZ ~/.ssh/authorized_keys

# Should show: ssh_home_t
# Example: unconfined_u:object_r:ssh_home_t:s0
```

**If NOT showing ssh_home_t**:
```bash
sudo restorecon -v ~/.ssh/authorized_keys
# Then check again
```

---

### Step 5B: Enable SELinux Permissive Mode (Safe)

```bash
# 5.4: Set to permissive (non-blocking) mode
sudo setenforce 0

# 5.5: Verify
getenforce
# Should output: Permissive
```

```bash
# 5.6: Edit config for permanent setting
sudo nano /etc/selinux/config
```

**Ensure these values**:
```
SELINUX=enforcing
SELINUXTYPE=targeted
```

**Save**: `Ctrl+X`, `Y`, `Enter`

---

### Step 5C: Test SSH in Permissive Mode

**⚠️ CRITICAL TEST - Do NOT skip this!**

**On local machine, open NEW terminal**:

```bash
# Test SSH still works with SELinux in permissive
ssh YOUR_USERNAME@YOUR_SERVER_IP

# Should connect successfully
```

**Did it work?**
- ✅ **YES**: Continue to Step 5D
- ❌ **NO**: SELinux is blocking SSH. Fix contexts and test again!

---

### Step 5D: Check for SELinux Denials

```bash
# 5.7: Check for any SSH-related denials
sudo ausearch -m avc -ts recent | grep sshd

# If no output: Great! No denials
# If denials shown: Need to fix them before enforcing
```

**If denials found**:
```bash
# Generate policy to allow
sudo ausearch -m avc -ts recent | audit2allow -M ssh-policy
sudo semodule -i ssh-policy.pp

# Test SSH again
```

---

### Step 5E: Enable Enforcing Mode

**Only after SSH tested working in permissive mode!**

```bash
# 5.8: Enable enforcing mode
sudo setenforce 1

# 5.9: Verify
getenforce
# Should output: Enforcing
```

---

### ⚠️ CRITICAL TEST IN ENFORCING MODE ⚠️

**On local machine, open ANOTHER new terminal**:

```bash
# Test SSH works in enforcing mode
ssh YOUR_USERNAME@YOUR_SERVER_IP

# Should connect successfully
```

**Did it work?**
- ✅ **YES**: SELinux properly configured! Continue to verification
- ❌ **NO**: Revert to permissive immediately!

**If SSH failed**:
```bash
# On server (in existing terminal)
sudo setenforce 0
# SSH should work again

# Fix contexts again
sudo restorecon -R -v ~/.ssh
sudo restorecon -R -v /etc/ssh

# Test again, then re-enable enforcing
```

---

### Step 5F: Schedule Full Relabel (Optional but Recommended)

```bash
# 5.10: Schedule full filesystem relabel on next boot
sudo touch /.autorelabel

# This ensures ALL files have correct contexts
# Next boot will take 5-10 minutes longer
```

---

### Verification

```bash
# Check SELinux is enforcing
getenforce
# Should output: Enforcing

# Check SSH contexts
ls -lZ ~/.ssh/authorized_keys
# Should show: ssh_home_t

# Test SSH from new terminal (local machine)
ssh YOUR_USERNAME@YOUR_SERVER_IP
# Should connect!

# Check for denials
sudo ausearch -m avc -ts recent
# Should be none or very few
```

✅ **Success**: SELinux enabled and enforcing WITHOUT breaking SSH!

---

## Step 6: Kernel Hardening

### What We're Doing
Configuring kernel security parameters.

```bash
# 6.1: Create security configuration
sudo nano /etc/sysctl.d/99-security.conf
```

**Add this content**:
```conf
# Network security
net.ipv4.ip_forward = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.all.log_martians = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1
net.ipv4.conf.all.rp_filter = 1

# Kernel security
kernel.randomize_va_space = 2
kernel.kptr_restrict = 2
kernel.dmesg_restrict = 1
kernel.yama.ptrace_scope = 2

# Disable IPv6 (optional)
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
```

**Save**: `Ctrl+X`, `Y`, `Enter`

```bash
# 6.2: Apply settings
sudo sysctl -p /etc/sysctl.d/99-security.conf

# 6.3: Verify
sysctl net.ipv4.tcp_syncookies
# Should output: net.ipv4.tcp_syncookies = 1
```

✅ **Success**: Kernel hardened

---

## Step 7: Secure Shared Memory

```bash
# 7.1: Backup fstab
sudo cp /etc/fstab /etc/fstab.backup.$(date +%Y%m%d)

# 7.2: Edit fstab
sudo nano /etc/fstab
```

**Add at the end**:
```
tmpfs /run/shm tmpfs defaults,noexec,nodev,nosuid 0 0
```

**Save**: `Ctrl+X`, `Y`, `Enter`

```bash
# 7.3: Remount
sudo mount -o remount /run/shm

# 7.4: Verify
mount | grep /run/shm
# Should show: noexec,nodev,nosuid
```

✅ **Success**: Shared memory secured

---

## Step 8: Install Security Tools

```bash
# 8.1: Install tools
sudo dnf install -y lynis aide rkhunter

# 8.2: Initialize AIDE
sudo aide --init
sudo mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz

# 8.3: Update RKHunter
sudo rkhunter --update

# 8.4: Run Lynis audit
sudo lynis audit system --quick
```

✅ **Success**: Security tools installed

---

## Step 9: Audit Logging

```bash
# 9.1: Install auditd
sudo dnf install -y audit

# 9.2: Enable auditd
sudo systemctl enable --now auditd

# 9.3: Create audit rules
sudo nano /etc/audit/rules.d/audit.rules
```

**Add this content**:
```conf
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
```

**Save**: `Ctrl+X`, `Y`, `Enter`

```bash
# 9.4: Load rules
sudo augenrules --load

# 9.5: Restart auditd
sudo service auditd restart

# 9.6: Verify
sudo auditctl -l
# Should show all rules
```

✅ **Success**: Audit logging enabled

---

## Step 10: System Limits

```bash
# 10.1: Edit limits
sudo nano /etc/security/limits.conf
```

**Add at the end**:
```conf
# Limit core dumps
* hard core 0

# Limit processes
* soft nproc 1024
* hard nproc 2048

# Limit open files
* soft nofile 65536
* hard nofile 65536
```

**Save**: `Ctrl+X`, `Y`, `Enter`

✅ **Success**: System limits configured

---

## Final Verification Script

```bash
# Create verification script
cat > ~/security-check.sh << 'EOF'
#!/bin/bash
echo "=== Security Hardening Verification ==="
echo ""

echo "1. System Updates:"
systemctl is-active --quiet dnf-automatic.timer && echo "✓ Automatic updates enabled" || echo "✗ Automatic updates disabled"

echo "2. Firewall:"
systemctl is-active --quiet firewalld && echo "✓ Firewall active" || echo "✗ Firewall inactive"

echo "3. SSH:"
sudo grep -q "^PasswordAuthentication no" /etc/ssh/sshd_config.d/99-hardening.conf && echo "✓ Password auth disabled" || echo "✗ Password auth enabled"

echo "4. CrowdSec:"
systemctl is-active --quiet crowdsec && echo "✓ CrowdSec active" || echo "✗ CrowdSec inactive"
systemctl is-active --quiet crowdsec-firewall-bouncer && echo "✓ Bouncer active" || echo "✗ Bouncer inactive"

echo "5. SELinux:"
[ "$(getenforce)" = "Enforcing" ] && echo "✓ SELinux enforcing" || echo "✗ SELinux not enforcing"

echo "6. Kernel:"
[ "$(sysctl -n net.ipv4.tcp_syncookies)" = "1" ] && echo "✓ SYN cookies enabled" || echo "✗ SYN cookies disabled"

echo "7. Auditd:"
systemctl is-active --quiet auditd && echo "✓ Auditd active" || echo "✗ Auditd inactive"

echo "8. Security Tools:"
command -v lynis > /dev/null && echo "✓ Lynis installed" || echo "✗ Lynis not installed"

echo ""
echo "=== Verification Complete ==="
EOF

chmod +x ~/security-check.sh
```

```bash
# Run verification
~/security-check.sh
```

**All items should show ✓**

---

## Emergency Recovery

**If you get locked out of SSH**:

1. **Use console access** (IPMI, hosting provider web console)

2. **Check SSH service**:
   ```bash
   sudo systemctl status sshd
   ```

3. **Restore SSH config**:
   ```bash
   sudo rm /etc/ssh/sshd_config.d/99-hardening.conf
   sudo systemctl restart sshd
   ```

4. **If SELinux issue**:
   ```bash
   sudo setenforce 0
   sudo restorecon -R -v ~/.ssh
   sudo setenforce 1
   ```

5. **If still broken**:
   ```bash
   sudo touch /.autorelabel
   sudo reboot
   # Wait 10 minutes for relabeling
   ```

---

## Summary

✅ **System Hardening**: Updated, automatic updates enabled
✅ **Network Security**: Firewall configured, CrowdSec protecting
✅ **Access Control**: SSH hardened with key-only authentication
✅ **SELinux**: Enabled and enforcing WITHOUT breaking SSH
✅ **Monitoring**: Audit logging, security tools installed

**You should have at least 3 SSH terminals open throughout this entire process!**

---

## Next Steps

1. Review security check output: `~/security-check.sh`
2. Test SSH from multiple devices
3. Proceed to VPN setup guide

**IMPORTANT**: Before closing all terminals, open one more NEW terminal and test SSH works:
```bash
ssh YOUR_USERNAME@YOUR_SERVER_IP
```

Only when this works should you consider the hardening complete!
