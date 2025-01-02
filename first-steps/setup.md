# Proxmox-Installation on a Dedicated Server (Hetzner)

This is a guide to initially install and sevurely setup a Proxmox VE on a dedicated server with Hetzner.

To securely access the server remotely, we add a VPS that runs on another server in Hetzner cloud and connect these via vSwitch.

The Proxmox VE will be used for several VMs to host workplace environments such as Nextcloud, Mailcow, etc.

In later steps we will connect these environments to a Kubernetes cluster that serves LLMs and AI solutions to automate and enhance workflows in the workplace environments.

The intention for documententing the steps is to enable anyone who wants to build their own infrastructure from scratch.

## VPS Setup

Create Cloud Server
- Location: Same as main server
- OS: Debian 12
- Type: CPX11 (2 vCPU, 2GB RAM)
- Enable private networking

Configure Network:
nano /etc/network/interfaces
auto ens10
iface ens10 inet static
    address 10.0.0.2/24

Install WireGuard
apt update && apt install wireguard

Generate Keys
wg genkey | tee /etc/wireguard/private.key | wg pubkey > /etc/wireguard/public.key

Configure WireGuard
nano /etc/wireguard/wg0.conf
Interface
Address = 10.8.0.1/24
PrivateKey = server-private-key
ListenPort = 51820
[Peer]
PublicKey = client-public-key
AllowedIPs = 10.8.0.2/32

Enable IP Forwarding
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p

## Installation and basic setup

Login to root@rescue

$ installimage

Select Other / Bookworm

In editor, select RAID Level

In editor, add hostname 

subdomain.domain.xyz

if several nodes are going to be intalled (later) use: node1.subdomain.domain.xyz

In editor, in section PARTITIONING / FILESYSTEM add

PART swap swap 6G
PART /boot ext3 1024M
PART lvm vg0 all
LV vg0 root / ext4 25G
LV vg0 data /var/lib/vz ext4 3540G

...depending on the total size of thedisk / partition to be used

After succesful installation, ssh into the server

edit the DNS-Lookup in the config, so that ssh doesnt take forever:

nano /etc/ssh/sshd_config

UseDNS no

systemctl restart sshd

After restart change the root password

passwd root

and note it down

Go to subdomain.domain.xyz and login to the Promox VE console with the new passwort

In Datacenter -> Permissions -> Two Factor and set up TFA
In Datacenter -> User lookup TFA and select "yes"

In Proxmox VE open Shell

either:
bash -c "$(wget -qLO - https://github.com/tteck/Proxmox/raw/main/misc/post-pve-install.sh)"

or:
bash -c "$(wget -qLO - https://github.com/sebeumann/Proxmox/raw/main/misc/post-pve-install.sh)"

- deactive Proxmox Enterprise
- enable no-subscription repo
- correct ceph-packages
- don't add pve test-repo
- disable subscription nag
- don't disable high availability
- update Proxmox VE
- reboot

## Firewall and VSwitch Setup


In Proxmox VE Datacenter -> Firewall -> Security Group click on "Create", add Group Name 

In this new group, click "Add"-Button
1. Direction "in", Action "Accept", Macro: "SSH", mark enable
2. Direction "in", Action "Accept", Macro: "HTTP", mark enable
3. Direction "in", Action "Accept", Macro: "HTTPS", mark enable
4. Direction "in", Action "Accept", Protocol: "tcp", Dest. port: "8006" mark enable

In Datacenter -> Firewall click button: "Insert Security Group", select security group, mark enable

In Node1 -> Firewall click button: "Insert Security Group", select security group, mark enable

In Datacenter -> Firewall -> Firewall Options Click on "Firewall" and select yes

In Node1 -> Firewall -> Firewall Options Click on "Firewall" and select yes








