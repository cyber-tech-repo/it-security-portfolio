# Implementation Guide
> **Project:** OpenWrt Network Segmentation and VPN Architecture
> **Document Type:** Technical implementation notes
> **Release Type:** Public sanitized documentation

---

## 📑 Table of Contents

- [Implementation Scope](#implementation-scope)
- [Sanitized Values Used](#sanitized-values-used)
- [Phase 1: Baseline Router Role](#phase-1-baseline-router-role)
- [Phase 2: VLAN and Bridge Design](#phase-2-vlan-and-bridge-design)
- [Phase 3: Interface Configuration](#phase-3-interface-configuration)
- [Phase 4: DHCP Configuration](#phase-4-dhcp-configuration)
- [Phase 5: Wireless Segmentation](#phase-5-wireless-segmentation)
- [Phase 6: Firewall Zone Policy](#phase-6-firewall-zone-policy)
- [Phase 7: WireGuard VPN Integration](#phase-7-wireguard-vpn-integration)
- [Phase 8: DNS Design](#phase-8-dns-design)
- [Phase 9: Backup and Change Control](#phase-9-backup-and-change-control)
- [Implementation Evidence](#implementation-evidence)

---

## Implementation Scope

This implementation documents an OpenWrt router used as a routed segmentation device. The router provides:

- WAN uplink to an upstream gateway.
- VLAN-backed internal networks.
- DHCP service for each internal network.
- Wireless SSIDs mapped to separate logical networks.
- Low-trust wired access ports assigned to the guest VLAN.
- Firewall zones controlling traffic between network segments.
- WireGuard VPN egress for guest and IoT networks.

This is not a direct backup of the router configuration. It is a sanitized technical representation suitable for public portfolio use.

---

## Sanitized Values Used

| Item | Public Example Value | Notes |
| --- | --- | --- |
| Trusted VLAN | `10` | Administrative and trusted personal devices |
| Guest VLAN | `20` | Guest Wi-Fi and low-trust wired ports |
| IoT VLAN | `30` | Smart devices and other low-trust clients |
| Trusted subnet | `10.40.10.0/24` | Example only |
| Guest subnet | `10.40.20.0/24` | Example only |
| IoT subnet | `10.40.30.0/24` | Example only |
| Trusted gateway | `10.40.10.1` | Router interface IP |
| Guest gateway | `10.40.20.1` | Router interface IP |
| IoT gateway | `10.40.30.1` | Router interface IP |
| VPN interface | `wg-vpn` | Generic WireGuard client name |
| Trusted SSID | `lab-main` | Example only |
| Guest SSID | `lab-guest` | Example only |
| IoT SSID | `lab-iot` | Example only |

---

## Phase 1: Baseline Router Role

The OpenWrt device was configured as a routed firewall/router, not as a simple access point.

### Design Decisions

| Decision | Reason |
| --- | --- |
| Keep WAN separate from LAN bridge | Prevents upstream/WAN traffic from being bridged into internal networks |
| Use router mode instead of AP mode | Allows DHCP, routing, firewall zones, VPN policy, and segmentation |
| Keep VPN as routed interface | WireGuard should be routed/firewalled, not bridged into LAN |
| Maintain trusted management path | Prevents lockout while changing bridge VLAN settings |

### Important Rule

The WAN interface and WireGuard tunnel are **not** bridge members.

```text
Do not add WAN to br-lan.
Do not add WireGuard to br-lan.
Use routing and firewall policy instead.
```

---

## Phase 2: VLAN and Bridge Design

OpenWrt bridge VLAN filtering was used to create VLAN-backed logical networks.

| VLAN | Device | Physical/Wireless Use | Purpose |
| ---: | --- | --- | --- |
| 10 | `br-lan.10` | Trusted SSID | Trusted client and admin network |
| 20 | `br-lan.20` | Guest SSID + low-trust wired LAN ports | Guest and untrusted wired access |
| 30 | `br-lan.30` | IoT SSID | Smart devices and low-trust endpoints |

### Wired Access Port Behavior

The sanitized design assigns available wired LAN access ports to Guest VLAN 20 as untagged/PVID ports.

| Physical Role | VLAN Membership | Purpose |
| --- | --- | --- |
| WAN port | Not a LAN bridge member | Upstream internet connection |
| LAN access port A | VLAN 20 untagged/PVID | Wired guest/low-trust client |
| LAN access port B | VLAN 20 untagged/PVID | Wired guest/low-trust client |

This means a normal device plugged into a low-trust LAN port does not need VLAN support. The router classifies untagged ingress traffic into the guest VLAN.

---

## Phase 3: Interface Configuration

Each VLAN-backed device was assigned to a logical OpenWrt interface.

| Interface | Device | Protocol | Address | Gateway Field |
| --- | --- | --- | --- | --- |
| `trusted` | `br-lan.10` | Static | `10.40.10.1/24` | Blank |
| `guest` | `br-lan.20` | Static | `10.40.20.1/24` | Blank |
| `iot` | `br-lan.30` | Static | `10.40.30.1/24` | Blank |
| `wan` | WAN device | DHCP client | Upstream assigned | Upstream assigned |
| `wg-vpn` | WireGuard tunnel | WireGuard | Provider assigned | Tunnel route |

### Why Internal Gateways Are Blank

The VLAN interfaces are gateway interfaces for their client networks. They do not need another gateway configured inside the interface definition.

```text
Client default gateway: 10.40.20.1
Router guest interface: 10.40.20.1
Interface gateway field: blank
```

The router decides the next hop using its routing table, firewall zones, and VPN policy.

---

## Phase 4: DHCP Configuration

OpenWrt DHCP service was enabled separately for each internal network.

| Network | DHCP Start | DHCP Limit | Example Lease Range | Lease Time |
| --- | ---: | ---: | --- | --- |
| Trusted | 100 | 100 | `10.40.10.100-10.40.10.199` | `12h` |
| Guest | 100 | 100 | `10.40.20.100-10.40.20.199` | `12h` |
| IoT | 100 | 100 | `10.40.30.100-10.40.30.199` | `12h` |

### DHCP Design Rationale

- DHCP ranges make client placement easy to confirm during troubleshooting.
- Each segment uses a predictable gateway address ending in `.1`.
- Static reservations can be added later for infrastructure services.
- DHCP DNS can later be pointed to the Debian + AdGuard Home DNS appliance.

---

## Phase 5: Wireless Segmentation

Wireless SSIDs were mapped to their corresponding OpenWrt networks.

| SSID | Network Interface | Example Band | Example Radio Settings | Security |
| --- | --- | --- | --- | --- |
| `lab-main` | `trusted` | 5 GHz | Channel 44, 80 MHz | WPA2/WPA3 mixed |
| `lab-guest` | `guest` | 5 GHz | Channel 44, 80 MHz | WPA2/WPA3 mixed, client isolation |
| `lab-iot` | `iot` | 2.4 GHz | Channel 6, 20 MHz | WPA2-PSK AES |

### Wireless Hardening Choices

- WPS disabled.
- Remote administration disabled.
- Guest client isolation enabled where supported.
- IoT devices placed on a separate SSID and VLAN-backed interface.
- Legacy-compatible IoT security kept separate from trusted devices.
- SSIDs and passphrases are not published in the repository.

---

## Phase 6: Firewall Zone Policy

OpenWrt firewall zones were used to control routing between networks.

| Zone | Covered Network | Input | Output | Forward | Masquerading | Intended Forwarding |
| --- | --- | --- | --- | --- | --- | --- |
| `trusted` | `trusted` | ACCEPT | ACCEPT | REJECT | No | To WAN or VPN as policy allows |
| `guest` | `guest` | REJECT | ACCEPT | REJECT | No | To VPN only |
| `iot` | `iot` | REJECT | ACCEPT | REJECT | No | To VPN only |
| `wan` | `wan` | REJECT | ACCEPT | REJECT | Yes | Internet uplink |
| `vpn` | `wg-vpn` | REJECT | ACCEPT | REJECT | Yes | VPN tunnel egress |

### Required Router Services for Low-Trust Networks

Guest and IoT clients still require limited access to the router for DHCP and DNS.

| Rule | Source Zone | Destination Port | Protocol | Purpose |
| --- | --- | --- | --- | --- |
| Allow DHCP Guest | `guest` | 67/68 | UDP | Client address assignment |
| Allow DNS Guest | `guest` | 53 | TCP/UDP | DNS resolution through router or future DNS policy |
| Allow DHCP IoT | `iot` | 67/68 | UDP | Client address assignment |
| Allow DNS IoT | `iot` | 53 | TCP/UDP | DNS resolution through router or future DNS policy |

### Blocked From Low-Trust Networks

Low-trust zones are not allowed to access:

- Router LuCI/admin web interface.
- SSH management.
- Trusted subnet clients.
- Other low-trust VLANs.
- Infrastructure services unless explicitly allowed later.

---

## Phase 7: WireGuard VPN Integration

WireGuard was configured as a router-level client tunnel. Provider-specific details are not published.

### VPN Design

| Component | Public Description |
| --- | --- |
| VPN type | WireGuard client |
| Interface name | `wg-vpn` |
| Provider | Redacted |
| Public endpoint | Redacted |
| Private key | Redacted |
| Peer public key | Redacted |
| Allowed IPs | Default route policy for selected low-trust zones |

### Routing Policy

| Source Zone | Normal WAN Forwarding | VPN Forwarding | Result |
| --- | --- | --- | --- |
| Trusted | Optional | Optional | Can be policy-routed as needed |
| Guest | No | Yes | Guest clients egress through VPN |
| IoT | No | Yes | IoT clients egress through VPN |

### VPN Kill-Switch Concept

Guest and IoT zones are not given direct forwarding to the WAN zone. This prevents low-trust networks from silently falling back to direct WAN egress if the VPN is unavailable.

```text
Guest -> VPN: allowed
Guest -> WAN: not allowed
IoT -> VPN: allowed
IoT -> WAN: not allowed
```

---

## Phase 8: DNS Design

DNS was designed in two stages.

### Stage 1: Router/VPN DNS

During the initial segmentation build, DHCP clients used router-provided DNS behavior. Low-trust networks followed the same egress path as their traffic policy.

### Stage 2: Debian + AdGuard Home Integration

A later related project introduces a Debian 12 + AdGuard Home DNS appliance for centralized visibility and filtering.

Future DHCP DNS option example:

```text
DHCP option 6: 10.40.10.53
```

Expected future DNS path:

```text
Client -> OpenWrt DHCP DNS assignment -> Debian AdGuard Home -> Encrypted upstream resolver -> Internet
```

### DNS Security Notes

- Guest and IoT DNS should be explicitly controlled before claiming full DNS enforcement.
- DNS visibility does not automatically prevent DNS-over-HTTPS bypass.
- Firewall rules can later restrict outbound DNS and redirect client DNS to the approved resolver.

---

## Phase 9: Backup and Change Control

Before making VLAN and firewall changes, a recovery plan was used.

### Change Safety Checklist

- Export OpenWrt backup before major changes.
- Keep a trusted management path active.
- Apply one major change at a time.
- Use LuCI rollback behavior when possible.
- Validate DHCP and connectivity after each major change.
- Document expected and actual results.
- Avoid relying on the ports being actively reconfigured for management access.

### Public Repository Sanitization

Before publishing, the following were removed or generalized:

- Hardware model.
- Actual SSIDs.
- Actual subnets and management addresses.
- VPN provider and tunnel endpoint.
- WireGuard keys.
- MAC addresses.
- Hostnames.
- Screenshots containing identifiable details.

---

## Implementation Evidence

Evidence to capture for this type of project includes:

| Evidence | Example Command or Source | Sanitization Required |
| --- | --- | --- |
| VLAN membership | `bridge vlan` | Replace real port names if needed |
| Interface status | `ip addr show` | Redact MAC addresses and actual IPs if sensitive |
| DHCP leases | LuCI DHCP leases or `/tmp/dhcp.leases` | Redact client names and MAC addresses |
| Firewall zones | LuCI firewall page or `uci show firewall` | Redact custom hostnames |
| VPN status | `wg show` | Redact keys, endpoints, and public IPs |
| DNS behavior | `nslookup`, `dig`, AdGuard query logs later | Redact domains if personal |
| Isolation tests | ping, traceroute, browser access attempts | Use sanitized targets |

The goal is to show that the network was built, tested, and validated without exposing sensitive operational details.