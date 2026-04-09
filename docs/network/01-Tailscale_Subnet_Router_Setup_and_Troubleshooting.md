# Tailscale Subnet Router — Setup and Troubleshooting

**Document Type:** Admin Technical Reference  
**Author:** Will Chang, Tailscale Customer Success Engineer  
**Audience:** IT Administrator / Tailscale CSE Reference  
**Last Updated:** April 2026  
**Repository:** https://github.com/willshchang/WSHC-Entra-IaC-Zero-Trust-Lab  
**Official Reference:** https://tailscale.com/docs/features/subnet-routers

---

## Overview

This document covers the complete setup, verification, and 
troubleshooting process for Tailscale subnet routers in a 
site-to-site topology. It is written as a learning journal — 
including commands that didn't work, why they failed, and 
what the correct approach was.

**Lab topology:**
- **Site A (Cloud):** Azure VM (`tinyco-vm`) — Ubuntu 24.04, 
  Tailscale IP `100.93.4.6`
- **Site B (Home LAN):** Apple TV (`iwilltvlivingroom`) — 
  subnet router advertising `192.168.1.0/24`, 
  Tailscale IP `100.109.140.74`
- **Home network:** Telus fibre modem (`192.168.1.254`) → 
  ASUS router in AP mode (`192.168.1.59`)

**Goal:** Azure VM (Site A) reaches non-Tailscale devices on 
the home LAN (`192.168.1.0/24`) through the Apple TV subnet 
router (Site B).

---

## What is a Subnet Router?

A subnet router is a Tailscale device that acts as a gateway 
between your Tailnet and a physical subnet — allowing Tailscale 
devices to reach non-Tailscale devices on that subnet without 
installing Tailscale on every device.

**Key distinction:**
- **Exit node** — routes all outbound internet traffic through 
  a device (like a VPN)
- **Subnet router** — routes traffic to specific private subnets 
  only, does not affect internet traffic

**Why it matters:**
Devices like printers, routers, modems, cameras, and legacy 
systems can't run Tailscale. A subnet router extends Tailscale's 
Zero Trust security model to cover these devices without 
requiring client installation.

---

## Lab Network Explained

Before troubleshooting subnet routing, understand your network 
topology:

```bash
ipconfig          # Windows
ifconfig          # macOS/Linux
```

**Key terms:**

| Term | Definition |
|---|---|
| **Default Gateway** | The device that routes traffic from your LAN to the internet — usually your router or modem |
| **Subnet** | A range of IP addresses on the same local network |
| **AP Mode** | Access Point mode — device provides WiFi only, does not route traffic |
| **SNAT** | Source NAT — subnet router rewrites source IP so return traffic routes correctly |

**Lab network breakdown:**
- `192.168.1.254` — Telus fibre modem (actual default gateway)
- `192.168.1.59` — ASUS router in AP mode (WiFi only, not routing)
- `192.168.1.157` — Windows PC (Tailscale device, direct Tailnet member)
- `192.168.1.0/24` — full home subnet advertised by Apple TV

> **Note:** When the ASUS router is in AP mode, the modem 
> (`192.168.1.254`) is the real gateway — not the router's 
> typical `192.168.1.1` address. Always verify with `ipconfig` 
> before assuming gateway IP.

---

## Setup — Apple TV as Subnet Router

### tvOS Configuration

Apple TV (tvOS) is one of the simplest subnet router platforms — 
no CLI required, no IP forwarding configuration needed. tvOS 
handles this automatically.

1. Install **Tailscale** from the App Store on Apple TV
2. Sign in with your Tailscale account
3. Go to **Settings** in the Tailscale app
4. Enable **Subnet Router**
5. Enable **Allow Local Network Access**
6. The Apple TV will advertise its local subnet automatically

### Admin Console — Approve the Route

After the Apple TV advertises its subnet:

1. Go to **login.tailscale.com/admin/machines**
2. Find your Apple TV device
3. Click on it → find the **Subnets** section
4. Check the checkbox next to `192.168.1.0/24`
5. Click **Save**

> **UI note:** The Tailscale admin console no longer shows 
> "Approved/Advertised" labels explicitly. A checkmark ✅ next 
> to the subnet means it is approved and active.

---

## Setup — Linux Client Accepting Subnet Routes

**This is the most common missed step on Linux.**

Windows, macOS, iOS, and tvOS automatically accept advertised 
subnet routes. Linux does not — it requires explicit opt-in:

```bash
sudo tailscale set --accept-routes
```

**Why Linux requires this:**
It is a deliberate security measure. A rogue subnet router on 
your Tailnet cannot automatically inject routes into your Linux 
device's routing table without your explicit consent.

---

## Correct Diagnostic Toolkit

### The Wrong Tool — `ip route show`

During initial troubleshooting, `ip route show` was used to 
check if the subnet route was present:

```bash
ip route show
```

**This was the wrong tool for this problem.**

`ip route show` displays the Linux kernel routing table — it 
shows routes managed by the OS network stack. Tailscale manages 
subnet routes in its own internal routing layer, which intercepts 
packets before they reach the kernel.

`192.168.1.0/24` will never appear in `ip route show` even when 
Tailscale subnet routing is working perfectly.

**When `ip route show` IS useful:**
- Diagnosing kernel-level routing issues
- Verifying static routes
- Checking default gateway configuration
- Non-Tailscale network troubleshooting

---

### The Right Tools — Tailscale Layer Diagnostics

#### 1. Check Tailscale device status

```bash
tailscale status
```

Shows all Tailnet devices, their IPs, OS, and connection type 
(`direct` or `relay`). Does **not** show subnet routes in the 
simple output.

#### 2. Check active subnet routes (correct command)

```bash
tailscale status --json | python3 -m json.tool | grep -A5 "PrimaryRoutes"
```

**Command breakdown:**
- `tailscale status --json` — outputs full status as raw JSON
- `python3 -m json.tool` — pretty-prints JSON for readability
- `grep -A5 "PrimaryRoutes"` — finds `PrimaryRoutes` and shows 
  5 lines after it (`-A` = after)

**What to look for:**
```json
"PrimaryRoutes": [
    "192.168.1.0/24"
],
```

If `PrimaryRoutes` contains your subnet → Tailscale has an 
active approved route via that device.

**Alternative — cleaner output:**
```bash
tailscale status --json | python3 -c "
import json,sys
data=json.load(sys.stdin)
for peer in data.get('Peer',{}).values():
    routes=peer.get('PrimaryRoutes',[])
    if routes:
        print(peer['HostName'], routes)
"
```

#### 3. Check if Linux client is accepting routes

```bash
tailscale status --self
```

Look for `--accept-routes` in the flags. If absent, run:
```bash
sudo tailscale set --accept-routes
```

#### 4. Check ACL policy

If routing is confirmed working but traffic is still blocked, 
the issue is the ACL policy. Open:
`login.tailscale.com/admin/acls`

Verify there is a rule allowing traffic from your source device 
to the subnet destination.

---

## Correct Troubleshooting Flow+
ping fails to subnet device
↓
tailscale status
→ Is the subnet router device connected?
→ Is it showing idle/active (not offline)?
↓
tailscale status --json | grep PrimaryRoutes
→ Is the subnet showing under PrimaryRoutes?
→ If not → check admin console subnet approval ✅
↓
sudo tailscale set --accept-routes  (Linux only)
→ Did the Linux client accept routes?
↓
ping subnet router Tailscale IP first
→ Can you reach the subnet router device itself?
↓
ping device on home subnet
→ 192.168.1.254 (modem), 192.168.1.59 (ASUS AP)
↓
If still failing → check ACL policy
→ Is traffic from VM to 192.168.1.0/24 permitted?

---

## Understanding `ping -c 4`

```bash
ping -c 4 192.168.1.254
```

- `-c` = **count** — number of packets to send
- `4` = send exactly 4 packets then stop

Without `-c`, Linux ping runs indefinitely until `Ctrl+C`. 
Using `-c 4` is standard for quick connectivity verification.

---

## DERP vs Direct Connection

When checking `tailscale status`, connection type appears next 
to each peer:
iwillwindows  active; direct 108.173.x.x:41641
iwilltvlivingroom  idle; relay: sea

**Direct connection** — Tailscale established a peer-to-peer 
WireGuard tunnel directly between devices. Best performance.

**DERP relay** — Tailscale cannot establish a direct connection 
and routes traffic through a relay server (in this case Seattle). 
Traffic remains end-to-end encrypted — DERP only sees encrypted 
packets, not content.

**Why Apple TV uses DERP in this lab:**
The home network uses double NAT — Telus modem + ASUS router 
(even in AP mode) creates NAT layers that block WireGuard's UDP 
hole-punching. DERP handles this automatically.

**DERP = Detoured Encrypted Routing Protocol** — Tailscale's 
relay network. Not a fallback or failure — a deliberate design 
for networks where direct P2P isn't possible.

**Impact on latency:**
- First ping via DERP: `300ms` (DERP relay + route negotiation)
- Subsequent pings: `74ms` (Tailscale optimised to direct or 
  cached DERP path)

**Fix for direct connection (optional):**
Enable UDP port forwarding on the Telus modem for port `41641` 
pointing to the Apple TV's LAN IP. However this defeats 
Tailscale's core value proposition — zero port forwarding 
required. DERP at `74ms` is acceptable for most use cases.

---

## High Availability — Multiple Subnet Routers

Tailscale supports multiple devices advertising the same subnet 
for redundancy. This is called **HA (High Availability) subnet 
routing**.

**How it works:**
- Two devices both advertise `192.168.1.0/24`
- Tailscale selects one as the `PrimaryRoutes` router
- If the primary goes offline → Tailscale automatically fails 
  over to the secondary
- Clients require no configuration changes

**Planned lab upgrade:**
A second Apple TV (`iwilltvlivingroom2`) will be configured as 
a secondary subnet router advertising the same `192.168.1.0/24` 
subnet — demonstrating HA failover in a real home lab environment.

**Verify HA is working:**
```bash
tailscale status --json | python3 -c "
import json,sys
data=json.load(sys.stdin)
for peer in data.get('Peer',{}).values():
    routes=peer.get('PrimaryRoutes',[])
    advertised=peer.get('AdvertisedRoutes',[])
    if routes or advertised:
        print(peer['HostName'])
        print('  Primary:', routes)
        print('  Advertised:', advertised)
"
```

## HA Failover — Live Test Results

### Test Procedure

To verify HA failover works correctly:

1. Confirm primary subnet router (`iwilltvliving`) is active
2. In admin console → find `iwilltvliving` → **Edit route settings** 
   → uncheck `192.168.1.0/24` → Save
3. Wait 5 seconds
4. From Azure VM, ping a device on the home subnet:
```bash
   ping -c 4 192.168.1.254
```
5. Restore HA — re-check `192.168.1.0/24` on `iwilltvliving`

### Results

| Test | Result |
|---|---|
| Primary subnet router disabled | `iwilltvmaster` (bedroom) automatically became primary |
| Ping to `192.168.1.254` after failover | ✅ Success — 4/4 packets received |
| Connection type | DERP relay Seattle (expected — WiFi + NAT on bedroom Apple TV) |
| Client reconfiguration needed | ❌ None — fully automatic |
| Time to failover | ~5 seconds |

### Key Observations

**The `-` in `tailscale status` means standby — not broken.**
When two devices advertise the same subnet, Tailscale picks one 
as primary. The other shows `-` in `tailscale status` — this is 
correct standby behaviour, not an error.

**Living room Apple TV wins primary consistently** because it is 
connected via ethernet directly to the Telus modem — lower 
latency path than bedroom Apple TV on WiFi.

**Failover is zero-touch for clients** — no commands needed on 
the Azure VM or any other Tailnet device. Tailscale handles 
rerouting automatically.

### Production Framing

In this lab, both Apple TVs are on the same physical subnet 
`192.168.1.0/24` — this demonstrates **HA failover** rather 
than true site-to-site routing.

In production, each subnet router would sit on a different 
physical network:

| Device | Subnet | Represents |
|---|---|---|
| HQ subnet router | `192.168.1.0/24` | Corporate HQ LAN |
| Branch subnet router | `10.10.0.0/24` | Branch office LAN |
| Cloud subnet router | `10.0.0.0/24` | Azure VPC |

Each site advertises its own unique subnet — Tailscale devices 
reach any site transparently through the correct subnet router.

**Official HA reference:**
https://tailscale.com/docs/how-to/set-up-high-availability

---

## Common Failure Points — Quick Reference

| Symptom | Likely Cause | Fix |
|---|---|---|
| Ping fails, subnet router connected | Linux not accepting routes | `sudo tailscale set --accept-routes` |
| Subnet not in PrimaryRoutes | Route not approved in admin console | Admin console → Machines → approve subnet ✅ |
| Ping fails after accept-routes | ACL blocking traffic | Check ACL policy, add permit rule for subnet |
| Subnet router shows offline | Device powered off or Tailscale not running | Check device, restart Tailscale |
| High latency first ping | DERP relay in use | Expected — Tailscale optimises after first packet |
| Direct connection not establishing | Double NAT / UDP blocked | DERP handles automatically — no action needed |
| Route present but traffic dropped | SNAT issue or firewall on target device | Check target device firewall, verify SNAT enabled |

---

## Key Lessons Learned

**1. Use the right diagnostic layer**
`ip route show` is for kernel routing — not Tailscale routing. 
Always start with `tailscale status --json` for Tailscale issues.

**2. Linux requires explicit route acceptance**
Windows, macOS, iOS, and tvOS accept subnet routes automatically. 
Linux always requires `--accept-routes`. This is intentional 
security design, not a bug.

**3. Admin console approval is required**
Advertising a subnet on the device is not enough — the route 
must also be approved in the admin console. The checkmark ✅ 
confirms approval.

**4. DERP is not a failure**
Seeing `relay: sea` in `tailscale status` is not an error. 
DERP provides connectivity where direct P2P isn't possible, 
with full end-to-end encryption maintained.

**5. Identify the network layer before troubleshooting**
> "Is this a Tailscale layer issue or a network layer issue?"

Tailscale layer → `tailscale status`, admin console, ACL policy  
Network layer → `ip route`, `traceroute`, `ping`, firewall rules

---

## Official References

| Topic | URL |
|---|---|
| Subnet routers | https://tailscale.com/docs/features/subnet-routers |
| HA subnet routers | https://tailscale.com/docs/how-to/set-up-high-availability |
| Site-to-site networking | https://tailscale.com/docs/features/site-to-site |
| ACL reference | https://tailscale.com/docs/reference/syntax/policy-file |
| Troubleshooting | https://tailscale.com/docs/reference/troubleshooting |
| DERP explanation | https://tailscale.com/blog/how-tailscale-works |