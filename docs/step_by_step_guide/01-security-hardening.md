# Step-by-Step Security Hardening Guide

**Estimated Time**: 2-3 hours
**Difficulty**: Intermediate
**Prerequisites**: Root access to Fedora 43 Server

## ⚠️ Important Safety Notes

1. **Always keep a second terminal open** with an active SSH session while making changes
2. **Test SSH access** in a new terminal before closing existing sessions
3. **Have physical/console access** available in case you lock yourself out
4. **Take snapshots/backups** before making system changes if possible

## Step 1: System Update and Package Management

### What We're Doing
Updating all packages to latest versions and configuring automatic security updates.

### Why It Matters
- Patches known vulnerabilities
- Fixes security issues
- Ensures system stability

### Commands

```bash
# 1.1: Update all packages
sudo dnf update -y
```

**Expected Output**: List of packages being updated, then "Complete!"

### Verification

```bash
# Check for remaining updates
sudo dnf check-update

# Should output: nothing or "Last metadata expiration check"
```

✅ **Success Criteria**: No pending updates listed

---

### Configure Automatic Security Updates

```bash
# 1.2: Install automatic update tool
sudo dnf install -y dnf-automatic
```

**Expected Output**: Package installation confirmation

```bash
# 1.3: Configure automatic updates
sudo nano /etc/dnf/automatic.conf
```

**Change these lines**:
```ini
[commands]
upgrade_type = security          # Only security updates (was: default)
apply_updates = yes              # Auto-apply (was: no)
download_updates = yes           # Auto-download (should be yes)
```

**Save and exit**: `Ctrl+X`, then `Y`, then `Enter`

```bash
# 1.4: Enable automatic updates
sudo systemctl enable --now dnf-automatic.timer
```

**Expected Output**: Created symlink message

### Verification

```bash
# Check timer status
sudo systemctl status dnf-automatic.timer

# Should show: Active: active (waiting)
# Next scheduled run time shown

# List all timers
sudo systemctl list-timers | grep dnf-automatic
```

✅ **Success Criteria**: Timer is active and scheduled

---

## Step 2: Firewall Configuration

### What We're Doing
Configuring firewalld to only allow necessary traffic and block everything else by default.

### Why It Matters
- Reduces attack surface
- Blocks unauthorized access
- Controls what services are exposed

### Check Current Status

```bash
# 2.1: Check firewall is running
sudo firewall-cmd --state

# Should output: running
```

```bash
# 2.2: Check current configuration
sudo firewall-cmd --list-all

# Note the current zone and services
```

### Configure Firewall Zones

```bash
# 2.3: Create custom zone for services
sudo firewall-cmd --permanent --new-zone=services
```

**Expected Output**: success

```bash
# 2.4: Add your server's IP range to services zone
# Replace 0.0.0.0/0 with specific IP range if you know it
sudo firewall-cmd --permanent --zone=services --add-source=0.0.0.0/0
```

**Expected Output**: success

```bash
# 2.5: Allow essential services
sudo firewall-cmd --permanent --zone=services --add-service=ssh
sudo firewall-cmd --permanent --zone=services --add-service=http
sudo firewall-cmd --permanent --zone=services --add-service=https
```

**Expected Output**: success (for each command)

```bash
# 2.6: Allow Tailscale VPN port (will set up later)
sudo firewall-cmd --permanent --zone=services --add-port=41641/udp
```

**Expected Output**: success

```bash
# 2.7: Reload firewall to apply changes
sudo firewall-cmd --reload
```

**Expected Output**: success

### Verification

```bash
# Check services zone configuration
sudo firewall-cmd --zone=services --list-all

# Should show:
# - sources: 0.0.0.0/0
# - services: ssh http https
# - ports: 41641/udp
```

```bash
# Test SSH is still allowed
# In a NEW terminal, try to SSH to your server
ssh your-username@your-server-ip

# Should connect successfully
```

✅ **Success Criteria**:
- Services zone properly configured
- SSH still works from new terminal
- HTTP, HTTPS ports open

**📝 Note**: We'll restrict SSH to VPN-only after setting up Tailscale in the next guide.

---

## Step 3: SSH Hardening

### What We're Doing
Securing SSH by using key-based authentication only, disabling root login, and using strong encryption.

### Why It Matters
- Prevents brute force attacks
- Eliminates password vulnerabilities
- Uses strong cryptographic algorithms

### 3A: Create Non-Root User (if not already done)

**Check if you're using root**:
```bash
whoami

# If output is "root", create a non-root user
```

If you're already using a non-root user (like `rainreport`), skip to 3B.

```bash
# 3.1: Create new user (only if currently root)
sudo useradd -m -G wheel antoine

# 3.2: Set password
sudo passwd antoine

# Enter a strong password twice
```

```bash
# 3.3: Test sudo access
su - antoine
sudo whoami

# Should output: root
# Then exit back to your original user
exit
```

### 3B: Set Up SSH Key Authentication

**On your LOCAL machine** (laptop/desktop):

```bash
# 3.4: Check if you have an SSH key
ls -la ~/.ssh/id_ed25519.pub

# If file exists, skip to step 3.6
# If "No such file or directory", create one:
```

```bash
# 3.5: Generate SSH key (if needed)
ssh-keygen -t ed25519 -C "rainreport.dev@proton.me"

# Press Enter to accept default location
# Enter a passphrase (recommended) or leave empty
```

**Expected Output**: Key saved message, fingerprint displayed

```bash
# 3.6: Copy public key to server
ssh-copy-id rainreport@YOUR_SERVER_IP

# Replace YOUR_SERVER_IP with your actual server IP
# Enter your current password when prompted
```

**Expected Output**: "Number of key(s) added: 1"

### Verification

```bash
# 3.7: Test SSH with key (from local machine)
ssh rainreport@YOUR_SERVER_IP

# Should connect WITHOUT asking for password
```

✅ **Success Criteria**: SSH connection works with key, no password needed

---

### 3C: Harden SSH Configuration

**Back on the SERVER**:

```bash
# 3.8: Backup current SSH config
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup.$(date +%Y%m%d)

# Verify backup created
ls -la /etc/ssh/sshd_config.backup.*
```

```bash
# 3.9: Edit SSH configuration
sudo nano /etc/ssh/sshd_config
```

**Find and modify these lines** (use `Ctrl+W` to search in nano):

```conf
# Disable root login
PermitRootLogin no

# Use SSH keys only
PasswordAuthentication no
PubkeyAuthentication yes

# Disable empty passwords
PermitEmptyPasswords no

# Disable X11 forwarding (if not needed)
X11Forwarding no

# Limit authentication attempts
MaxAuthTries 3

# Enable strict mode
StrictModes yes

# Set client timeout (5 minutes of inactivity)
ClientAliveInterval 300
ClientAliveCountMax 2

# Disable unused authentication
ChallengeResponseAuthentication no
KerberosAuthentication no
GSSAPIAuthentication no
```

**Save**: `Ctrl+X`, `Y`, `Enter`

```bash
# 3.10: Test configuration for syntax errors
sudo sshd -t

# No output = configuration is valid
# If errors shown, fix them before proceeding
```

### ⚠️ CRITICAL: Test Before Restarting SSH

**IMPORTANT**: Do this in a NEW terminal while keeping your current session open!

```bash
# 3.11: In a NEW terminal window, test SSH with key
ssh rainreport@YOUR_SERVER_IP

# Should work with key-based auth
```

If the new connection works:

```bash
# 3.12: In your original terminal, restart SSH
sudo systemctl restart sshd

# Check status
sudo systemctl status sshd

# Should show: Active: active (running)
```

### Verification

```bash
# 3.13: Test SSH again in another NEW terminal
ssh rainreport@YOUR_SERVER_IP

# Should connect with key only
```

```bash
# 3.14: Try to SSH as root (should fail)
ssh root@YOUR_SERVER_IP

# Should show: Permission denied
```

```bash
# 3.15: Verify configuration
sudo sshd -T | grep -E 'permitrootlogin|passwordauthentication|pubkeyauthentication'

# Should show:
# permitrootlogin no
# passwordauthentication no
# pubkeyauthentication yes
```

✅ **Success Criteria**:
- SSH works with key
- Password authentication disabled
- Root login disabled
- At least 2 terminals connected successfully

---

## Step 4: Install and Configure Fail2Ban

### What We're Doing
Installing Fail2Ban to automatically ban IPs with repeated failed login attempts.

### Why It Matters
- Prevents brute force attacks
- Automatically blocks malicious IPs
- Reduces server load from attack attempts

### Installation

```bash
# 4.1: Install Fail2Ban
sudo dnf install -y fail2ban fail2ban-firewalld

# Expected Output: Package installation confirmation
```

```bash
# 4.2: Enable and start Fail2Ban
sudo systemctl enable --now fail2ban
```

**Expected Output**: Created symlink message

### Configuration

```bash
# 4.3: Create local configuration
sudo nano /etc/fail2ban/jail.local
```

**Add this content**:

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

# Email notifications (optional, configure later)
# destemail = rainreport.dev@proton.me
# sendername = Fail2Ban
# action = %(action_mwl)s

[sshd]
enabled = true
port = 22
logpath = /var/log/secure
maxretry = 3
```

**Save**: `Ctrl+X`, `Y`, `Enter`

```bash
# 4.4: Restart Fail2Ban to apply config
sudo systemctl restart fail2ban
```

### Verification

```bash
# 4.5: Check Fail2Ban status
sudo systemctl status fail2ban

# Should show: Active: active (running)
```

```bash
# 4.6: Check SSH jail is active
sudo fail2ban-client status

# Should show: sshd in jail list
```

```bash
# 4.7: Check SSH jail details
sudo fail2ban-client status sshd

# Should show:
# - Currently banned: 0 (initially)
# - Total banned: 0 (initially)
```

```bash
# 4.8: Monitor Fail2Ban logs
sudo tail -f /var/log/fail2ban.log

# Press Ctrl+C to stop monitoring
# Should see: "Jail 'sshd' started"
```

### Test Fail2Ban (Optional)

**From another machine** (or use a VM):

```bash
# Try to SSH with wrong password multiple times
# (This will fail since we disabled password auth, but Fail2Ban monitors attempts)

# After 3 failed attempts, IP should be banned
```

```bash
# On server, check banned IPs
sudo fail2ban-client status sshd

# Should show the test IP in banned list
```

```bash
# Unban test IP
sudo fail2ban-client set sshd unbanip TEST_IP_ADDRESS
```

✅ **Success Criteria**:
- Fail2Ban running
- SSH jail enabled
- Can detect and ban IPs with failed attempts

---

## Step 5: Verify SELinux

### What We're Doing
Verifying SELinux is enabled and in enforcing mode.

### Why It Matters
- Mandatory access control
- Prevents privilege escalation
- Limits damage from compromised services

### Check SELinux Status

```bash
# 5.1: Check SELinux status
sestatus

# Expected output should show:
# SELinux status: enabled
# Current mode: enforcing
```

```bash
# 5.2: Check current mode
getenforce

# Should output: Enforcing
```

### If SELinux is Disabled or Permissive

```bash
# 5.3: Check config file
cat /etc/selinux/config | grep ^SELINUX=

# Should show: SELINUX=enforcing
```

If not enforcing:

```bash
# 5.4: Edit config
sudo nano /etc/selinux/config
```

**Set**:
```
SELINUX=enforcing
SELINUXTYPE=targeted
```

**Save and reboot**:
```bash
sudo reboot
```

**After reboot, verify**:
```bash
sestatus
# Should now show: enforcing
```

### Verification

```bash
# 5.5: Verify SELinux is protecting services
sudo ps -eZ | grep sshd

# Should show SELinux contexts like: system_u:system_r:sshd_t
```

```bash
# 5.6: Check for recent denials
sudo ausearch -m avc -ts recent

# If nothing found, that's good
# If denials found, they may need policy adjustments later
```

✅ **Success Criteria**: SELinux enabled and enforcing

---

## Step 6: Kernel Hardening

### What We're Doing
Configuring kernel parameters for better security.

### Why It Matters
- Prevents network-based attacks
- Hardens TCP/IP stack
- Restricts kernel information leakage

### Create Sysctl Configuration

```bash
# 6.1: Create security configuration file
sudo nano /etc/sysctl.d/99-security.conf
```

**Add this content**:

```conf
# Disable IP forwarding (we'll enable for VPN later if needed)
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

# Enable bad error message protection
net.ipv4.icmp_ignore_bogus_error_responses = 1

# Enable reverse path filtering
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# Disable IPv6 (if not using it - optional)
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

**Save**: `Ctrl+X`, `Y`, `Enter`

```bash
# 6.2: Apply settings
sudo sysctl -p /etc/sysctl.d/99-security.conf

# Should show all settings being applied
```

### Verification

```bash
# 6.3: Verify specific settings
sysctl net.ipv4.tcp_syncookies
# Should output: net.ipv4.tcp_syncookies = 1

sysctl kernel.randomize_va_space
# Should output: kernel.randomize_va_space = 2

sysctl net.ipv4.ip_forward
# Should output: net.ipv4.ip_forward = 0
```

```bash
# 6.4: View all custom settings
sudo sysctl -a | grep -E 'tcp_syncookies|randomize_va_space|ip_forward|kptr_restrict'
```

✅ **Success Criteria**: All security settings applied and verified

---

## Step 7: Secure Shared Memory

### What We're Doing
Mounting shared memory with security options to prevent privilege escalation.

### Why It Matters
- Prevents certain types of privilege escalation attacks
- Restricts execution of code from shared memory

### Configuration

```bash
# 7.1: Backup fstab
sudo cp /etc/fstab /etc/fstab.backup.$(date +%Y%m%d)
```

```bash
# 7.2: Edit fstab
sudo nano /etc/fstab
```

**Add this line at the end**:
```
tmpfs /run/shm tmpfs defaults,noexec,nodev,nosuid 0 0
```

**Save**: `Ctrl+X`, `Y`, `Enter`

```bash
# 7.3: Remount shared memory
sudo mount -o remount /run/shm
```

### Verification

```bash
# 7.4: Verify mount options
mount | grep /run/shm

# Should show: noexec,nodev,nosuid in options
```

```bash
# 7.5: Test noexec (should fail)
echo '#!/bin/bash' > /run/shm/test.sh
echo 'echo "test"' >> /run/shm/test.sh
chmod +x /run/shm/test.sh
/run/shm/test.sh

# Should fail with: Permission denied
# Clean up
rm /run/shm/test.sh
```

✅ **Success Criteria**: Shared memory mounted with security options, execution prevented

---

## Step 8: Install Security Audit Tools

### What We're Doing
Installing tools to audit security and detect intrusions.

### Why It Matters
- Identifies security misconfigurations
- Detects rootkits and malware
- Monitors file integrity

### Installation

```bash
# 8.1: Install security tools
sudo dnf install -y lynis aide rkhunter

# Expected Output: Package installation confirmation
```

### 8A: Lynis - Security Auditing

```bash
# 8.2: Run initial Lynis audit
sudo lynis audit system

# This will take a few minutes
# Review the output - shows security score and warnings
```

**Expected Output**:
- Hardening index (e.g., 67/100)
- List of warnings and suggestions
- Color-coded results

```bash
# 8.3: Save audit report
sudo lynis audit system --quick > ~/lynis-initial-audit.txt

# Review report
less ~/lynis-initial-audit.txt
```

### 8B: AIDE - File Integrity Monitoring

```bash
# 8.4: Initialize AIDE database (takes a few minutes)
sudo aide --init

# Expected Output: "AIDE initialized" message
```

```bash
# 8.5: Move database to active location
sudo mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz
```

```bash
# 8.6: Test AIDE check
sudo aide --check

# Expected Output: No differences (since we just initialized)
```

### 8C: RKHunter - Rootkit Detection

```bash
# 8.7: Update RKHunter database
sudo rkhunter --update

# Expected Output: Update complete message
```

```bash
# 8.8: Run initial scan
sudo rkhunter --check --skip-keypress

# This will take several minutes
# Review any warnings (some false positives are common)
```

### Verification

```bash
# 8.9: Verify Lynis installed
lynis --version

# Should show version number
```

```bash
# 8.10: Verify AIDE database exists
ls -lh /var/lib/aide/aide.db.gz

# Should show file size
```

```bash
# 8.11: Verify RKHunter configuration
sudo rkhunter --config-check

# Should complete without errors
```

✅ **Success Criteria**:
- Lynis audit completed with security score
- AIDE database initialized
- RKHunter scan completed

---

## Step 9: Configure Audit Logging

### What We're Doing
Enabling auditd to log security-relevant events.

### Why It Matters
- Tracks authentication events
- Monitors file changes
- Provides forensic evidence

### Installation and Configuration

```bash
# 9.1: Install auditd (usually already installed)
sudo dnf install -y audit

# Expected Output: Already installed or installation confirmation
```

```bash
# 9.2: Enable and start auditd
sudo systemctl enable --now auditd

# Note: auditd requires special restart method
```

```bash
# 9.3: Create audit rules
sudo nano /etc/audit/rules.d/audit.rules
```

**Add this content**:

```conf
## Audit rules for security monitoring

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

# Monitor SSH configuration
-w /etc/ssh/sshd_config -p wa -k sshd_config

# Monitor network configuration
-w /etc/sysconfig/network -p wa -k network_changes

# Monitor firewall changes
-w /etc/firewalld/ -p wa -k firewall_changes

# Monitor file deletions
-a always,exit -F arch=b64 -S unlink -S unlinkat -S rename -S renameat -k delete
```

**Save**: `Ctrl+X`, `Y`, `Enter`

```bash
# 9.4: Load audit rules
sudo augenrules --load
```

```bash
# 9.5: Restart auditd (special command)
sudo service auditd restart
```

### Verification

```bash
# 9.6: Check auditd status
sudo systemctl status auditd

# Should show: Active: active (running)
```

```bash
# 9.7: List active audit rules
sudo auditctl -l

# Should show all rules we added
```

```bash
# 9.8: Test audit logging - change password
echo "test" | sudo passwd --stdin rainreport 2>/dev/null || echo "Test: password command executed"

# Search audit log for this event
sudo ausearch -k passwd_changes -ts recent

# Should show recent password-related events
```

```bash
# 9.9: Generate audit report
sudo aureport --auth

# Shows authentication report
```

✅ **Success Criteria**:
- Auditd running
- Rules loaded and active
- Events being logged

---

## Step 10: Set System Limits

### What We're Doing
Setting resource limits to prevent DoS and system abuse.

### Why It Matters
- Prevents fork bombs
- Limits resource consumption
- Protects against certain attacks

### Configuration

```bash
# 10.1: Edit limits configuration
sudo nano /etc/security/limits.conf
```

**Add at the end** (before the "# End of file" line):

```conf
# Limit core dumps
* hard core 0

# Limit number of processes
* soft nproc 1024
* hard nproc 2048

# Limit open files
* soft nofile 65536
* hard nofile 65536

# Specific user limits (optional)
rainreport soft nproc 4096
rainreport hard nproc 8192
```

**Save**: `Ctrl+X`, `Y`, `Enter`

### Verification

```bash
# 10.2: Check current limits
ulimit -a

# Shows all current limits for your user
```

```bash
# 10.3: Check specific limits
ulimit -n  # Open files
ulimit -u  # Max user processes
ulimit -c  # Core file size (should be 0)
```

**Note**: Limits take effect on new login sessions

```bash
# 10.4: Test in new session (open new SSH session)
ssh rainreport@YOUR_SERVER_IP
ulimit -u

# Should show 1024 (soft limit)
```

✅ **Success Criteria**: Resource limits configured and active

---

## Security Hardening Checklist

Let's verify everything is properly configured:

```bash
# Create verification script
cat > ~/security-check.sh << 'EOF'
#!/bin/bash

echo "=== Security Hardening Verification ==="
echo ""

echo "1. System Updates:"
dnf check-update > /dev/null 2>&1 && echo "✓ System up to date" || echo "✗ Updates available"
systemctl is-active --quiet dnf-automatic.timer && echo "✓ Automatic updates enabled" || echo "✗ Automatic updates disabled"
echo ""

echo "2. Firewall:"
systemctl is-active --quiet firewalld && echo "✓ Firewall active" || echo "✗ Firewall inactive"
sudo firewall-cmd --zone=services --list-services | grep -q ssh && echo "✓ SSH allowed" || echo "✗ SSH not in firewall"
echo ""

echo "3. SSH Hardening:"
sudo sshd -t 2>/dev/null && echo "✓ SSH config valid" || echo "✗ SSH config has errors"
sudo grep -q "^PermitRootLogin no" /etc/ssh/sshd_config && echo "✓ Root login disabled" || echo "✗ Root login enabled"
sudo grep -q "^PasswordAuthentication no" /etc/ssh/sshd_config && echo "✓ Password auth disabled" || echo "✗ Password auth enabled"
echo ""

echo "4. Fail2Ban:"
systemctl is-active --quiet fail2ban && echo "✓ Fail2Ban active" || echo "✗ Fail2Ban inactive"
sudo fail2ban-client status sshd > /dev/null 2>&1 && echo "✓ SSH jail enabled" || echo "✗ SSH jail disabled"
echo ""

echo "5. SELinux:"
[ "$(getenforce)" = "Enforcing" ] && echo "✓ SELinux enforcing" || echo "✗ SELinux not enforcing"
echo ""

echo "6. Kernel Hardening:"
[ "$(sysctl -n net.ipv4.tcp_syncookies)" = "1" ] && echo "✓ SYN cookies enabled" || echo "✗ SYN cookies disabled"
[ "$(sysctl -n kernel.randomize_va_space)" = "2" ] && echo "✓ ASLR enabled" || echo "✗ ASLR disabled"
echo ""

echo "7. Audit Logging:"
systemctl is-active --quiet auditd && echo "✓ Auditd active" || echo "✗ Auditd inactive"
[ "$(sudo auditctl -l | wc -l)" -gt 5 ] && echo "✓ Audit rules configured" || echo "✗ No audit rules"
echo ""

echo "8. Security Tools:"
command -v lynis > /dev/null && echo "✓ Lynis installed" || echo "✗ Lynis not installed"
[ -f /var/lib/aide/aide.db.gz ] && echo "✓ AIDE initialized" || echo "✗ AIDE not initialized"
command -v rkhunter > /dev/null && echo "✓ RKHunter installed" || echo "✗ RKHunter not installed"
echo ""

echo "=== Verification Complete ==="
EOF

chmod +x ~/security-check.sh
```

```bash
# Run verification
~/security-check.sh
```

**Expected Output**: All items should show ✓ (checkmark)

---

## Final Security Test

### Test 1: SSH Security

```bash
# From local machine, verify SSH works with key
ssh rainreport@YOUR_SERVER_IP

# Should connect without password

# Try password auth (should fail)
ssh -o PreferredAuthentications=password rainreport@YOUR_SERVER_IP

# Should show: Permission denied
```

### Test 2: Firewall

```bash
# On server, test firewall rules
sudo firewall-cmd --zone=services --list-all

# Verify:
# - services: ssh http https
# - ports: 41641/udp
```

### Test 3: Service Status

```bash
# Check all security services
sudo systemctl status firewalld fail2ban auditd | grep Active

# All should show: Active: active (running)
```

---

## Summary of What We Accomplished

✅ **System Hardening**:
- System fully updated
- Automatic security updates enabled

✅ **Network Security**:
- Firewall configured with restrictive rules
- Only essential ports open
- Fail2Ban protecting against brute force

✅ **Access Control**:
- SSH hardened (key-only, no root)
- SELinux enforcing
- Strong kernel security parameters

✅ **Monitoring & Auditing**:
- Audit logging enabled
- File integrity monitoring (AIDE)
- Security scanning tools (Lynis, RKHunter)

✅ **Resource Protection**:
- System limits configured
- Shared memory secured

---

## Next Steps

1. **Review Lynis audit report**: `less ~/lynis-initial-audit.txt`
2. **Set up regular security scans**: We'll automate these later
3. **Proceed to VPN setup**: This will further secure SSH access
4. **Document any customizations**: Keep notes of changes

---

## Troubleshooting

### Can't SSH After Changes

**If you get locked out**:
1. Use physical/console access
2. Check SSH service: `sudo systemctl status sshd`
3. Restore backup config: `sudo cp /etc/ssh/sshd_config.backup.* /etc/ssh/sshd_config`
4. Restart SSH: `sudo systemctl restart sshd`

### Firewall Blocking Connections

```bash
# Temporarily disable to test
sudo systemctl stop firewalld

# If that fixes it, review rules
sudo firewall-cmd --list-all

# Re-enable firewall
sudo systemctl start firewalld
```

### SELinux Denials

```bash
# Check for denials
sudo ausearch -m avc -ts recent

# Generate policy (if needed)
sudo ausearch -m avc -ts recent | audit2allow -M mypolicy
sudo semodule -i mypolicy.pp
```

---

## Security Maintenance Schedule

### Daily (Automated)
- Security updates check and apply
- Audit logs collected

### Weekly
- Review Fail2Ban bans
- Check system logs
- Review Lynis warnings

### Monthly
- Full Lynis audit
- RKHunter scan
- AIDE integrity check
- Review user accounts

---

## Important Notes

- **Keep backup SSH sessions open** when making SSH changes
- **Test SSH in new terminal** before closing existing sessions
- **Document all changes** you make beyond this guide
- **Review logs regularly** for suspicious activity
- **Run Lynis monthly** to maintain security posture

---

**🎉 Security Hardening Complete!**

Your server is now significantly more secure. The next step is to set up Tailscale VPN, which will allow us to further restrict SSH and admin access to VPN-only.

**Time to complete**: Review this checklist:
- [ ] All verification steps passed
- [ ] `~/security-check.sh` shows all ✓
- [ ] SSH works with key from new terminal
- [ ] All security services running
- [ ] Lynis audit completed and reviewed

**Ready for next guide**: `02-vpn-setup.md`
