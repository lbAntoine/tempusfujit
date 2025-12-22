# Step-by-Step VPN Setup Guide (Tailscale)

**Difficulty**: Intermediate
**Prerequisites**:

- Completed guide 01 (Security Hardening)
- Fedora 43 Server with SSH working
- User with sudo privileges (fedora)

---

## What You'll Learn

This guide teaches you how to set up a VPN (Virtual Private Network) using Tailscale, which creates a secure private network between your devices. You'll learn to:

1. **Install and configure Tailscale VPN**
2. **Understand the difference between VPN and public internet access**
3. **Configure your firewall to control what's accessible from where**
4. **Test your VPN connection works correctly**

### Why Use a VPN?

Think of your server like a house:
- **Without VPN**: Every service is like a door facing the street - anyone can try to access it
- **With VPN**: You create a private tunnel only you can access. Services can be hidden from the public internet and only accessible through your VPN

**Example use cases**:
- Keep SSH accessible only via VPN (more secure than exposing to internet)
- Run admin panels that only you can access
- Access private services without exposing them publicly

---

## ⚠️ Safety Rules

1. **Keep your current SSH connection open** until VPN is working
2. **Don't block SSH from public internet until VPN is tested**
3. **Test VPN connection before restricting access**
4. **Save your Tailscale admin console URL** (to manage devices)

---

## Current Working State

Verify your starting point:

```bash
# 1. SSH is working
echo "SSH working - you're connected!"

# 2. Firewall is active
sudo firewall-cmd --state
# Output: running

# 3. Check what's currently allowed
sudo firewall-cmd --list-all
```

---

## Step 1: Understanding Tailscale

**What is Tailscale?**
- Creates a secure mesh VPN between your devices
- Each device gets a Tailscale IP (like 100.x.x.x)
- Uses WireGuard protocol (fast and secure)
- Works through NAT/firewalls automatically
- Free for personal use (up to 100 devices)

**How it works**:
1. Install Tailscale on your server
2. Install Tailscale on your laptop/phone
3. Both devices get private IPs (100.x.x.x)
4. You can now connect to your server using the Tailscale IP
5. This connection is encrypted and direct (or relayed if needed)

---

## Step 2: Install Tailscale

```bash
# Add Tailscale repository
sudo dnf config-manager --add-repo https://pkgs.tailscale.com/stable/fedora/tailscale.repo

# Install Tailscale
sudo dnf install -y tailscale

# Enable and start Tailscale
sudo systemctl enable --now tailscaled

# Verify service is running
systemctl is-active tailscaled
```

✅ **Expected**: `active`

---

## Step 3: Connect to Tailscale Network

```bash
# Start Tailscale and authenticate
sudo tailscale up

# This will output a URL like:
# To authenticate, visit: https://login.tailscale.com/a/xxxxx
```

**Action required**:
1. Copy the URL from the output
2. Open it in your web browser
3. Sign in with Google, GitHub, or Microsoft account
4. Authorize the device
5. Give your server a name (e.g., "fedora-server")

### Verify Connection

```bash
# Check Tailscale status
sudo tailscale status

# Should show:
# - Your server with a 100.x.x.x IP
# - Status: online

# Get your Tailscale IP
TAILSCALE_IP=$(tailscale ip -4)
echo "Your Tailscale IP: $TAILSCALE_IP"

# Save this IP - you'll use it to connect via VPN
```

✅ **Success**: You see your server listed with a 100.x.x.x IP

---

## Step 4: Test VPN From Your Computer

### Install Tailscale on Your Computer

**On your local machine** (not the server):

- **Linux**: https://tailscale.com/download/linux
- **macOS**: https://tailscale.com/download/macos
- **Windows**: https://tailscale.com/download/windows
- **iOS/Android**: Install from App Store/Play Store

1. Install Tailscale
2. Sign in with the **same account** you used for the server
3. You should see your server appear in the device list

### Test VPN Connection

**On your local machine**:

```bash
# Ping your server's Tailscale IP
ping 100.x.x.x
# (Use the IP you saved earlier)

# Should get responses - this proves VPN is working!
```

### Test SSH Over VPN

**On your local machine**:

```bash
# SSH using Tailscale IP instead of public IP
ssh fedora@100.x.x.x
# (Use your server's Tailscale IP)
```

✅ **Critical**: This SSH connection must work before continuing!

**What just happened?**
- You connected to your server through the VPN tunnel
- Traffic is encrypted and goes through Tailscale network
- No one else can access this IP - it's private to your Tailscale network

---

## Step 5: Understanding Firewall Zones

Before we configure the firewall, let's understand how firewalld works:

### Firewall Zones Explained

**Zones** are like different security levels for different network interfaces:

1. **public zone**: For untrusted networks (the internet)
   - Strict rules
   - Only explicitly allowed services accessible

2. **trusted zone**: For trusted networks (your VPN)
   - Permissive rules
   - All traffic allowed by default

### Current Setup

```bash
# Check current zones
sudo firewall-cmd --get-active-zones

# Should show something like:
# FedoraServer (or public)
#   interfaces: eth0 (or similar)

# Check what's in default zone
sudo firewall-cmd --list-all
```

Currently, everything goes through one zone. We'll create a separation between public and VPN traffic.

---

## Step 6: Configure Firewall for VPN

### Identify Tailscale Interface

```bash
# Find Tailscale network interface
ip addr show | grep tailscale
# Should show: tailscale0

# Verify Tailscale IP on this interface
ip addr show tailscale0
```

### Add Tailscale Interface to Trusted Zone

```bash
# Add tailscale0 to trusted zone
sudo firewall-cmd --permanent --zone=trusted --add-interface=tailscale0

# Reload firewall
sudo firewall-cmd --reload

# Verify
sudo firewall-cmd --get-active-zones
```

**Should now show**:
```
trusted
  interfaces: tailscale0
FedoraServer (or public)
  interfaces: eth0 (or your main interface)
```

### What This Means

- **tailscale0** (VPN): In trusted zone - all traffic allowed
- **eth0** (public internet): In public/FedoraServer zone - only explicitly allowed services

---

## Step 7: Verify VPN Access

### Test VPN Access Still Works

**From your local machine**:

```bash
# SSH via Tailscale IP
ssh fedora@100.x.x.x

# Run a command to verify
sudo firewall-cmd --get-active-zones
```

✅ **Must work before continuing!**

### Test Public Access Still Works

**From your local machine** (using your regular public IP):

```bash
# SSH via public IP
ssh fedora@YOUR_PUBLIC_IP

# Should still work
```

Both methods working? Perfect! Now you have two ways to access your server:
1. Via VPN (Tailscale IP): 100.x.x.x
2. Via public internet: YOUR_PUBLIC_IP

---

## Step 8: Understanding Service Access Control

Now that we have VPN set up, we can decide what's accessible from where:

### Option A: Keep SSH Public (Current State)
- SSH accessible from both VPN and public internet
- Good for: Easy access, but rely on strong auth (keys, CrowdSec)

### Option B: SSH Only via VPN (More Secure)
- SSH only accessible through Tailscale VPN
- Public internet cannot reach SSH at all
- Good for: Maximum security, but requires VPN to always be connected

### Option C: Hybrid Approach
- Keep SSH public for now
- Future services (web admin panels, databases) VPN-only
- Good for: Balance of security and convenience

**For this guide, we'll implement Option C** (hybrid) because it's the most flexible. You can switch to Option B later if desired.

---

## Step 9: Configure Service Access

### Current Services Check

```bash
# What's allowed on public zone?
sudo firewall-cmd --zone=public --list-all

# What's allowed on trusted zone?
sudo firewall-cmd --zone=trusted --list-all
```

### Example: Making a Service VPN-Only

Let's say you'll run a web admin panel on port 8080. To make it VPN-only:

```bash
# Do NOT add to public zone (it's not there, so nothing to do)

# Port 8080 is automatically accessible via VPN (trusted zone allows all)

# Test: Access http://100.x.x.x:8080 - works via VPN
# Test: Access http://PUBLIC_IP:8080 - blocked by firewall
```

### Example: Making a Service Public

For a public website on ports 80/443:

```bash
# Already done in guide 01:
# sudo firewall-cmd --permanent --zone=public --add-service=http
# sudo firewall-cmd --permanent --zone=public --add-service=https

# This allows public access to web services
```

### Verification

Create a test to understand the difference:

```bash
# Check public zone services
echo "=== Public Zone (Internet) ==="
sudo firewall-cmd --zone=public --list-services

# Check trusted zone (VPN)
echo "=== Trusted Zone (VPN) ==="
sudo firewall-cmd --zone=trusted --list-all | grep -E "target|services"
```

**Understanding the output**:
- **Public zone**: Only listed services accessible from internet
- **Trusted zone**: Usually target=ACCEPT (everything allowed)

---

## Step 10: Enable Tailscale Features

### Enable MagicDNS (Optional but Useful)

MagicDNS lets you use hostnames instead of IPs:

```bash
# Enable MagicDNS
sudo tailscale up --accept-dns

# Now you can use: ssh fedora@fedora-server
# Instead of: ssh fedora@100.x.x.x
```

**Test it**:

```bash
# From your local machine:
ping fedora-server
# Should resolve to 100.x.x.x and respond
```

### Check Tailscale Settings

```bash
# View current Tailscale status
sudo tailscale status

# View detailed network info
sudo tailscale netcheck
```

---

## Step 11: Configure Tailscale ACLs (Access Control Lists)

### Understanding Tailscale ACLs

By default, all devices in your Tailscale network can talk to each other. ACLs let you control:
- Which devices can access which devices
- What ports are allowed
- Who has admin access

### View/Edit ACLs

1. Go to https://login.tailscale.com/admin/acls
2. You'll see a JSON configuration file

### Example ACL (Simple)

```json
{
  "acls": [
    // Allow all users to access all devices
    {
      "action": "accept",
      "src": ["*"],
      "dst": ["*:*"]
    }
  ]
}
```

### Example ACL (Restricted)

```json
{
  "acls": [
    // Allow SSH from any device to servers
    {
      "action": "accept",
      "src": ["*"],
      "dst": ["tag:server:22"]
    },
    // Allow HTTP(S) from any device to servers
    {
      "action": "accept",
      "src": ["*"],
      "dst": ["tag:server:80,443"]
    }
  ],
  "tagOwners": {
    "tag:server": ["your-email@example.com"]
  }
}
```

**For now**: Keep the default (allow all) since you only have your own devices. You can restrict later as you add more devices.

---

## Step 12: Advanced - Restricting SSH to VPN Only (with Failback)

**⚠️ WARNING**: Only do this if you're comfortable and have tested VPN access!

If you want to make SSH accessible ONLY via VPN, you should ALWAYS configure a failback IP address first. This ensures you can still access your server if the VPN fails.

### Before You Start

1. **Test VPN SSH works**: `ssh fedora@100.x.x.x` ✅
2. **Have console access** (via hosting provider panel) in case of lockout
3. **Keep 2 SSH sessions open** during this change
4. **Know your static IP** (home/office IP that won't change often)

### Step 12.1: Add Failback IP (Critical Safety Step!)

First, identify your trusted static IP address:

```bash
# From your local machine, check your public IP:
curl ifconfig.me
# Or: curl ipinfo.io/ip

# Example output: 203.0.113.45
```

**Important**: Use a static IP address that you control:
- Your home IP (if static - check with your ISP)
- Your office IP (usually static)
- A bastion host IP (another server you control)
- A cloud VM IP you control

**⚠️ Warning about Dynamic IPs**: If your home/office IP changes (dynamic IP from ISP), you'll need to update the firewall rule. Most residential IPs change infrequently, but keep this in mind. You can:
- Contact your ISP for a static IP (may cost extra)
- Use a cloud VM as a permanent bastion host
- Update the rule when your IP changes: `curl ifconfig.me` then update firewall rule

Now add this IP as a failback on your server:

```bash
# Replace 203.0.113.45 with YOUR trusted IP
TRUSTED_IP="203.0.113.45"

# Add rich rule to allow SSH from your trusted IP
sudo firewall-cmd --permanent --zone=public --add-rich-rule="rule family='ipv4' source address='$TRUSTED_IP' service name='ssh' accept"

# Reload firewall
sudo firewall-cmd --reload

# Verify the rule was added
sudo firewall-cmd --zone=public --list-rich-rules
```

**Should show**:
```
rule family="ipv4" source address="203.0.113.45" service name="ssh" accept
```

### Test Failback IP

**From your trusted IP location**:
```bash
# SSH via public IP should work
ssh fedora@YOUR_PUBLIC_IP
# ✅ Should work even without VPN
```

### Step 12.2: Restrict SSH to VPN Only

Now that you have a failback, restrict SSH:

```bash
# Remove general SSH service from public zone
sudo firewall-cmd --permanent --zone=public --remove-service=ssh

# Reload firewall
sudo firewall-cmd --reload

# Verify
sudo firewall-cmd --zone=public --list-all
```

**Should show**:
- NO `ssh` in services list
- Your rich rule still present (allowing SSH from trusted IP)

### Test Immediately

**Test 1 - VPN access** (should work):
```bash
# From your local machine with Tailscale:
ssh fedora@100.x.x.x
# ✅ Should work
```

**Test 2 - Failback IP access** (should work):
```bash
# From your trusted IP location (disable Tailscale temporarily):
ssh fedora@YOUR_PUBLIC_IP
# ✅ Should work because you're on the trusted IP
```

**Test 3 - Random public IP** (should fail):
```bash
# From a different location (not VPN, not trusted IP):
ssh fedora@YOUR_PUBLIC_IP
# ❌ Should timeout or refuse connection
```

### Understanding What You've Created

Your SSH is now accessible from:
1. ✅ **Any device via Tailscale VPN** (100.x.x.x)
2. ✅ **Your trusted static IP via public internet** (failback)
3. ❌ **Everywhere else** (blocked)

This gives you both security AND a safety net!

### Adding More Trusted IPs

If you need to add more trusted IPs (e.g., office + home):

```bash
# Add another trusted IP
sudo firewall-cmd --permanent --zone=public --add-rich-rule="rule family='ipv4' source address='198.51.100.25' service name='ssh' accept"

# Reload
sudo firewall-cmd --reload

# View all trusted IPs
sudo firewall-cmd --zone=public --list-rich-rules
```

### Removing a Trusted IP

If an IP is no longer trusted:

```bash
# Remove the rule (use exact syntax from list-rich-rules)
sudo firewall-cmd --permanent --zone=public --remove-rich-rule="rule family='ipv4' source address='203.0.113.45' service name='ssh' accept"

# Reload
sudo firewall-cmd --reload
```

### Revert If Needed

If something goes wrong:

```bash
# Re-enable SSH on public zone for everyone
sudo firewall-cmd --permanent --zone=public --add-service=ssh
sudo firewall-cmd --reload

# Or just remove the restrictive rules
sudo firewall-cmd --permanent --zone=public --remove-rich-rule="rule family='ipv4' source address='YOUR_IP' service name='ssh' accept"
sudo firewall-cmd --reload
```

---

## Step 13: Set Up Tailscale Auto-Update

```bash
# Enable Tailscale auto-updates
sudo tailscale set --auto-update

# Verify
sudo tailscale status | grep -i update
```

---

## Final Verification

```bash
# Create VPN verification script
tee ~/vpn-check.sh > /dev/null << 'EOF'
#!/bin/bash
echo "=== VPN Status ==="
echo ""

echo "Tailscale Service:"
systemctl is-active tailscaled && echo "✓ Tailscaled active" || echo "✗ Inactive"
echo ""

echo "Tailscale Connection:"
sudo tailscale status | head -5
echo ""

echo "Tailscale IP:"
tailscale ip -4 || echo "✗ No IP assigned"
echo ""

echo "Firewall Zones:"
sudo firewall-cmd --get-active-zones
echo ""

echo "Public Zone Services:"
sudo firewall-cmd --zone=public --list-services
echo ""

echo "Trusted Zone Interfaces:"
sudo firewall-cmd --zone=trusted --list-interfaces
echo ""
EOF

chmod +x ~/vpn-check.sh
~/vpn-check.sh
```

**Expected output**:
- ✓ Tailscaled active
- Tailscale IP: 100.x.x.x
- Two zones: trusted (tailscale0) and public (eth0)
- Public zone: shows allowed services
- Trusted zone: has tailscale0 interface

---

## Testing Checklist

Run these tests to confirm everything works:

### Test 1: VPN Connectivity
```bash
# From your local machine:
ping $(tailscale ip -4 fedora-server)
```
✅ **Must respond**

### Test 2: SSH via VPN
```bash
# From your local machine:
ssh fedora@100.x.x.x
```
✅ **Must connect**

### Test 3: SSH via Public (if still enabled)
```bash
# From your local machine:
ssh fedora@YOUR_PUBLIC_IP
```
✅ **Should work if SSH still in public zone**
❌ **Should fail if you restricted SSH to VPN only**

### Test 4: MagicDNS (if enabled)
```bash
# From your local machine:
ssh fedora@fedora-server
```
✅ **Should work if MagicDNS enabled**

---

## Understanding Your Setup

### Network Diagram

```
Internet (Public)           Tailscale VPN (Private)
     |                            |
     | (eth0)                     | (tailscale0)
     |                            |
  [Public Zone]              [Trusted Zone]
     |                            |
  - HTTP/HTTPS              - All traffic allowed
  - SSH (optional)          - SSH always works
  - Failback IPs (SSH)      - Full access to all ports
  - Limited access
     |                            |
     +------------+---------------+
                  |
            [Your Server]
```

**With Failback IP configured**:
- Random IP → SSH = ❌ Blocked
- Trusted IP → SSH = ✅ Allowed (failback)
- Any device via VPN → SSH = ✅ Allowed

### What You've Achieved

1. **VPN tunnel**: Encrypted connection to your server
2. **Two access paths**: VPN (private) and public internet
3. **Flexible security**: Choose what's public vs VPN-only
4. **Easy management**: Tailscale admin console for device management

---

## Troubleshooting

### VPN Not Connecting

```bash
# Check Tailscale service
sudo systemctl status tailscaled

# Check Tailscale status
sudo tailscale status

# Check logs
sudo journalctl -u tailscaled -n 50

# Try reconnecting
sudo tailscale down
sudo tailscale up
```

### Can't SSH via VPN

```bash
# Check Tailscale IP
tailscale ip -4

# Check firewall has tailscale0 in trusted zone
sudo firewall-cmd --zone=trusted --list-interfaces

# Should show: tailscale0
```

### Locked Out After Restricting SSH

**If you have console access** (via hosting provider):

```bash
# Log in via console
# Re-enable SSH on public zone
sudo firewall-cmd --permanent --zone=public --add-service=ssh
sudo firewall-cmd --reload
```

**If you configured failback IP**:

```bash
# Connect from your trusted IP address
ssh fedora@YOUR_PUBLIC_IP
# Should work even if VPN is down!
```

**Prevention**: Always configure a failback IP before restricting SSH!

### Tailscale IP Not Showing

```bash
# Check if device authenticated
sudo tailscale status

# If not authenticated, re-authenticate
sudo tailscale up

# Check network interfaces
ip addr show tailscale0
```

### Firewall Not Working as Expected

```bash
# List all zones and their settings
sudo firewall-cmd --list-all-zones | less

# Check which zone an interface is in
sudo firewall-cmd --get-zone-of-interface=eth0
sudo firewall-cmd --get-zone-of-interface=tailscale0

# Reset if needed (careful!)
sudo firewall-cmd --reload
```

---

## Summary

✅ **Tailscale VPN**: Installed and running
✅ **VPN Connection**: Tested and working
✅ **Firewall Zones**: Configured (trusted for VPN, public for internet)
✅ **MagicDNS**: Enabled (optional)
✅ **SSH Access**: Working via VPN (and optionally public)
✅ **Failback IP**: Configured for emergency access (if you restricted SSH)
✅ **Service Control**: Know how to make services VPN-only or public

**Key Concepts Learned**:
1. **VPN creates a private network** between your devices
2. **Firewall zones** separate VPN traffic from public internet traffic
3. **Trusted zone** (VPN) allows all traffic by default
4. **Public zone** only allows explicitly permitted services
5. **Failback IPs** provide emergency access when VPN is down
6. **You control** what's accessible from where

---

## Next Steps

### Recommended Configuration

Based on what you'll host, choose your security model:

**High Security (Recommended)**:
- SSH: VPN only + failback IP (emergency access)
- Admin panels: VPN only
- Databases: VPN only (never public!)
- Public website: Public (HTTP/HTTPS)

**Moderate Security**:
- SSH: Public (protected by keys + CrowdSec)
- Admin panels: VPN only
- Databases: VPN only
- Public website: Public

**Why failback IP?**: If your VPN service has issues or your VPN client breaks, you can still access your server from your trusted IP. This prevents complete lockout scenarios.

### Future Guides

1. **Guide 03**: Setting up Caddy web server
   - Reverse proxy configuration
   - Automatic HTTPS with Let's Encrypt
   - Serving websites via VPN vs public

2. **Guide 04**: Database setup
   - PostgreSQL installation
   - VPN-only database access
   - Backup configuration

3. **Guide 05**: Monitoring and logging
   - Log aggregation
   - Uptime monitoring
   - Security alerts

### Testing Your Setup

```bash
# Reboot test (optional)
sudo reboot

# Wait 2-3 minutes
# Test VPN connection
ping 100.x.x.x

# Test SSH via VPN
ssh fedora@100.x.x.x

# Everything should work after reboot!
```

---

## Quick Reference Commands

```bash
# Check VPN status
sudo tailscale status

# Get your Tailscale IP
tailscale ip -4

# View firewall zones
sudo firewall-cmd --get-active-zones

# List public zone services
sudo firewall-cmd --zone=public --list-all

# List rich rules (failback IPs)
sudo firewall-cmd --zone=public --list-rich-rules

# Add failback IP for SSH
sudo firewall-cmd --permanent --zone=public --add-rich-rule="rule family='ipv4' source address='YOUR_IP' service name='ssh' accept"
sudo firewall-cmd --reload

# Remove failback IP
sudo firewall-cmd --permanent --zone=public --remove-rich-rule="rule family='ipv4' source address='YOUR_IP' service name='ssh' accept"
sudo firewall-cmd --reload

# Add service to public zone
sudo firewall-cmd --permanent --zone=public --add-service=SERVICE_NAME
sudo firewall-cmd --reload

# Remove service from public zone
sudo firewall-cmd --permanent --zone=public --remove-service=SERVICE_NAME
sudo firewall-cmd --reload

# Restart Tailscale
sudo systemctl restart tailscaled

# View VPN setup
~/vpn-check.sh
```

---

## Additional Resources

- **Tailscale Docs**: https://tailscale.com/kb/
- **Tailscale Admin Console**: https://login.tailscale.com/admin
- **Firewalld Docs**: https://firewalld.org/documentation/

---

**Congratulations!** You now have a secure VPN setup and understand how to control access to your server services. You can safely proceed to setting up web services and applications.
