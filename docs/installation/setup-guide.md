# Proxmox VE Installation on Hetzner Dedicated Server

This comprehensive guide details how to securely install and configure Proxmox VE on a Hetzner dedicated server using a VPS in Hetzner Cloud as a VPN gateway. The setup ensures maximum security by restricting access to your infrastructure through a WireGuard VPN.

> **⚠️ Important Safety Note**: Throughout this guide, whenever making changes to SSH, firewall rules, or network configurations, always maintain at least one active SSH session while testing changes in another. This prevents accidental lockouts.

## Core Features

- Secure access to Proxmox through WireGuard VPN
- SSH and Proxmox Web UI restricted to VPN subnet
- Minimal attack surface with comprehensive security measures
- Detailed backup and monitoring strategies
- High availability and load balancing configurations

## Installation Methods
This guide describes the manual installation process for setting up Proxmox VE on a Hetzner dedicated server with a VPN gateway. Generally, this repository offers two approaches available for this setup:

1. Manual Installation (This Guide)
This guide provides step-by-step instructions for manually setting up and configuring all components. This approach is recommended for:

- Learning and understanding each component of the setup
- Custom configurations not covered by automation
- Situations requiring maximum flexibility
- Testing and development environments

2. Automated Installation
An alternative automated approach using Infrastructure as Code (IaC) is available, utilizing:

- Terraform for VPS/VPN Gateway provisioning
- Ansible for configuration management
- Version-controlled configuration files
- Reproducible deployment process

The automated approach is recommended for:

- Production environments
- Multiple server deployments
- Consistent, reproducible setups
- Team-based management

→ For the automated approach, see Automated Deployment Guide

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [VPN Server Configuration](#vpn-server-configuration)
   - [Initial VPS Access](#1-initial-vps-access)
   - [Install and Configure WireGuard](#2-install-and-configure-wireguard)
3. [Local Machine VPN Setup](#local-machine-vpn-setup)
   - [Cross-Platform Configuration](#cross-platform-configuration)
   - [Create Client Configuration](#create-client-configuration)
4. [Proxmox Installation and Configuration](#proxmox-installation-and-configuration)
   - [Boot into Rescue Mode](#1-boot-into-rescue-mode)
   - [Install Debian](#2-install-debian)
   - [Install Proxmox](#3-install-proxmox)
   - [Configure as VPN Client](#4-configure-as-vpn-client)
   - [Restrict Proxmox Web Interface to VPN](#5-restrict-proxmox-web-interface-to-vpn)
5. [Security Hardening](#security-hardening)
   - [SSH Key Management](#1-ssh-key-management)
   - [Firewall Configuration](#2-firewall-configuration)
   - [Security Auditing](#3-security-auditing)
6. [Advanced Configuration](#advanced-configuration)
   - [Backup Strategy](#1-backup-strategy)
   - [High Availability](#2-high-availability)
   - [Load Balancing](#3-load-balancing)
7. [Monitoring and Alerting](#monitoring-and-alerting)
   - [System Monitoring](#1-system-monitoring)
   - [Log Monitoring](#2-log-monitoring)
8. [Troubleshooting Guide](#troubleshooting-guide)
   - [VPN Connectivity Issues](#1-vpn-connectivity-issues)
   - [Proxmox Web UI Access](#2-proxmox-web-ui-access)
   - [SSH Connection Problems](#3-ssh-connection-problems)
9. [Maintenance Procedures](#maintenance-procedures)
   - [Regular Updates](#1-regular-updates)
   - [Security Checks](#2-security-checks)
   - [Backup Verification](#3-backup-verification)
10. [Additional Considerations](#additional-considerations)
    - [SSL Certificates](#1-ssl-certificates)
    - [Performance Optimization](#2-performance-optimization)
11. [References](#references)

## Prerequisites

Before beginning the installation, ensure you have:

- SSH key pair generated on your local machine
  ```bash
  # Generate if needed - DO NOT store private key in public locations
  ssh-keygen -t ed25519 -C "your_email@example.com" -f ~/.ssh/id_ed25519_server
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

## VPN Server Configuration

### 1. Initial VPS Access

First, establish SSH access to your VPS:
```bash
# SSH into VPS - keep this session open while testing configurations
ssh root@<vps-public-ip>

# Update system
apt update && apt upgrade -y
```

### 2. Install and Configure WireGuard

Install required packages:
```bash
# Install WireGuard
apt install -y wireguard
```

Generate server keys:
```bash
# Generate and secure keys
cd /etc/wireguard
wg genkey | tee server_private.key | wg pubkey > server_public.key
chmod 600 server_private.key
```

Configure WireGuard:
1. Copy `config/vpn/wireguard-server.conf` to `/etc/wireguard/wg0.conf`
2. Replace placeholder variables with actual values

Enable IP forwarding:
```bash
# Enable IP forwarding for VPN
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
```

Start WireGuard:
```bash
# Enable and start service
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0

# Verify interface
wg show
```

## Local Machine VPN Setup

### Cross-Platform Configuration

Generate client keys based on your operating system:

**Linux**:
```bash
# Generate client keys
wg genkey | tee client_private.key | wg pubkey > client_public.key
chmod 600 client_private.key
```

**Windows/macOS**:
- Use the WireGuard GUI
- Import the configuration from `config/vpn/wireguard-client.conf`

### Create Client Configuration

1. Copy `config/vpn/wireguard-client.conf` to your local machine
2. Replace placeholder variables with actual values
3. Import into WireGuard application

## Proxmox Installation and Configuration

### 1. Boot into Rescue Mode

1. Access Hetzner Robot
2. Select Linux 64-bit rescue system
3. Note the temporary password
4. Reboot server
5. SSH into rescue system using temporary password

### 2. Install Debian

Access rescue system and start installation:
```bash
# SSH into rescue system
ssh root@<dedicated-server-ip>

# Start installation
installimage
```

Select the following options in the installation menu:
- Operating system: Debian 12
- Hostname: your-hostname
- Partitioning: Use custom layout from `config/proxmox/partitioning.conf`

### 3. Install Proxmox

Add Proxmox repository:
```bash
# Add repository configuration
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > \
  /etc/apt/sources.list.d/pve-no-subscription.list

# Add GPG key
wget -qO - http://download.proxmox.com/debian/proxmox-ve-release-8.x.gpg | apt-key add -
```

Install Proxmox:
```bash
# Update and install
apt update
apt install -y proxmox-ve postfix open-iscsi
```

### 4. Configure as VPN Client

Install WireGuard:
```bash
# Install WireGuard
apt install -y wireguard
```

Configure client:
1. Copy `config/vpn/wireguard-client.conf` to `/etc/wireguard/wg0.conf`
2. Replace placeholder variables
3. Start service:
```bash
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0
```

### 5. Restrict Proxmox Web Interface to VPN

1. Copy `config/proxmox/pveproxy.conf` to `/etc/default/pveproxy`
2. Restart proxy:
```bash
systemctl restart pveproxy
```

[Continue with the remaining sections following the same pattern of separating configuration files and commands...]

## Security Hardening

### 1. SSH Key Management

Configure key-based authentication:
```bash
# Prepare SSH directory
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```

1. Copy `config/ssh/sshd_config` to `/etc/ssh/sshd_config.d/restrict.conf`
2. Add your public key:
```bash
echo "your-public-key" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Test and apply configuration:
```bash
# Test in new session before restarting
ssh -i /path/to/private_key root@10.8.0.2

# If successful, restart SSH
systemctl restart sshd
```

[Continue with remaining sections...]

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
