# Troubleshooting Guide
> **Project:** OpenWrt Network Segmentation and VPN Architecture
> **Document Type:** Support-style troubleshooting reference
> **Release Type:** Public sanitized documentation

---

## 📑 Table of Contents

- [Troubleshooting Method](#troubleshooting-method)
- [Common Issues](#common-issues)
- [Safe Rollback Process](#safe-rollback-process)
- [Useful OpenWrt Commands](#useful-openwrt-commands)
- [Client-Side Commands](#client-side-commands)
- [Escalation Notes](#escalation-notes)

---

## Troubleshooting Method

When troubleshooting segmentation, test in layers:

```text
Physical link -> VLAN membership -> Interface IP -> DHCP -> Firewall zone -> Routing -> DNS -> VPN egress
```

Avoid changing several layers at once. Confirm each layer before moving to the next.

---

## Common Issues

### Issue 1: Wired Device Gets No IP Address

| Area | Check |
| --- | --- |
| Physical | Link light, cable, port selection |
| VLAN | Port is untagged/PVID member of Guest VLAN 20 |
| Interface | Guest interface uses `br-lan.20` |
| DHCP | DHCP server enabled on guest interface |
| Firewall | Guest zone allows DHCP to router |

Validation commands:

```bash
bridge vlan
ip addr show br-lan.20
logread | grep -i dhcp
```

Expected result:

```text
Wired client receives 10.40.20.x address.
```

---

### Issue 2: Wired Device Gets Trusted IP Instead of Guest IP

Likely causes:

- Wired port is still assigned to the trusted VLAN.
- VLAN filtering was not saved.
- Port PVID is wrong.
- Client is actually connected to trusted Wi-Fi instead of Ethernet.

Checks:

```bash
bridge vlan
```

Expected concept:

```text
lan-port-a 20 PVID Egress Untagged
lan-port-b 20 PVID Egress Untagged
```

Fix:

- Recheck bridge VLAN filtering.
- Confirm the correct physical port mapping.
- Renew the client DHCP lease.

---

### Issue 3: Guest Client Has IP but No Internet

Likely causes:

- Guest zone does not forward to VPN zone.
- VPN tunnel is down.
- Masquerading is not enabled on the VPN zone.
- DNS is failing.
- Guest zone was accidentally left without any egress path.

Checks:

```bash
wg show
ip route
uci show firewall | grep guest
nslookup example.com
ping 1.1.1.1
```

Interpretation:

| Result | Meaning |
| --- | --- |
| Ping to IP works, DNS fails | DNS issue |
| DNS works, no web traffic | Firewall/routing issue |
| VPN handshake missing | VPN tunnel issue |
| Guest can reach WAN directly | Kill-switch policy may be wrong |

---

### Issue 4: VPN Check Fails from Guest or IoT

Likely causes:

- WireGuard tunnel is disconnected.
- Guest/IoT forwarding points to WAN instead of VPN.
- Policy-based routing excludes the new interface.
- VPN provider DNS/routing is not applied to the new network.

Checks:

```bash
wg show
ip route
uci show firewall
```

Expected concept:

```text
guest -> vpn forwarding exists
iot -> vpn forwarding exists
guest -> wan forwarding does not exist when fail-closed policy is required
iot -> wan forwarding does not exist when fail-closed policy is required
```

---

### Issue 5: Guest or IoT Can Reach Trusted Devices

This is a security failure and should be corrected.

Likely causes:

- Guest/IoT forwarding to trusted zone exists.
- Firewall zone forward policy is too permissive.
- Device is actually on trusted network.
- Test target is not really in the trusted subnet.

Checks:

```bash
uci show firewall | grep forwarding
ipconfig /all
ip addr
```

Fix:

- Remove guest-to-trusted forwarding.
- Remove IoT-to-trusted forwarding.
- Keep low-trust forward policy set to REJECT.
- Retest from a fresh DHCP lease.

---

### Issue 6: Router Admin Page Loads from Guest or IoT

Likely causes:

- Low-trust zone input policy is ACCEPT.
- Admin service rule is too broad.
- Guest/IoT networks were added to the trusted zone by mistake.

Fix:

- Set guest and IoT input policy to REJECT.
- Allow only DHCP and DNS from low-trust zones.
- Confirm LuCI/SSH is reachable only from trusted network.

Expected result:

```text
Trusted -> LuCI/SSH: allowed
Guest -> LuCI/SSH: blocked
IoT -> LuCI/SSH: blocked
WAN -> LuCI/SSH: blocked
```

---

### Issue 7: DNS Works on Trusted but Not Guest or IoT

Likely causes:

- DNS allow rule missing for guest or IoT.
- DHCP is handing out the wrong DNS server.
- Future AdGuard resolver is unreachable from that VLAN.
- VPN DNS behavior does not include the new network.

Checks:

```bash
nslookup example.com
nslookup example.com 10.40.20.1
nslookup example.com 10.40.10.53
```

Fix options:

- Allow TCP/UDP 53 from low-trust zones to approved DNS resolver.
- Correct DHCP option 6.
- Confirm firewall permits DNS to the intended resolver.
- Confirm AdGuard is listening and reachable if integrated.

---

### Issue 8: IoT Device Cannot Function After Segmentation

Likely causes:

- Device depends on local discovery/multicast.
- Device expects phone/controller to be on the same subnet.
- Device does not work through VPN.
- Vendor application uses hardcoded DNS or unusual ports.

Security-conscious options:

- Keep device isolated and accept limited functionality.
- Add narrow allow rules only for required controller traffic.
- Use an IoT-specific exception list.
- Avoid broad IoT-to-trusted forwarding.

Do not solve this by allowing full IoT access into the trusted network.

---

### Issue 9: Locked Out After VLAN Changes

Likely causes:

- Management device was connected to a port that was reassigned.
- Trusted SSID was misconfigured.
- Bridge VLAN filtering removed the management path.
- Firewall policy blocked trusted admin access.

Recovery options:

1. Wait for LuCI rollback if using Save & Apply.
2. Reconnect using the trusted SSID or trusted port.
3. Use a previously exported OpenWrt backup.
4. Use OpenWrt failsafe/reset process if necessary.

Preventive controls:

- Keep a trusted management path active.
- Change one major network setting at a time.
- Export backups before bridge/firewall changes.
- Avoid reconfiguring the only path you are using to administer the router.

---

## Safe Rollback Process

Before major changes:

```text
1. Export backup.
2. Record current management IP.
3. Confirm trusted admin access.
4. Apply one change.
5. Validate.
6. Continue only after expected results are confirmed.
```

If a change fails:

```text
1. Stop making additional changes.
2. Reconnect through trusted management path.
3. Revert the last change.
4. Restore backup if needed.
5. Document the failure and likely cause.
```

---

## Useful OpenWrt Commands

```bash
bridge vlan
ip addr show
ip route
wg show
logread
logread | grep -i dhcp
cat /tmp/dhcp.leases
uci show network
uci show wireless
uci show firewall
/etc/init.d/network reload
/etc/init.d/firewall restart
```

Sanitize output before publishing.

---

## Client-Side Commands

### Windows

```cmd
ipconfig /all
ipconfig /release
ipconfig /renew
ipconfig /flushdns
tracert 1.1.1.1
nslookup example.com
```

### Linux/macOS

```bash
ip addr
ip route
dig example.com
nslookup example.com
traceroute 1.1.1.1
curl https://ifconfig.me
```

---

## Escalation Notes

For support-style documentation, capture:

- What network the client should be on.
- What IP address it actually received.
- Whether DNS works.
- Whether direct IP connectivity works.
- Whether the issue follows wired, wireless, or both.
- Whether the issue is isolated to one VLAN.
- Whether VPN status is healthy.
- What changed most recently.

Example ticket note format:

```text
Issue:
Guest wired client had valid DHCP lease but no internet.

Observed:
Client received 10.40.20.x/24 with gateway 10.40.20.1.
DNS lookup failed and VPN handshake was stale.

Action:
Restarted WireGuard interface, confirmed new handshake, retested DNS and web access.

Result:
Guest wired client restored internet through VPN egress. No guest-to-trusted access observed.
```