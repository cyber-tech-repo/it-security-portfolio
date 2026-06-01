# Debian 12 + AdGuard Home Secure DNS Appliance Setup Guide

> Platform: Debian 12  
> Software: AdGuard Home  
> Focus Areas: Linux administration, DNS infrastructure, network services, self-hosting, system hardening  
> Environment: Self-hosted home lab  
> Status: Public sanitized release  

---

## Overview

This document provides a complete walkthrough for deploying a self-hosted DNS filtering appliance on Debian 12 using AdGuard Home with Quad9 DNS-over-HTTPS as the upstream resolver.

The system is designed for local network use, providing DNS visibility, filtering, and controlled outbound resolution in a lightweight and maintainable configuration.

---

## Project Summary

The goal of this project is to demonstrate the deployment and operational configuration of a dedicated Debian-based DNS appliance.

It emphasizes practical infrastructure skills including system setup, network configuration, service management, DNS resolution architecture, and system hardening within a controlled home lab environment.

The documentation is written to be reproducible and suitable as a reference implementation for similar deployments.

---

## Table of Contents

* [Goal](#goal)
* [Architecture](#architecture)
* [Hardware Assumptions](#hardware-assumptions)
* [Why Debian 12](#why-debian-12)
* [Download Debian 12 ISO](#download-debian-12-iso)
* [Create Bootable USB](#create-bootable-usb)
* [Install Debian 12](#install-debian-12)
* [Debian Installer Screen-by-Screen Choices](#debian-installer-screen-by-screen-choices)
* [First Login](#first-login)
* [Initial System Updates](#initial-system-updates)
* [Verify User Privileges](#verify-user-privileges)
* [SSH Hardening](#ssh-hardening)
* [Disable Sleep / Lid Suspend](#disable-sleep--lid-suspend)
* [Identify the Correct Wired Interface](#identify-the-correct-wired-interface)
* [Configure Static IP](#configure-static-ip)
* [Firewall Setup with UFW](#firewall-setup-with-ufw)
* [Install AdGuard Home](#install-adguard-home)
* [AdGuard Home Initial Setup](#adguard-home-initial-setup)
* [Configure Upstream DNS with Quad9 DoH](#configure-upstream-dns-with-quad9-doh)
* [Verify DNS Resolution](#verify-dns-resolution)
* [Switch Router DNS to AdGuard](#switch-router-dns-to-adguard)
* [Flush / Renew Client DNS](#flush--renew-client-dns)
* [Verify Traffic Is Flowing Through AdGuard](#verify-traffic-is-flowing-through-adguard)
* [AdGuard Tuning](#adguard-tuning)
* [HTTPS / Encryption Notes](#https--encryption-notes)
* [Hardening the Debian Appliance](#hardening-the-debian-appliance)
* [Troubleshooting](#troubleshooting)
* [Final Recommended State](#final-recommended-state)

---

# Goal

Build a small, wired, always-on Debian 12 appliance that provides:

* Local DNS filtering
* Per-device DNS visibility
* Malware/ad/tracker blocking
* Encrypted upstream DNS using Quad9 DNS-over-HTTPS
* A lightweight web dashboard through AdGuard Home
* A hardened Linux base system
* Minimal unnecessary services and background activity

This is designed as a **home network visibility and DNS control node**, not a general-purpose desktop.

---

# Architecture

```mermaid
graph LR
    A[Devices<br/>Phones / Laptops / IoT] --> B[Router<br/>DHCP Server]
    B --> C[AdGuard Home<br/>Debian 12 DNS Appliance]

    C --> D[Quad9 DNS-over-HTTPS<br/>dns.quad9.net]
    D --> E[Internet]

    C --> F[DNS Query Log]
    C --> G[Blocklists / Filtering Engine]
````

## System Flow

Devices on the local network use the router for DHCP.
The router forwards DNS requests to the AdGuard Home appliance.

AdGuard Home provides:

* DNS resolution for all LAN devices
* Ad and tracker blocking via blocklists
* Per-device query logging and visibility
* Encrypted upstream DNS via Quad9 (DoH)



Example static IP used in this guide:

```text
AdGuard Home server: 192.168.0.2 (example static ip)
Router/gateway:      192.168.0.1 (example gateway)
Primary upstream:    https://dns.quad9.net/dns-query
Fallback IP DNS:     9.9.9.9 / 149.112.112.112
```

---

# Hardware Assumptions

This guide assumes:

* An unused laptop
* Wired Ethernet connection
* At least 8 GB RAM
* Internal SSD/NVMe drive
* Debian 12 installed directly to disk
* Laptop will remain powered on
* Laptop will act like an appliance, not a daily-use workstation

8 GB RAM is more than enough for:

* Debian minimal
* AdGuard Home
* SSH
* Basic logging
* Light diagnostic tools

---

# Why Debian 12

Recommended OS:

```text
Debian 12 Bookworm minimal install
```

## Why Debian 12?

| Reason                 | Benefit                                |
| ---------------------- | -------------------------------------- |
| Stable                 | Good for always-on infrastructure      |
| Lightweight            | Excellent for older or low-RAM systems |
| Predictable            | Fewer unexpected changes               |
| Huge package ecosystem | Easy to install tools                  |
| Low noise              | Better for network visibility          |
| Long support lifecycle | Less maintenance pressure              |

## Why not Kali?

Kali can run AdGuard Home, but it is not ideal for always-on infrastructure.

| Kali                        | Debian                    |
| --------------------------- | ------------------------- |
| Pentesting-focused          | Server/appliance-friendly |
| More tools by default       | Minimal and clean         |
| More background noise       | Lower noise baseline      |
| More frequent breakage risk | More stable               |

Recommended approach:

```text
Debian minimal + only install the tools you actually need
```

---

# Download Debian 12 ISO

Use the official Debian download site:

```text
https://www.debian.org/distrib/
```

Recommended ISO type:

```text
Debian 12 amd64 netinst ISO
```

The file will look something like:

```text
debian-12.x.x-amd64-netinst.iso
```

Use the `amd64` ISO for normal Intel/AMD laptops.

---

# Create Bootable USB

Use one of the following:

## Windows

Recommended:

```text
Rufus
```

Basic Rufus choices:

```text
Device: your USB drive
Boot selection: Debian 12 netinst ISO
Partition scheme: GPT
Target system: UEFI
File system: FAT32
```

## macOS / Linux

Recommended:

```text
balenaEtcher
```

Flow:

```text
Select ISO → Select USB → Flash
```

---

# Install Debian 12

## Boot from USB

1. Insert USB installer
2. Power on laptop
3. Press boot menu key

Choose the USB device.

---

# Debian Installer Screen-by-Screen Choices

At the first Debian boot menu, options may include:

```text
Graphical install
Install
Advanced options
Accessible dark contrast installer menu
```

Choose:

```text
Install
```

Why:

* Minimal
* Cleaner
* No graphical installer needed
* Best for a headless appliance

---

## Language

Choose:

```text
English
```

---

## Location

Choose:

```text
United States
```

---

## Keyboard

Choose:

```text
American English
```

---

## Network Interface Selection

If prompted to choose an interface, choose:

```text
Ethernet / wired interface
```

Do not choose Wi-Fi unless absolutely necessary.

A wired install is preferred because:

* More reliable
* Cleaner network baseline
* Better for an always-on DNS appliance
* Avoids Wi-Fi dropouts breaking DNS

---

## Does Debian Need Internet During Install?

Strictly:

```text
No, not absolutely.
```

Practically:

```text
Yes, recommended.
```

Using internet during install helps:

* Pull current packages
* Avoid outdated package metadata
* Install SSH server cleanly
* Reduce post-install cleanup

---

## Hostname

Use:

```text
adguard
```

---

## Domain Name

Leave blank unless you use a real local domain.

```text
<blank>
```

---

## Root Password

Recommended:

```text
Leave root password blank
```

This disables direct root login and encourages sudo usage.

---

## Create User

Example:

```text
Full name: user
Username: user
Password: strong password
```

Any username is fine.

---

## Time Zone

Choose your local time zone.

Accurate time matters because DNS logs and security events depend on correct timestamps.

---

## Disk Partitioning

When asked where to install Debian, carefully identify the internal disk.

Typical internal NVMe disk:

```text
/dev/nvme0n1
```

Typical USB installer may appear as:

```text
SCSI1
/dev/sda
```

Choose the internal drive, for example:

```text
/dev/nvme0n1
```

Do **not** install to the USB installer.

---

## Partitioning Method

Recommended:

```text
Guided - use entire disk
```

Then:

```text
All files in one partition
```

Then:

```text
Finish partitioning and write changes to disk
```

Choose:

```text
Yes
```

---

## Encryption?

Recommended for this appliance:

```text
Do not use encrypted LVM
```

Why:

| No Disk Encryption             | Encrypted LVM                       |
| ------------------------------ | ----------------------------------- |
| Auto-reboots cleanly           | Requires boot password              |
| Easier appliance behavior      | Harder remote recovery              |
| Less complexity                | More lockout risk                   |
| Good for minimal DNS appliance | Better for sensitive personal files |

Use encryption only if the laptop will store sensitive files or is at high risk of theft.

For a local DNS appliance, no encryption is simpler and more reliable.

---

## Package Manager Mirror

Country:

```text
United States
```

Mirror:

```text
Default Debian mirror is fine
```

---

## Package Survey

Choose:

```text
No
```

---

## Software Selection

This is an important screen.

You may see:

```text
Debian desktop environment
GNOME
KDE Plasma
Xfce
LXDE
Cinnamon
MATE
SSH server
standard system utilities
```

Recommended final selection:

```text
[ ] Debian desktop environment
[ ] GNOME
[ ] KDE Plasma
[ ] Xfce
[ ] LXDE
[ ] Cinnamon
[ ] MATE
[x] SSH server
[x] standard system utilities
```

Only select:

```text
SSH server
standard system utilities
```

Do **not** install a desktop environment.

AdGuard Home provides its own web dashboard, so the Debian machine does not need a GUI.

---

## GRUB Bootloader

Choose:

```text
Yes
```

Install GRUB to the primary internal disk, for example:

```text
/dev/nvme0n1
```

---

## Finish Install

After install completes:

1. Reboot
2. Remove USB installer
3. Boot from internal disk

---

# First Login

Log in with the user created during installation.

Example:

```text
user
```

---

# Initial System Updates

Update packages:

```bash
sudo apt update
sudo apt upgrade -y
```

Install curl:

```bash
sudo apt install curl -y
```

Optional useful tools:

```bash
sudo apt install net-tools dnsutils ufw fail2ban -y
```

`dnsutils` provides tools such as:

```bash
nslookup
dig
```

---

# Verify User Privileges

Check groups:

```bash
groups
```

Example output:

```text
user cdrom floppy sudo audio dip video plugdev users netdev bluetooth
```

The important group is:

```text
sudo
```

If your user is in the `sudo` group, you have administrative privileges.

---

# SSH Hardening

Edit SSH config:

```bash
sudo nano /etc/ssh/sshd_config
```

Find these lines:

```text
#PermitRootLogin prohibit-password
#PasswordAuthentication yes
```

Change them to:

```text
PermitRootLogin no
PasswordAuthentication yes
```
> Note: SSH key authentication is recommended for production hardening.

Important:

* Remove the `#`
* Do not leave duplicate active lines
* Only one active `PermitRootLogin`
* Only one active `PasswordAuthentication`

Restart SSH:

```bash
sudo systemctl restart ssh
```

Check SSH service:

```bash
sudo systemctl status ssh
```

---

# Disable Sleep / Lid Suspend

This laptop should behave like an appliance.

Disable sleep, suspend, hibernate, and hybrid sleep:

```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

Configure lid close behavior:

```bash
sudo nano /etc/systemd/logind.conf
```

Set:

```text
HandleLidSwitch=ignore
```

Restart logind:

```bash
sudo systemctl restart systemd-logind
```

This allows the laptop to remain running even when the lid is closed.

---

# Identify the Correct Wired Interface

Modern Debian systems often do not use `eth0`.

Check interfaces:

```bash
ip a
```

Look for a wired interface name like:

```text
<ethernet-interface>
```

You may see the same interface listed with IPv4 and IPv6 information. That is normal.

Example:

```text
2: <ethernet-interface>: <BROADCAST,MULTICAST,UP,LOWER_UP>
```

The interface name in this example is:

```text
<ethernet-interface>
```

---

# Configure Static IP

The AdGuard server should have a predictable IP address.

Example:

```text
192.168.0.2 (example static ip)
```

Edit Debian network config:

```bash
sudo nano /etc/network/interfaces
```

You may see something like:

```text
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

allow-hotplug <ethernet-interface>
iface <ethernet-interface> inet dhcp
```

Replace the DHCP block:

```text
allow-hotplug <ethernet-interface>
iface <ethernet-interface> inet dhcp
```

With a static configuration:

```text
auto <ethernet-interface>
iface <ethernet-interface> inet static
  address 192.168.0.2
  netmask 255.255.255.0
  gateway 192.168.0.1
  dns-nameservers 9.9.9.9 149.112.112.112
```

Leave loopback alone:

```text
auto lo
iface lo inet loopback
```

Apply by rebooting:

```bash
sudo reboot
```

After reboot, verify:

```bash
ip a
```

Expected:

```text
192.168.0.2
```

---

# Firewall Setup with UFW

Install UFW:

```bash
sudo apt install ufw -y
```

Default deny incoming traffic:

```bash
sudo ufw default deny incoming
```

Default allow outgoing traffic:

```bash
sudo ufw default allow outgoing
```

Allow SSH:

```bash
sudo ufw allow 22/tcp
```

Allow DNS (TCP + UDP):

```bash
sudo ufw allow 53
```

Allow AdGuard Web UI (HTTP):

```bash
sudo ufw allow 80/tcp
```

Allow HTTPS (optional, for future hardening):

```bash
sudo ufw allow 443/tcp
```

Allow AdGuard setup interface (temporary, initial configuration only):

```bash
sudo ufw allow 3000/tcp
```

Enable firewall:

```bash
sudo ufw enable
```

Check status:

```bash
sudo ufw status numbered
```

Recommended active rules during initial setup (temporary configuration phase):

```text
22/tcp     ALLOW (SSH)
53         ALLOW (DNS TCP/UDP)
80/tcp     ALLOW (AdGuard Web UI)
443/tcp    ALLOW (optional future HTTPS)
3000/tcp   ALLOW (setup-only access)
```

After validating DNS functionality and confirming AdGuard is accessible through the intended interface, setup-only access rules are removed to reduce attack surface.

---

## Hardened Firewall State (Post-Setup)

After initial configuration and verification, the firewall is tightened for long-term operation.

Recommended production state:

```text
22/tcp     ALLOW (SSH - ideally key-based authentication only)
53         ALLOW (DNS TCP/UDP for LAN clients)
80/tcp     ALLOW (AdGuard Web UI OR disabled if HTTPS is enabled)
443/tcp    OPTIONAL (only if HTTPS is explicitly enabled for AdGuard admin UI)

3000/tcp   DENIED (setup-only port, must not remain open)
```

### Apply hardening:

```bash
sudo ufw deny 3000/tcp
sudo ufw delete allow 3000/tcp
sudo ufw status numbered
```

### Notes:

```text
- Port 3000 is only required during initial AdGuard setup and should be closed afterward
- DNS (53) must remain open for LAN functionality (TCP and UDP)
- Web UI access should be restricted to trusted local network devices
- HTTPS (443) should only be enabled if certificate-based configuration is implemented
- SSH (22) should ideally be restricted to trusted hosts or LAN subnet in advanced setups
```

---

## Final Firewall Posture Summary

```text
- Minimal exposed services
- No temporary setup ports remaining open
- LAN-only DNS resolver operation
- Administrative access restricted to SSH and local network
- Clear separation between setup-phase and production-phase rules
```

## Does Rule Order Matter?

This command:

```bash
sudo ufw default deny incoming
```

does **not** override allow rules.

It means:

```text
Deny incoming traffic unless an allow rule exists.
```

So allowing ports afterward is correct.

---

# Install AdGuard Home

Install with official script:

```bash
curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh
```

If needed with sudo:

```bash
curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sudo sh
```

Check service:

```bash
sudo systemctl status AdGuardHome
```

Expected:

```text
active (running)
```

---

# AdGuard Home Initial Setup

Open browser from another device:

```text
http://192.168.0.2
```

Or if using default setup port:

```text
http://192.168.0.2:3000
```

From the AdGuard machine itself:

```text
http://127.0.0.1
```

or:

```text
http://127.0.0.1:3000
```

---

## Admin Web Interface

Recommended:

```text
Listen interface: All interfaces
Port: 80
```

Why port 80:

```text
http://192.168.0.2
```

is cleaner than:

```text
http://192.168.0.2:3000
```

---

## DNS Server

Recommended:

```text
Listen interface: All interfaces
Port: 53
```

Port 53 is the standard DNS port.

---

## DHCP Server

Do **not** enable AdGuard DHCP.

Recommended:

```text
DHCP server: disabled
```

Your router should remain DHCP server.

AdGuard should be DNS only.

---

# Configure Upstream DNS with Quad9 DoH

In AdGuard:

```text
Settings → DNS Settings → Upstream DNS servers
```

Use Quad9 DNS-over-HTTPS:

```text
https://dns.quad9.net/dns-query
```

This means:

```text
Devices → AdGuard: local DNS
AdGuard → Quad9: encrypted DoH
```

---

## Bootstrap DNS

Set bootstrap DNS to Quad9 IPs:

```text
9.9.9.9
149.112.112.112
```

Bootstrap DNS is used so AdGuard can resolve the hostname:

```text
dns.quad9.net
```

before using DoH.

---

# Verify DNS Resolution

From the AdGuard server:

```bash
nslookup google.com 127.0.0.1
```

Expected:

```text
Name: google.com
Address: <some IP address>
```

Also test:

```bash
dig google.com @127.0.0.1
```

Expected:

```text
NOERROR
```

Check AdGuard logs:

```text
AdGuard Home → Query Log
```

You should see your test query.

---

# Switch Router DNS to AdGuard

Once AdGuard works locally, change your router’s DHCP DNS setting.

Set router DHCP DNS to:

```text
Primary DNS:   192.168.0.2
Secondary DNS: 192.168.0.2
```

or:

```text
Primary DNS:   192.168.0.2
Secondary DNS: blank
```

Do not switch router DNS before AdGuard is confirmed working.

---

# Flush / Renew Client DNS

After changing router DHCP DNS, clients may still have old DNS leases.

## Windows

Open Command Prompt as Administrator:

```cmd
ipconfig /flushdns
ipconfig /release
ipconfig /renew
```

Then verify:

```cmd
ipconfig /all
```

Look for:

```text
DNS Servers . . . . . . . . . . . : 192.168.0.2
```

## macOS

Toggle Wi-Fi off/on, or run:

```bash
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

## Linux with systemd-resolved

```bash
sudo resolvectl flush-caches
```

or:

```bash
sudo systemd-resolve --flush-caches
```

If unavailable, reconnect networking or reboot the client.

---

# Verify Traffic Is Flowing Through AdGuard

On a client device, confirm DNS:

```text
DNS server = 192.168.0.2
```

Open a few websites.

Then check:

```text
AdGuard Home → Query Log
```

You should see:

* Client IP
* Requested domains
* Allowed/blocked status
* Upstream DNS server used

Click a query and confirm upstream shows:

```text
https://dns.quad9.net/dns-query
```

That confirms DoH upstream is working.

---

# AdGuard Tuning

## DNS Settings

Recommended upstream:

```text
https://dns.quad9.net/dns-query
```

Bootstrap:

```text
9.9.9.9
149.112.112.112
```

Recommended options:

```text
Enable DNSSEC: optional
Use private reverse DNS resolvers: optional
Cache size: default is fine
```

If available:

```text
Parallel requests: enabled
Fastest IP address: enabled
```

---

## Query Log

Recommended:

```text
Query log: enabled
```

This is the core visibility feature.

You can see:

* Which device requested which domain
* Time of request
* Whether it was blocked
* Which upstream handled it

---

## Blocklists

Recommended baseline AdGuard blocklist:

```text
https://adguardteam.github.io/HostlistsRegistry/assets/filter_1.txt
```

Optional stronger security list:

```text
https://adguardteam.github.io/HostlistsRegistry/assets/filter_11.txt
```

Do not add too many lists immediately.

Start small, verify stability, then tune.

---

## Custom Filtering Rules

Example block rule:

```text
||example-bad-domain.com^
```

This blocks:

```text
example-bad-domain.com
sub.example-bad-domain.com
```

---

# HTTPS / Encryption Notes

There are two different encryption questions:

1. **Client/admin web UI encryption**
2. **Upstream DNS encryption**

They are not the same.

---

## Upstream DNS Encryption

If AdGuard upstream DNS is:

```text
https://dns.quad9.net/dns-query
```

then AdGuard is using DoH to Quad9.

This means:

```text
AdGuard → Quad9 = encrypted
```

---

## Local Client DNS

Local devices normally talk to AdGuard using plain DNS:

```text
Client → 192.168.0.2:53
```

That is normal for a LAN DNS server.

---

## Admin Console HTTPS

AdGuard has encryption settings for:

* DNS-over-HTTPS server
* DNS-over-TLS server
* DNS-over-QUIC server
* HTTPS admin access

Those require certificates.

Settings may include:

```text
Enable encryption
Disable plain DNS
Server name
DNS-over-TLS port
DNS-over-QUIC port
Certificate path/content
Private key path/content
```

Recommended initial choice:

```text
Do not enable AdGuard server-side encryption yet
Do not disable plain DNS
Do not enable DoT/DoQ server mode yet
```

Why:

* Local-only admin access
* No internet exposure
* Certificate management adds complexity
* Disabling plain DNS can break clients
* Local DNS appliances normally serve LAN DNS over port 53

If HTTPS admin access is later desired, use a certificate and then remove HTTP access only after HTTPS is confirmed working.

---

# Hardening the Debian Appliance

## 1. Keep system updated

```bash
sudo apt update
sudo apt upgrade -y
```

---

## 2. Enable automatic security updates

Install:

```bash
sudo apt install unattended-upgrades -y
```

Configure:

```bash
sudo dpkg-reconfigure unattended-upgrades
```

Choose:

```text
Yes
```

---

## 3. Install Fail2ban

```bash
sudo apt install fail2ban -y
```

Start and enable:

```bash
sudo systemctl enable --now fail2ban
```

Check:

```bash
sudo systemctl status fail2ban
```

---

## 4. Disable unnecessary services

If installed, disable Avahi:

```bash
sudo systemctl disable avahi-daemon
```

If installed, disable CUPS:

```bash
sudo systemctl disable cups
```

If a service does not exist, that is fine.

---

## 5. Disable Wi-Fi if using Ethernet only

Find Wi-Fi interface:

```bash
ip a
```

Example Wi-Fi interface:

```text
<wifi-interface>
```

Disable temporarily:

```bash
sudo ip link set <wifi-interface> down
```

For long-term disabling, use BIOS/UEFI if available or blacklist the driver.

---

## 6. Keep Remote Exposure Closed

Do not port-forward to the AdGuard server.

Do not expose:

```text
22/tcp
53/tcp
53/udp
80/tcp
443/tcp
3000/tcp
```

to the public internet.

This appliance should be LAN-only.

---

## 7. Use Strong Passwords

Use a strong password for:

* Debian user
* SSH login
* AdGuard admin dashboard

Do not reuse passwords.

---

## 8. Physical Security

Recommended:

* Keep laptop plugged into Ethernet
* Keep lid closed
* Keep device out of sight
* Ensure ventilation
* Use reliable power
* Avoid daily browsing from this machine

---

# Troubleshooting

## AdGuard Service Status

```bash
sudo systemctl status AdGuardHome
```

Expected:

```text
active (running)
```

Restart:

```bash
sudo systemctl restart AdGuardHome
```

Start manually:

```bash
sudo systemctl start AdGuardHome
```

---

## View AdGuard Logs

```bash
sudo journalctl -u AdGuardHome -n 50 --no-pager
```

---

## Check Listening Ports

Check port 53:

```bash
sudo ss -tulnp | grep ':53'
```

Check port 80:

```bash
sudo ss -tulnp | grep ':80'
```

Check port 3000:

```bash
sudo ss -tulnp | grep ':3000'
```

Check common AdGuard ports:

```bash
sudo ss -tulnp | grep -E '53|80|443|3000'
```

---

## If Web UI Does Not Load

Try:

```text
http://192.168.0.2
```

If setup still uses port 3000:

```text
http://192.168.0.2:3000
```

From the server itself:

```text
http://127.0.0.1
```

or:

```text
http://127.0.0.1:3000
```

Check firewall:

```bash
sudo ufw status numbered
```

Make sure the UI port is allowed:

```bash
sudo ufw allow 80/tcp
sudo ufw allow 3000/tcp
```

---

## If DNS Breaks After Router DNS Change

Temporarily set router DNS back to public DNS:

```text
9.9.9.9
149.112.112.112
```

Then fix AdGuard.

Common checks:

```bash
sudo systemctl status AdGuardHome
sudo ss -tulnp | grep ':53'
nslookup google.com 127.0.0.1
```

Do not point clients to AdGuard until this works:

```bash
nslookup google.com 127.0.0.1
```

---

## If AdGuard Cannot Bind to Port 53

Check what is using port 53:

```bash
sudo ss -tulnp | grep ':53'
```

If nothing is shown, port 53 is free.

If something is using port 53, identify it from the process name.

On some systems, `systemd-resolved` may use port 53. If present, disable it:

```bash
sudo systemctl disable --now systemd-resolved
```

If the service does not exist, that is fine.

If needed, recreate `/etc/resolv.conf`:

```bash
sudo rm -f /etc/resolv.conf
echo "nameserver 9.9.9.9" | sudo tee /etc/resolv.conf
```

Restart AdGuard:

```bash
sudo systemctl restart AdGuardHome
```

---

## If AdGuard Install Breaks

Remove and reinstall.

Try uninstalling service:

```bash
sudo /opt/AdGuardHome/AdGuardHome -s uninstall
```

If that fails, use installer uninstall flag:

```bash
curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sudo sh -s -- -u
```

Reinstall:

```bash
curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sudo sh
```

If the installer specifically says to use reinstall mode:

```bash
curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sudo sh -s -- -r
```

---

## Permissions Warning

If logs show:

```text
unexpected permissions
perm = 0755 want 0700
```

Fix:

```bash
sudo chmod 700 /opt/AdGuardHome
sudo systemctl restart AdGuardHome
```

---

## If Static IP Did Not Apply

Check current IP:

```bash
ip a
```

If the server still has a DHCP address instead of `192.168.0.2`, check:

```bash
sudo nano /etc/network/interfaces
```

Make sure the correct interface name is used.

Example correct static block:

```text
auto <ethernet-interface>
iface <ethernet-interface> inet static
  address 192.168.0.2
  netmask 255.255.255.0
  gateway 192.168.0.1
  dns-nameservers 9.9.9.9 149.112.112.112
```

Make sure there is not also a DHCP block for the same interface:

```text
iface <ethernet-interface> inet dhcp
```

Reboot:

```bash
sudo reboot
```

---

## If Client Still Shows Old DNS

Windows:

```cmd
ipconfig /flushdns
ipconfig /release
ipconfig /renew
ipconfig /all
```

macOS:

```bash
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

Linux:

```bash
sudo resolvectl flush-caches
```

or reconnect networking.

---

# Final Recommended State

## Debian

```text
Debian 12 minimal
No desktop environment
SSH server installed
Standard system utilities installed
Static wired IP
Sleep disabled
Firewall enabled
Automatic security updates enabled
```

---

## Network

```text
AdGuard IP: 192.168.0.2
Gateway:    192.168.0.1
```

---

## UFW

Recommended active ports:

```text
22/tcp    SSH
53        DNS
80/tcp    AdGuard web UI
443/tcp   Reserved for future HTTPS
3000/tcp  Optional setup/fallback UI
```

After setup, if not using port 3000:

```bash
sudo ufw deny 3000/tcp
```

---

## AdGuard

```text
Admin Web Interface:
  Listen interface: All interfaces
  Port: 80

DNS Server:
  Listen interface: All interfaces
  Port: 53

DHCP:
  Disabled

Upstream DNS:
  https://dns.quad9.net/dns-query

Bootstrap DNS:
  9.9.9.9
  149.112.112.112
```

---

## Router DNS

Router DHCP DNS:

```text
192.168.0.2
```

---

## Verification Commands

From AdGuard server:

```bash
nslookup google.com 127.0.0.1
```

Check service:

```bash
sudo systemctl status AdGuardHome
```

Check listening DNS:

```bash
sudo ss -tulnp | grep ':53'
```

Check firewall:

```bash
sudo ufw status numbered
```

From Windows client:

```cmd
ipconfig /all
```

Expected DNS:

```text
192.168.0.2 (example static ip)
```

Then check:

```text
AdGuard Home → Query Log
```

You should see device-specific DNS traffic.

## Key Outcomes

* Deployed a fully functional DNS filtering appliance on Debian 12
* Implemented encrypted upstream DNS using Quad9 DNS-over-HTTPS
* Configured network-level DNS routing via router DHCP
* Applied system hardening and service minimization practices
* Documented full installation and troubleshooting workflow for reproducibility