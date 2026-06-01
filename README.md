# IT & Cybersecurity Portfolio
  
Public portfolio of hands-on IT infrastructure, networking, systems administration, and cybersecurity projects.

This repository documents practical lab work focused on building, configuring, validating, troubleshooting, and securing self-hosted infrastructure. The projects are written to demonstrate real implementation experience across LAN/WAN networking, VLANs, DNS, DHCP, VPNs, Linux administration, firewall policy, monitoring, endpoint visibility, and security automation.

Most projects originate from larger private lab environments and are reviewed before publication to remove credentials, real network details, private keys, device identifiers, and unsafe operational configurations.
  
---

## About Me

I am an IT infrastructure and cybersecurity professional focused on infrastructure support, network administration, systems administration, and security operations.

My background includes hands-on work with:

- LAN/WAN/VLAN troubleshooting
- Windows and Linux systems
- DNS, DHCP, VPNs, routers, switches, and firewalls
- Active Directory and endpoint administration
- SIEM monitoring and alert investigation
- Security-focused documentation and validation

This portfolio supports my progression toward network administration, systems administration, security operations, and security engineering roles.

---

## Featured Projects

| Project | Focus Areas | Status |
| --- | --- | --- |
| [OpenWrt Network Segmentation and VPN Architecture](projects/openwrt-network-segmentation-vpn/) | VLANs, DHCP, wireless segmentation, guest/IoT isolation, firewall zones, WireGuard VPN routing | Published |
| [Debian 12 + AdGuard Home Secure DNS Appliance](projects/debian12-adguard-home-secure-dns-appliance/) | Debian 12, AdGuard Home, DNS filtering, UFW hardening, Quad9 DNS-over-HTTPS | Published |

For the full project roadmap, see the [projects directory](projects/).

---

## Technical Focus Areas

```mermaid
flowchart LR

    Support[IT Support]
    Networking[Network Administration]
    Systems[Systems Administration]
    SecurityOps[Security Operations]
    Engineering[Security Engineering]

    Support --> Networking
    Networking --> Systems
    Systems --> SecurityOps
    SecurityOps --> Engineering
```

What This Portfolio Demonstrates
--------------------------------

This portfolio is designed to show practical ability in:

-   Building and documenting self-hosted infrastructure
  
-   Designing segmented network environments
  
-   Deploying Linux-based network services
  
-   Applying least-privilege and isolation concepts
  
-   Validating configurations through testing
  
-   Troubleshooting realistic infrastructure issues
  
-   Writing clear technical documentation for long-term maintainability

Repository Structure
--------------------

    .
    ├── README.md
    └── projects/
        ├── README.md
        ├── openwrt-network-segmentation-vpn/
        └── debian12-adguard-home-secure-dns-appliance/
  

Documentation and Sanitization
------------------------------

Public documentation is intentionally sanitized.

This repository does not publish credentials, private keys, real SSIDs, real internal addressing, public IP addresses, MAC addresses, hostnames, exact hardware identifiers, or sensitive operational details.

Example values are used where needed to preserve the technical design while keeping the public release safe.

Profiles
-------

-   LinkedIn: https://www.linkedin.com/in/zachary-moseder
  
-   GitHub: https://www.github.com/cyber-tech-repo

-   TryHackMe: https://www.tryhackme.com/p/zachary.techy

-   CodeCademy: https://www.codecademy.com/profiles/zachary.techy