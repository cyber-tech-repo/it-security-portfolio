# Security Considerations
> **Project:** OpenWrt Network Segmentation and VPN Architecture
> **Document Type:** Security notes, limitations, and public sanitization policy
> **Release Type:** Public sanitized documentation

---

## 📑 Table of Contents

- [Security Goals](#security-goals)
- [Public Sanitization Policy](#public-sanitization-policy)
- [Segmentation Controls](#segmentation-controls)
- [Firewall Policy](#firewall-policy)
- [VPN Security Considerations](#vpn-security-considerations)
- [DNS Security Considerations](#dns-security-considerations)
- [Wireless Hardening](#wireless-hardening)
- [Administrative Access](#administrative-access)
- [Limitations](#limitations)
- [Future Hardening](#future-hardening)

---

## Security Goals

The goal of this project is to reduce unnecessary trust between device groups while keeping the network usable.

Primary security goals:

- Reduce lateral movement between trusted, guest, and IoT systems.
- Keep router administration limited to trusted clients.
- Prevent low-trust wired devices from landing on the trusted LAN.
- Route low-trust networks through controlled VPN egress.
- Prepare the environment for centralized DNS visibility and future monitoring.
- Publish technical evidence without exposing sensitive implementation details.

---

## Public Sanitization Policy

The public repository intentionally excludes:

- Exact router model.
- Exact physical interface names.
- Real SSIDs.
- Real wireless channels if they identify the live environment.
- Real private subnets and management addresses.
- Real public IP addresses.
- VPN provider name.
- WireGuard private keys, public keys, peer endpoints, and assigned tunnel addresses.
- MAC addresses.
- Internal hostnames.
- Screenshots containing identifying details.
- Serial numbers or ISP/account details.

The repository uses representative values such as:

```text
10.40.10.0/24
10.40.20.0/24
10.40.30.0/24
VLAN 10 / 20 / 30
lab-main / lab-guest / lab-iot
```

These are documentation examples, not sensitive production values.

---

## Segmentation Controls

### Trusted Network

The trusted network is used for administrative and personal devices.

Security intent:

- Permit router administration from trusted devices.
- Allow normal internet access.
- Allow future access to infrastructure services such as AdGuard, SIEM, or management hosts.
- Avoid allowing guest or IoT devices to initiate connections into trusted systems.

### Guest Network

The guest network is used for visitors, temporary devices, work devices, or other systems that should not be trusted.

Security intent:

- Provide internet access.
- Prevent access to trusted devices.
- Prevent access to router administration.
- Route through VPN egress.
- Allow only required router services such as DHCP and DNS.

### IoT Network

The IoT network is used for low-trust smart devices.

Security intent:

- Contain devices with weaker update practices or unknown behavior.
- Prevent access to trusted devices.
- Prevent access to guest devices.
- Route through VPN egress when compatible.
- Keep IoT compatibility separate from stronger trusted-network wireless settings.

---

## Firewall Policy

VLANs provide separation at Layer 2, but firewall zones enforce policy at Layer 3.

### Deny-by-Default Concept

Low-trust networks use a restrictive approach:

```text
Input: REJECT by default
Forward: REJECT by default
Output: ACCEPT as needed
```

Then specific rules allow required services:

```text
Guest -> Router DHCP: allowed
Guest -> Router DNS: allowed
IoT -> Router DHCP: allowed
IoT -> Router DNS: allowed
```

### Blocked Paths

```text
Guest -> Trusted: blocked
IoT -> Trusted: blocked
Guest -> IoT: blocked
IoT -> Guest: blocked
Guest -> Router admin: blocked
IoT -> Router admin: blocked
```

### Allowed Paths

```text
Trusted -> WAN/VPN: allowed by policy
Guest -> VPN: allowed
IoT -> VPN: allowed
```

Guest and IoT are not given direct WAN forwarding when VPN fail-closed behavior is required.

---

## VPN Security Considerations

The WireGuard tunnel is used as a routed egress path.

Important choices:

- The VPN interface is not bridged into LAN.
- Low-trust zones forward to the VPN zone.
- Low-trust zones do not forward directly to WAN when fail-closed behavior is desired.
- Provider details and keys are not published.
- VPN status is validated with handshake and egress testing.

### VPN Trade-Offs

A VPN does not automatically make a network secure. It changes the egress path and shifts some trust from the ISP/upstream network to the VPN provider.

VPN routing is useful here because it:

- Gives low-trust networks a controlled egress path.
- Supports privacy goals for guest/IoT traffic.
- Provides a practical firewall policy exercise.
- Creates a clear validation requirement.

---

## DNS Security Considerations

DNS is handled in phases.

### Current-State DNS

Initial segmentation validates that each network can resolve DNS through the intended router/VPN path.

### Future-State DNS

The related Debian + AdGuard Home project adds:

- DNS query visibility.
- Filtering.
- Centralized DNS policy.
- Encrypted upstream DNS.

### DNS Limitations

DNS visibility does not automatically prevent bypass.

Potential bypass paths include:

- Client-configured public DNS.
- DNS-over-HTTPS inside browsers or applications.
- Hardcoded DNS behavior in IoT devices.
- VPN clients running directly on endpoints.

Future firewall rules can improve enforcement by:

- Blocking outbound port 53 except to the approved resolver.
- Redirecting DNS to the approved resolver where appropriate.
- Reviewing DNS-over-HTTPS behavior.
- Logging DNS requests centrally.

---

## Wireless Hardening

Wireless settings are designed around separation and compatibility.

| Control | Status |
| --- | --- |
| WPS | Disabled |
| Remote administration | Disabled |
| Guest isolation | Enabled where supported |
| Trusted SSID | Separate from guest and IoT |
| IoT SSID | Separate lower-trust network |
| WPA2/WPA3 | Used where supported |
| WPA2-PSK AES | Used for IoT compatibility when needed |

Example channels in this repository are arbitrary publication values:

```text
2.4 GHz: channel 6, 20 MHz
5 GHz: channel 44, 80 MHz
```

These values are included to show documentation completeness, not to reveal the live RF environment.

---

## Administrative Access

Administrative access is intentionally limited.

Allowed:

```text
Trusted network -> OpenWrt LuCI
Trusted network -> OpenWrt SSH
```

Blocked:

```text
Guest network -> OpenWrt LuCI
Guest network -> OpenWrt SSH
IoT network -> OpenWrt LuCI
IoT network -> OpenWrt SSH
WAN -> OpenWrt administration
```

Recommended practices:

- Use strong admin credentials.
- Keep OpenWrt updated.
- Disable WAN administration.
- Use SSH keys where practical.
- Export backups before major changes.
- Keep a known trusted recovery path.

---

## Limitations

This project intentionally remains scoped to an OpenWrt router lab.

Known limitations:

- No managed switch is included yet.
- No dedicated pfSense firewall is included yet.
- No full packet capture or IDS sensor is included yet.
- DNS enforcement is not complete until the DNS project and firewall rules are integrated.
- Consumer and embedded hardware may have port/interface naming differences.
- Some IoT devices may break when forced through VPN egress.
- VLAN segmentation requires correct firewall rules to become a meaningful security boundary.

---

## Future Hardening

Planned improvements:

1. Add a managed switch for trunk/access port design.
2. Add pfSense for advanced firewall policy and inter-VLAN routing.
3. Add a dedicated management VLAN.
4. Integrate the Debian + AdGuard Home DNS appliance.
5. Enforce DNS policy at the firewall.
6. Add Security Onion for network visibility and IDS.
7. Add Wazuh for endpoint logs and alerting.
8. Add automated validation scripts for DHCP, DNS, VPN, and segmentation checks.
9. Version sanitized configuration exports for change tracking.