# Validation Plan
> **Project:** OpenWrt Network Segmentation and VPN Architecture
> **Document Type:** Test plan and verification checklist
> **Release Type:** Public sanitized documentation

---

## 📑 Table of Contents

- [Validation Goals](#validation-goals)
- [Pre-Change Checks](#pre-change-checks)
- [Client Placement Tests](#client-placement-tests)
- [Segmentation and Isolation Tests](#segmentation-and-isolation-tests)
- [VPN Egress Tests](#vpn-egress-tests)
- [DNS Tests](#dns-tests)
- [OpenWrt Router Verification](#openwrt-router-verification)
- [Evidence Checklist](#evidence-checklist)
- [Final Expected State](#final-expected-state)

---

## Validation Goals

Validation confirms that the project works as an implemented network design, not just as a set of configured options.

The validation process verifies:

- Clients receive addresses from the correct DHCP scope.
- Wired low-trust ports land in the guest VLAN.
- Wireless SSIDs map to the correct network zones.
- Guest and IoT networks cannot reach trusted devices.
- Low-trust networks egress through the WireGuard VPN tunnel.
- Router administration is not exposed to low-trust networks.
- DNS works now and can later be centralized through the AdGuard DNS project.

---

## Pre-Change Checks

Before changing VLAN or firewall settings:

| Check | Expected Result |
| --- | --- |
| OpenWrt backup exported | Backup archive saved offline |
| Trusted management path confirmed | Admin access works from trusted device |
| Upstream internet works | Router has WAN connectivity |
| Existing DHCP behavior documented | Known-good baseline captured |
| VPN baseline checked | WireGuard interface can connect |
| Rollback path understood | LuCI rollback or recovery method available |

---

## Client Placement Tests

### Test V-01: Trusted Wireless DHCP

Connect a trusted client to the trusted SSID.

Expected result:

```text
Client IP: 10.40.10.x
Default gateway: 10.40.10.1
Network role: Trusted
```

Windows check:

```cmd
ipconfig /all
```

Linux/macOS check:

```bash
ip addr
ip route
```

---

### Test V-02: Guest Wireless DHCP

Connect a test device to the guest SSID.

Expected result:

```text
Client IP: 10.40.20.x
Default gateway: 10.40.20.1
Network role: Guest
```

The client should not receive a trusted network address.

---

### Test V-03: Wired Guest Port DHCP

Connect a laptop or test device to a low-trust wired LAN port.

Expected result:

```text
Client IP: 10.40.20.x
Default gateway: 10.40.20.1
Network role: Guest wired access
```

This confirms that the wired port is an untagged/PVID member of the guest VLAN.

---

### Test V-04: IoT Wireless DHCP

Connect a test device to the IoT SSID.

Expected result:

```text
Client IP: 10.40.30.x
Default gateway: 10.40.30.1
Network role: IoT
```

---

## Segmentation and Isolation Tests

### Test V-05: Guest Cannot Reach Trusted Client

From a guest client, attempt to ping a trusted client.

```bash
ping 10.40.10.100
```

Expected result:

```text
Request timed out / unreachable
```

Passing this test helps confirm that guest-to-trusted lateral movement is blocked.

---

### Test V-06: IoT Cannot Reach Trusted Client

From an IoT client, attempt to reach a trusted client.

```bash
ping 10.40.10.100
```

Expected result:

```text
Request timed out / unreachable
```

---

### Test V-07: Guest Cannot Reach IoT

From a guest client, attempt to reach an IoT client.

```bash
ping 10.40.30.100
```

Expected result:

```text
Request timed out / unreachable
```

---

### Test V-08: Low-Trust Router Admin Block

From a guest or IoT client, attempt to open router administration services.

```text
http://10.40.20.1
https://10.40.20.1
ssh admin@10.40.20.1
```

Expected result:

```text
Blocked, refused, or unreachable
```

Low-trust networks may be allowed DHCP and DNS, but they should not be allowed router administration.

---

### Test V-09: Trusted Admin Access Still Works

From a trusted client, access OpenWrt administration.

Expected result:

```text
Trusted client can administer OpenWrt.
Guest and IoT clients cannot.
```

This confirms that management access remains available only from the trusted side.

---

## VPN Egress Tests

### Test V-10: Guest Uses VPN Egress

From a guest client, browse to a VPN provider status page or a public IP check service.

Expected result:

```text
Public IP: VPN provider egress IP
ISP IP: Not shown as direct client egress
```

Command-line option:

```bash
curl https://ifconfig.me
```

Do not publish the real output in the repository.

---

### Test V-11: IoT Uses VPN Egress

From an IoT client or test device on the IoT SSID, check public egress.

Expected result:

```text
IoT network exits through VPN policy.
```

---

### Test V-12: VPN Failure Does Not Fall Back to WAN

Temporarily disconnect or disable the WireGuard client during a controlled test window.

From guest or IoT, test internet access.

Expected result:

```text
Guest/IoT internet access fails closed instead of using direct WAN.
```

Restore the VPN immediately after testing.

---

## DNS Tests

### Test V-13: DNS Resolution Works

From each network, test DNS resolution.

```bash
nslookup example.com
```

or:

```bash
dig example.com
```

Expected result:

```text
DNS resolves successfully from each approved client network.
```

---

### Test V-14: DHCP DNS Assignment

Check DNS server assignment on each client.

Windows:

```cmd
ipconfig /all
```

Linux/macOS:

```bash
resolvectl status
```

Expected current-state result:

```text
Client receives the intended router or DNS resolver address by DHCP.
```

Expected future-state result after AdGuard integration:

```text
Client receives the Debian + AdGuard Home resolver address by DHCP.
```

---

### Test V-15: Future AdGuard Visibility

After the related Debian + AdGuard Home project is integrated, generate DNS traffic from each segment and confirm logs.

Expected result:

```text
AdGuard query log shows client DNS queries by source network/client.
```

---

## OpenWrt Router Verification

### Verify Bridge VLAN Membership

SSH to OpenWrt from a trusted client and run:

```bash
bridge vlan
```

Expected sanitized concept:

```text
lan-port-a 20 PVID Egress Untagged
lan-port-b 20 PVID Egress Untagged
```

Actual interface names vary by hardware and are not published.

---

### Verify Interfaces

```bash
ip addr show
```

Expected concept:

```text
br-lan.10 -> 10.40.10.1/24
br-lan.20 -> 10.40.20.1/24
br-lan.30 -> 10.40.30.1/24
wg-vpn -> WireGuard tunnel interface
```

---

### Verify DHCP Leases

```bash
cat /tmp/dhcp.leases
```

Expected result:

```text
Trusted clients appear in 10.40.10.x
Guest clients appear in 10.40.20.x
IoT clients appear in 10.40.30.x
```

Redact MAC addresses and hostnames before publishing evidence.

---

### Verify Firewall Configuration

```bash
uci show firewall
```

Expected concept:

```text
guest -> vpn forwarding exists
iot -> vpn forwarding exists
guest -> trusted forwarding does not exist
iot -> trusted forwarding does not exist
guest -> wan forwarding does not exist when VPN kill-switch behavior is required
iot -> wan forwarding does not exist when VPN kill-switch behavior is required
```

---

### Verify WireGuard Status

```bash
wg show
```

Expected result:

```text
latest handshake: recent
transfer: increasing counters
```

Do not publish keys, endpoints, or real peer information.

---

## Evidence Checklist

| Evidence Item | Status | Notes |
| --- | --- | --- |
| Sanitized architecture diagram | Ready | See `diagrams/architecture.mmd` |
| VLAN plan | Ready | See `configs/sanitized-network-plan.md` |
| DHCP validation | To capture | Redact MAC addresses and hostnames |
| Guest isolation test | To capture | Use sanitized target IPs |
| IoT isolation test | To capture | Use sanitized target IPs |
| VPN egress test | To capture | Redact public IP/provider-specific output |
| Firewall zone review | To capture | Redact sensitive custom names |
| DNS behavior | To expand | Link with Debian + AdGuard Home project |

---

## Final Expected State

The implementation is considered successful when:

```text
Trusted clients receive trusted subnet addresses.
Guest wireless clients receive guest subnet addresses.
Wired low-trust clients receive guest subnet addresses.
IoT clients receive IoT subnet addresses.
Guest and IoT clients cannot reach trusted clients.
Guest and IoT clients cannot access router administration.
Guest and IoT clients egress through the WireGuard VPN tunnel.
Guest and IoT clients do not fall back to direct WAN if VPN egress is unavailable.
The design can later use the Debian + AdGuard Home appliance for DNS visibility.
```