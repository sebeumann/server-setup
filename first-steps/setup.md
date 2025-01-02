# Secure Proxmox VE Installation on Hetzner Dedicated Server

This comprehensive guide details how to securely install and configure Proxmox VE on a Hetzner dedicated server using a VPS in Hetzner Cloud as a VPN gateway. The setup ensures maximum security by restricting access to your infrastructure through a WireGuard VPN.

> **⚠️ Important Safety Note**: Throughout this guide, whenever making changes to SSH, firewall rules, or network configurations, always maintain at least one active SSH session while testing changes in another. This prevents accidental lockouts.

## Core Features

- Secure access to Proxmox through WireGuard VPN
- SSH and Proxmox Web UI restricted to VPN subnet
- Minimal attack surface with comprehensive security measures
- Detailed backup and monitoring strategies
- High availability and load balancing configurations

## Table of Contents

[Previous sections remain the same...]

## Prerequisites

Before beginning the installation, ensure you have:

- SSH key pair generated on your local machine
  ```bash
  # Generate if needed - DO NOT store private key in public locations
  ssh-keygen -t ed25519 -C "your_email@example.com"
  ```
- WireGuard tools installed locally
  - Linux: `apt install wireguard` or equivalent
  - Windows: Official WireGuard GUI installer
  - macOS: App Store or `brew install wireguard-tools`
- Access to Hetzner Robot and Cloud Console
- Basic understanding of networking concepts
- Terminal/SSH client
- Text editor for configuration files

Required software versions:
- Debian 12 (Bookworm)
- WireGuard
- Proxmox VE 8.x (latest stable)

> **🔒 Security Note**: Throughout this guide, protect private keys and sensitive configurations. Never commit them to version control or share them insecurely.

[Network Architecture section remains the same...]

## VPN Server Configuration

### 1. Initial VPS Access

```bash
# SSH into VPS - keep this session open while testing configurations
ssh root@<vps-public-ip>

# Update system
apt update && apt upgrade -y
```

### 2. Install and Configure WireGuard

> **⚠️ Important**: Do not enable strict firewall rules until you've verified VPN connectivity. This prevents accidental lockouts.

```bash
# Install WireGuard
apt install -y wireguard

# Generate server keys
cd /etc/wireguard
wg genkey | tee server_private.key | wg pubkey > server_public.key
chmod 600 server_private.key  # Restrict private key access

# Create server config
cat > /etc/wireguard/wg0.conf << EOF
[Interface]
Address = 10.8.0.1/24
PrivateKey = $(cat server_private.key)
ListenPort = 51820

# These PostUp/PostDown rules enable NAT for VPN clients to access internet via VPS
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT
PostUp = iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
EOF

# Enable IP forwarding - needed for VPN traffic routing
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p

# Start WireGuard
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0

# Verify interface is up
wg show
```

## Local Machine VPN Setup

### Cross-Platform Configuration

The configuration process varies slightly by operating system:

**Linux**:
```bash
# Generate client keys
wg genkey | tee client_private.key | wg pubkey > client_public.key
chmod 600 client_private.key  # Restrict private key access
```

**Windows/macOS**:
- Use the official WireGuard GUI
- It will generate keys automatically
- Import the configuration file created below

### Create Client Configuration

Create `myvpn.conf` (Linux) or use GUI import (Windows/macOS):

```ini
[Interface]
Address = 10.8.0.10/24
PrivateKey = <contents-of-client_private.key>
DNS = 1.1.1.1

[Peer]
PublicKey = <contents-of-server_public.key>
AllowedIPs = 10.8.0.0/24
Endpoint = <vps-public-ip>:51820
PersistentKeepalive = 25
```

> **🔒 Security Note**: Protect your private key (`client_private.key`). Never share it or store it in unsecured locations.

[Continue with previous sections, adding similar explanatory notes and security warnings throughout...]

## Proxmox Installation and Configuration

### 1. Boot into Rescue Mode

> **⚠️ Note**: The rescue system is temporary. Any changes made here (except through `installimage`) will not persist after reboot.

1. Access Hetzner Robot
2. Enable rescue system (Linux 64-bit)
3. Set temporary password
4. Reboot server

### 2. Install Debian

We'll use a specific partition layout optimized for Proxmox:

```bash
# SSH into rescue system
ssh root@<dedicated-server-ip>

# Start installation
installimage
```

Partition layout explanation:
- `/boot` (1024MB): Separate partition for bootloader and kernels
- `swap` (6GB): Dedicated swap space for memory management
- `root` (25GB): System files and Proxmox installation
- `/var/lib/vz` (remaining space): Separate partition for VM storage
  - Keeps VM data separate from system
  - Survives OS reinstalls
  - Easier to manage backups


### 3. Install Proxmox

```bash
# SSH into fresh Debian install
ssh root@<dedicated-server-ip>

# Add Proxmox repository
# Using no-subscription repository - suitable for testing/personal use
# For production environments, consider a subscription for enterprise support
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
> /etc/apt/sources.list.d/pve-no-subscription.list

# Add GPG key - note version number in URL
wget -qO - http://download.proxmox.com/debian/proxmox-ve-release-8.x.gpg | apt-key add -

# Install Proxmox
apt update
apt install -y proxmox-ve postfix open-iscsi

# Disable enterprise repository (optional)
echo "# deb https://enterprise.proxmox.com/debian/pve bookworm pve-enterprise" \
> /etc/apt/sources.list.d/pve-enterprise.list
```

> **📝 Note**: The `pve-no-subscription` repository is suitable for personal or testing environments. For production use, consider purchasing a Proxmox subscription for enterprise support and updates.

### 4. Configure as VPN Client

```bash
# Install WireGuard
apt install -y wireguard

# Generate keys
wg genkey | tee /etc/wireguard/dedicated_private.key | wg pubkey > /etc/wireguard/dedicated_public.key
chmod 600 /etc/wireguard/dedicated_private.key

# Create config
cat > /etc/wireguard/wg0.conf << EOF
[Interface]
Address = 10.8.0.2/24
PrivateKey = $(cat /etc/wireguard/dedicated_private.key)
DNS = 1.1.1.1

[Peer]
PublicKey = <server_public.key from VPS>
AllowedIPs = 10.8.0.0/24
Endpoint = <vps-public-ip>:51820
PersistentKeepalive = 25
EOF

# Start WireGuard
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0

# Verify connection
ping -c 3 10.8.0.1  # Should reach VPN server
wg show  # Check for "latest handshake" time
```

### 5. Restrict Proxmox Web Interface to VPN

```bash
# Edit Proxmox proxy configuration
cat > /etc/default/pveproxy << EOF
ALLOW_FROM="127.0.0.1,10.8.0.0/24"
DENY_FROM="all"
EOF

# Restart proxy service
systemctl restart pveproxy

# Verify you can still access web UI via https://10.8.0.2:8006
```

> **⚠️ Warning**: Test the web UI access through VPN before proceeding. Keep your SSH session active while testing.

## Security Hardening

### 1. SSH Key Management

```bash
# In your FIRST terminal (keep this active)
ssh root@10.8.0.2

# In a SECOND terminal, prepare for key-only authentication
mkdir -p ~/.ssh
chmod 700 ~/.ssh
touch ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# Add your public key to authorized_keys
echo "your-public-key" >> ~/.ssh/authorized_keys

# Configure SSH daemon
cat > /etc/ssh/sshd_config.d/restrict.conf << EOF
PasswordAuthentication no
PermitRootLogin prohibit-password
UseDNS no
EOF

# Test new connection in a THIRD terminal before restarting sshd
ssh -i /path/to/private_key root@10.8.0.2

# If test succeeds, restart SSH daemon
systemctl restart sshd
```

> **🔒 Security Note**: Always test new SSH configurations with a separate session before restarting sshd.

### 2. Firewall Configuration

#### VPS Firewall

```bash
# Install UFW
apt install -y ufw

# Configure rules
ufw default deny incoming
ufw default allow outgoing
ufw allow 51820/udp comment 'WireGuard VPN'
ufw allow from 10.8.0.0/24 to any port 22 comment 'SSH from VPN'
ufw enable
```

#### Proxmox Firewall

1. Enable Proxmox firewall in web UI:
   - Datacenter → Firewall → Options → Enable: Yes
   - Node → Firewall → Options → Enable: Yes

2. Create security group:
   - Datacenter → Firewall → Security Group
   - Add new group: "vpn-access"
   - Add rules:
     ```
     Direction: IN
     Action: ACCEPT
     Protocol: tcp
     Port: 22
     Source: 10.8.0.0/24
     Comment: SSH from VPN
     ```
     ```
     Direction: IN
     Action: ACCEPT
     Protocol: tcp
     Port: 8006
     Source: 10.8.0.0/24
     Comment: Proxmox Web UI from VPN
     ```

### 3. Security Auditing

```bash
# Install security tools
apt install -y fail2ban rkhunter lynis

# Configure fail2ban
cat > /etc/fail2ban/jail.local << EOF
[sshd]
enabled = true
bantime = 1h
findtime = 10m
maxretry = 3
EOF

# Start fail2ban
systemctl enable fail2ban
systemctl start fail2ban

# Run initial security audit
lynis audit system
```

> **📝 Note**: Review `/var/log/auth.log` regularly for unauthorized access attempts.

## Advanced Configuration

### 1. Backup Strategy

#### Configure Proxmox Backup Server (PBS)

```bash
# Install PBS
apt install proxmox-backup-server

# Create datastore
proxmox-backup-manager create datastore backup /path/to/storage

# Configure retention
proxmox-backup-manager update-datastore backup --keep-last 7
```

#### Configure VM Backups

In Proxmox Web UI:
1. Datacenter → Backup
2. Add → Schedule:
   - Schedule: `0 2 * * *` (2 AM daily)
   - Selection: All
   - Storage: backup
   - Compression: zstd
   - Mode: Snapshot
   - Enable: Yes

> **⚠️ Important**: Regularly test backup restoration to ensure data recovery is possible.

### 2. High Availability

Prerequisites:
- Minimum 3 nodes for quorum
- Shared storage (e.g., Ceph, NFS)
- Identical network configuration

```bash
# On first node
pvecm create cluster_name

# On additional nodes
pvecm add first_node_ip

# Verify cluster
pvecm status
```

Configure HA Groups:
1. Datacenter → HA → Groups
2. Add → Configure:
   - Nodes: Select all nodes
   - Restricted: No
   - NoFailback: No

### 3. Load Balancing

```bash
# Install HAProxy
apt install -y haproxy

# Basic configuration
cat > /etc/haproxy/haproxy.cfg << EOF
global
    log /dev/log local0
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000

frontend proxmox_fe
    bind *:8006
    mode http
    default_backend proxmox_be

backend proxmox_be
    mode http
    balance roundrobin
    option httpchk GET /
    server prox1 10.8.0.2:8006 check
    server prox2 10.8.0.3:8006 check
EOF

# Start HAProxy
systemctl enable haproxy
systemctl start haproxy
```

## Monitoring and Alerting

### 1. System Monitoring

```bash
# Install monitoring stack
apt install -y prometheus node-exporter grafana

# Configure Prometheus
cat > /etc/prometheus/prometheus.yml << EOF
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']
EOF

# Start services
systemctl enable prometheus node-exporter grafana-server
systemctl start prometheus node-exporter grafana-server
```

### 2. Log Monitoring

```bash
# Configure systemd journal
cat > /etc/systemd/journald.conf << EOF
[Journal]
Storage=persistent
SystemMaxUse=4G
SystemMaxFileSize=256M
EOF

# Restart journald
systemctl restart systemd-journald
```

## Troubleshooting Guide

### 1. VPN Connectivity Issues

Check WireGuard status:
```bash
# View interface status
wg show

# Check logs
journalctl -xu wg-quick@wg0
dmesg | grep wireguard

# Verify routing
ip route show
```

Common issues:
- Incorrect keys: Verify public/private key pairs
- Firewall blocking: Check UFW status
- Routing issues: Verify AllowedIPs configuration

### 2. Proxmox Web UI Access

If web UI is inaccessible:
```bash
# Check pveproxy status
systemctl status pveproxy

# Verify listening ports
ss -tulpn | grep 8006

# Check firewall rules
ufw status verbose
```

### 3. SSH Connection Problems

```bash
# Enable verbose SSH logging
ssh -vv root@10.8.0.2

# Check SSH daemon logs
journalctl -u sshd

# Verify SSH configuration
sshd -T
```

## Maintenance Procedures

### 1. Regular Updates

```bash
# Update system packages
apt update && apt upgrade -y

# Check Proxmox version
pveversion

# Verify service status
systemctl status pve*
```

### 2. Security Checks

Monthly security audit:
```bash
# Update security databases
rkhunter --update
rkhunter --propupd

# Run security scan
lynis audit system

# Check fail2ban status
fail2ban-client status sshd
```

### 3. Backup Verification

Quarterly backup testing:
1. Create test VM
2. Restore from backup
3. Verify functionality
4. Document results

## Additional Considerations

### 1. SSL Certificates

```bash
# Install certbot
apt install -y certbot

# Generate certificate
certbot certonly --standalone -d proxmox.yourdomain.com

# Configure Proxmox to use certificate
cp /etc/letsencrypt/live/proxmox.yourdomain.com/fullchain.pem /etc/pve/local/pveproxy-ssl.pem
cp /etc/letsencrypt/live/proxmox.yourdomain.com/privkey.pem /etc/pve/local/pveproxy-ssl.key

# Restart proxy
systemctl restart pveproxy
```

### 2. Performance Optimization

```bash
# Configure CPU governor
echo "performance" | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# Optimize disk I/O
cat > /etc/sysctl.d/99-disk.conf << EOF
vm.dirty_ratio = 10
vm.dirty_background_ratio = 5
vm.swappiness = 10
EOF

# Apply changes
sysctl --system
```

## References

- [Proxmox VE Documentation](https://pve.proxmox.com/pve-docs/)
- [WireGuard Documentation](https://www.wireguard.com/)
- [Hetzner Robot API](https://robot.your-server.de/doc/webservice/en.html)
- [Debian Security Guide](https://www.debian.org/doc/manuals/securing-debian-howto/)

Remember to:
- Regularly review security policies
- Test backup restoration procedures
- Monitor system resources
- Keep documentation updated
- Maintain offline copies of critical configurations
5. Including troubleshooting tips inline with relevant sections
6. Adding version-specific notes
7. Enhancing backup and recovery instructions

Would you like me to continue with the complete document?
