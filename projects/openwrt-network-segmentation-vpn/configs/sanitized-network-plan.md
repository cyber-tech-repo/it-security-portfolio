# Sanitized Network Plan
> **Project:** OpenWrt Network Segmentation and VPN Architecture
> **Document Type:** Public-safe network design and reference configuration
> **Important:** These are example values, not live values.

---

## 📑 Table of Contents

- [Purpose](#purpose)
- [Zone Plan](#zone-plan)
- [VLAN Plan](#vlan-plan)
- [DHCP Plan](#dhcp-plan)
- [Wireless Plan](#wireless-plan)
- [Firewall Forwarding Matrix](#firewall-forwarding-matrix)
- [DNS Plan](#dns-plan)
- [UCI-Style Reference Snippets](#uci-style-reference-snippets)
- [Do Not Publish](#do-not-publish)

---

## Purpose

This file provides a sanitized reference plan for the project. It is meant to show design clarity and implementation structure without exposing sensitive operational details.

Do not paste the UCI snippets blindly into a live router. OpenWrt device names vary by hardware, release, and switch/DSA layout.

---

## Zone Plan

| Zone | Trust Level | Purpose | Example Subnet | Management Access |
| --- | --- | --- | --- | --- |
| Trusted | High | Admin and personal devices | `10.40.10.0/24` | Allowed |
| Guest | Low | Visitors, work devices, wired low-trust ports | `10.40.20.0/24` | Blocked |
| IoT | Low | Smart devices and embedded devices | `10.40.30.0/24` | Blocked |
| WAN | Untrusted | Upstream internet path | DHCP from upstream | Blocked |
| VPN | External | WireGuard egress path | Provider assigned | Blocked |

---

## VLAN Plan

| VLAN ID | Interface | Gateway | Role |
| ---: | --- | --- | --- |
| 10 | `trusted` | `10.40.10.1` | Trusted/admin network |
| 20 | `guest` | `10.40.20.1` | Guest Wi-Fi and wired low-trust access ports |
| 30 | `iot` | `10.40.30.1` | IoT network |

### Physical Port Concept

| Port Role | Membership | Notes |
| --- | --- | --- |
| WAN | Separate from LAN bridge | Do not bridge into LAN |
| LAN access port A | VLAN 20 untagged/PVID | Low-trust wired guest access |
| LAN access port B | VLAN 20 untagged/PVID | Low-trust wired guest access |
| WireGuard tunnel | Not a bridge member | Routed through firewall policy |

---

## DHCP Plan

| Interface | Start | Limit | Example Range | Lease Time |
| --- | ---: | ---: | --- | --- |
| Trusted | 100 | 100 | `10.40.10.100-10.40.10.199` | `12h` |
| Guest | 100 | 100 | `10.40.20.100-10.40.20.199` | `12h` |
| IoT | 100 | 100 | `10.40.30.100-10.40.30.199` | `12h` |

---

## Wireless Plan

| SSID | Network | Band | Channel | Width | Security | Notes |
| --- | --- | --- | ---: | --- | --- | --- |
| `lab-main` | Trusted | 5 GHz | 44 | 80 MHz | WPA2/WPA3 mixed | Admin and trusted clients |
| `lab-guest` | Guest | 5 GHz | 44 | 80 MHz | WPA2/WPA3 mixed | Client isolation enabled |
| `lab-iot` | IoT | 2.4 GHz | 6 | 20 MHz | WPA2-PSK AES | Compatibility network |

Additional wireless hardening:

```text
WPS: disabled
Remote admin over WAN: disabled
Guest isolation: enabled where supported
SSID names: sanitized examples only
Passphrases: never published
```

---

## Firewall Forwarding Matrix

| Source | Trusted | Guest | IoT | WAN | VPN | Router Services |
| --- | --- | --- | --- | --- | --- | --- |
| Trusted | Local | Blocked by default | Blocked by default | Optional | Optional | Admin allowed |
| Guest | Blocked | Local | Blocked | Blocked | Allowed | DHCP/DNS only |
| IoT | Blocked | Blocked | Local | Blocked | Allowed | DHCP/DNS only |
| WAN | Blocked | Blocked | Blocked | Local | Blocked | None |
| VPN | Return traffic only | Return traffic only | Return traffic only | N/A | Local | None |

---

## DNS Plan

### Current State

```text
Clients receive DNS through router-controlled DHCP behavior.
Low-trust DNS follows the same egress policy as the client network.
```

### Future State with Debian + AdGuard Home

```text
AdGuard example address: 10.40.10.53
DHCP option 6: 10.40.10.53
```

Expected flow:

```text
Client -> AdGuard Home -> encrypted upstream DNS -> Internet
```

Firewall rules should later restrict or redirect unauthorized DNS attempts.

---

## UCI-Style Reference Snippets

These snippets are intentionally generic. They show the implementation pattern without publishing a live configuration.

### `/etc/config/network` Concept

```text
config interface 'trusted'
option proto 'static'
option device 'br-lan.10'
option ipaddr '10.40.10.1'
option netmask '255.255.255.0'

config interface 'guest'
option proto 'static'
option device 'br-lan.20'
option ipaddr '10.40.20.1'
option netmask '255.255.255.0'

config interface 'iot'
option proto 'static'
option device 'br-lan.30'
option ipaddr '10.40.30.1'
option netmask '255.255.255.0'

config interface 'wg_vpn'
option proto 'wireguard'
option private_key '<redacted>'
list addresses '<provider-assigned-tunnel-address>'
```

### Bridge VLAN Filtering Concept

```text
config bridge-vlan
option device 'br-lan'
option vlan '20'
list ports '<lan-port-a>:u*'
list ports '<lan-port-b>:u*'
```

Notes:

```text
:u* means untagged PVID in OpenWrt bridge VLAN notation.
Actual port names are hardware-specific and are not published.
WAN is not included.
WireGuard is not included.
```

### `/etc/config/dhcp` Concept

```text
config dhcp 'trusted'
option interface 'trusted'
option start '100'
option limit '100'
option leasetime '12h'

config dhcp 'guest'
option interface 'guest'
option start '100'
option limit '100'
option leasetime '12h'

config dhcp 'iot'
option interface 'iot'
option start '100'
option limit '100'
option leasetime '12h'
```

Future AdGuard DNS option example:

```text
list dhcp_option '6,10.40.10.53'
```

### `/etc/config/firewall` Zone Concept

```text
config zone
option name 'trusted'
list network 'trusted'
option input 'ACCEPT'
option output 'ACCEPT'
option forward 'REJECT'

config zone
option name 'guest'
list network 'guest'
option input 'REJECT'
option output 'ACCEPT'
option forward 'REJECT'

config zone
option name 'iot'
list network 'iot'
option input 'REJECT'
option output 'ACCEPT'
option forward 'REJECT'

config zone
option name 'vpn'
list network 'wg_vpn'
option input 'REJECT'
option output 'ACCEPT'
option forward 'REJECT'
option masq '1'
option mtu_fix '1'
```

### Forwarding Concept

```text
config forwarding
option src 'guest'
option dest 'vpn'

config forwarding
option src 'iot'
option dest 'vpn'
```

No low-trust forwarding to trusted:

```text
Guest -> Trusted: not configured
IoT -> Trusted: not configured
```

No low-trust direct WAN forwarding when VPN fail-closed behavior is required:

```text
Guest -> WAN: not configured
IoT -> WAN: not configured
```

### DHCP and DNS Allow Rules for Low-Trust Zones

```text
config rule
option name 'Allow-Guest-DHCP'
option src 'guest'
option proto 'udp'
option dest_port '67-68'
option target 'ACCEPT'

config rule
option name 'Allow-Guest-DNS'
option src 'guest'
option proto 'tcp udp'
option dest_port '53'
option target 'ACCEPT'

config rule
option name 'Allow-IoT-DHCP'
option src 'iot'
option proto 'udp'
option dest_port '67-68'
option target 'ACCEPT'

config rule
option name 'Allow-IoT-DNS'
option src 'iot'
option proto 'tcp udp'
option dest_port '53'
option target 'ACCEPT'
```

---

## Do Not Publish

Never publish:

```text
/etc/config/network full live export
/etc/config/wireless full live export
/etc/config/firewall full live export
WireGuard private keys
WireGuard peer details
VPN endpoint
Public IP address
Real SSIDs
Real passphrases
Real MAC addresses
Router serial number
ISP account details
Screenshots with identifiable client names
```