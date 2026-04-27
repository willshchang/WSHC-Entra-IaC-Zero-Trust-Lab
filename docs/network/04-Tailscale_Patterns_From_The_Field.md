# Tailscale — How It Works in the Real World

**Document Type:** Conceptual Reference  
**Author:** Will Chang, Tailscale Customer Success Engineer  
**Audience:** IT Administrator / End User Education  
**Last Updated:** April 2026  
**Repository:** https://github.com/willshchang/WSHC-Entra-IaC-Zero-Trust-Lab  
**Official Reference:** https://tailscale.com/blog/how-tailscale-works

---

## Overview

This document explains how Tailscale works in plain English — 
using real-world analogies and packet journey diagrams. It is 
designed to help both technical administrators and non-technical 
users understand what Tailscale actually does and why it matters.

---

## The Standard Packet Journey

Every time you access anything on the internet, your packets 
follow the same path:
Your device → ISP → destination → ISP → your device

Your ISP (Internet Service Provider) is not a destination — 
it is the infrastructure. Think of it like this:

| Analogy | Reality |
|---|---|
| Canada Post | Your ISP (Telus, Rogers, Comcast) |
| Roads and highways | ISP network cables and routers |
| Your house | Your device |
| Sorting centre | ISP routers inspecting packets |
| Destination address on envelope | Packet destination IP |

Canada Post handles your mail and can see what's on the 
envelope. Your ISP handles your packets and can see where 
they're going — and who sent them.

**This is why ISP throttling works:**
Your ISP sees every packet destination. When they detect 
traffic going to a streaming service like Twitch, they can 
apply a speed limit specifically to that traffic — while 
leaving everything else at full speed.

---

## How a VPN Changes the Journey

A VPN (Virtual Private Network) — whether wgeasy, a 
commercial VPN, or Tailscale — changes what your ISP can 
see on the envelope.

**Without VPN:**
Your device → [Telus sees: "To Twitch.tv"] → Twitch
↑ throttled

**With VPN / exit node:**
Your device → [Telus sees: "To exit node — encrypted"] → Exit node → Twitch
↑ no throttle — can't identify traffic

Your ISP only sees encrypted packets going to your exit 
node. They cannot see the final destination — Twitch, a 
corporate server, or anything else. They cannot throttle 
what they cannot identify.

The exit node then forwards your request to the real 
destination using its own separate ISP connection. Twitch 
responds to the exit node, which sends the response back 
to you — encrypted — through your ISP.

**The full journey with exit node:**
Your device
↓ encrypted WireGuard packet
Your ISP (Telus) — sees encrypted traffic only, cannot throttle
↓
Exit node (Azure VM / Apple TV / VPS)
↓ plain request — exit node's ISP, not yours
Twitch / destination
↓ response to exit node
Exit node
↓ encrypted response
Your ISP (Telus)
↓
Your device — decrypts and displays

---

## wgeasy vs Tailscale Exit Node

Both achieve the same result — your ISP sees encrypted 
WireGuard traffic, not your real destination. The difference 
is in how the exit node is set up and managed.

| | wgeasy | Tailscale exit node |
|---|---|---|
| **What it is** | WireGuard server you self-host | Any Tailnet device set as exit node |
| **Where it runs** | Rented VPS (DigitalOcean, Vultr etc) | Your own device (VM, server, even Apple TV) |
| **Setup** | Manual — install wgeasy on VPS, manage yourself | Automatic — `tailscale set --advertise-exit-node` |
| **WireGuard management** | You manage keys, peers, config | Tailscale manages everything |
| **Cost** | ~$5–10 USD/month for VPS | Free if you already have a device |
| **Identity** | No identity layer | Entra ID SSO — who can use exit node is ACL-controlled |
| **Use case** | Simple personal VPN | Zero Trust network + exit node + subnet routing |

**VPS (Virtual Private Server):** A rented Linux computer 
in a data centre with its own public IP address. You pay a 
provider monthly and have full control over the server. 
wgeasy is software you install on a VPS to turn it into a 
WireGuard VPN server.

![Packet Journey — VPN Comparison](../packet_journey_vpn_comparison.png)

---

## Tailscale vs Traditional VPN — Real World Trade-offs

### The Problem Both Solve

Both traditional VPN and Tailscale solve the same core 
problem: encrypting your traffic so your ISP cannot inspect 
or throttle it. If all you need is to stop Telus from 
touching your packets, a $10/month commercial VPN works.

The question is: what are you giving up?

---

### Traditional VPN — How It Actually Works
Your device → encrypted → VPN company server → destination

A commercial VPN (NordVPN, ExpressVPN, Mullvad etc) routes 
ALL your traffic through their central servers:

- **You trust your ISP less** — Telus can't see your traffic ✅
- **You trust the VPN company more** — they see everything ❌
- **All traffic bottlenecks through their server** — shared 
  with thousands of other users
- **Your speed is limited by their server capacity and 
  your distance to it**
- **One central point of failure** — server goes down, 
  you lose VPN
- **No identity layer** — anyone with the credentials 
  gets full access

**The privacy paradox:**
You stopped trusting Telus — but now you fully trust 
NordVPN. You've moved the problem, not solved it. 
NordVPN could log, sell, or be compelled to share 
your traffic data. You have no way to verify their 
"no logs" policy.

---

### Self-Hosted VPN (wgeasy on VPS) — The DIY Solution
Your device → encrypted → YOUR VPS → destination

wgeasy on a rented VPS gives you control:

- **You own the exit node** — no third party sees your 
  traffic ✅
- **No shared bandwidth** — your VPS is yours alone ✅
- **You manage everything** — keys, updates, security ❌
- **Monthly VPS cost** — ~$5–10 USD/month
- **Still a central server** — single point of failure
- **No identity layer** — anyone with access gets in
- **No access control** — can't limit who uses what

This is exactly what Will ran before Tailscale. It works — 
but it's infrastructure you have to maintain.

---

### Tailscale — Zero Trust Mesh
Your device → encrypted WireGuard → exit node (your device) → destination

Tailscale replaces the central VPN server with a 
peer-to-peer encrypted mesh:

- **You own the exit node** — your Azure VM, your Apple TV ✅
- **No third party sees your traffic** ✅
- **Direct P2P connections** — no shared bandwidth bottleneck ✅
- **Identity layer** — Entra ID SSO controls who gets in ✅
- **ACL policy** — least privilege, not full network access ✅
- **No central server** — mesh survives any single node failure ✅
- **Zero config WireGuard** — Tailscale manages keys automatically ✅
- **Subnet routing** — reach non-Tailscale devices too ✅
- **Free for personal use** — no monthly VPS fee if you 
  already have a device

---

### Head-to-Head Comparison

| | Commercial VPN | wgeasy (self-hosted) | Tailscale |
|---|---|---|---|
| **Who sees your traffic** | VPN company | Only you | Only you |
| **Bandwidth** | Shared with thousands | Dedicated | Direct P2P |
| **Central server** | Yes — theirs | Yes — yours | No — mesh |
| **Identity/auth** | Password/key | Password/key | Entra ID SSO + MFA |
| **Access control** | None | None | ACL — least privilege |
| **Setup complexity** | Easy | Medium | Easy |
| **Maintenance** | None (they handle it) | You manage VPS | Minimal |
| **Cost** | $5–15 USD/month | $5–10 USD/month (VPS) | Free personal plan |
| **Single point of failure** | Yes | Yes | No |
| **Subnet routing** | No | No | Yes |
| **SSH access control** | No | No | Yes — identity-driven |
| **Audit logs** | Depends on provider | None | Tailscale admin console |

---

### The Real Differentiator — Identity + Access Control

Traditional VPNs and self-hosted solutions answer one 
question: "Is this person connected?"

Tailscale answers a fundamentally different question: 
"Who is this person, what are they allowed to reach, 
and can I prove it?"
Traditional VPN:
Connected = access everything on the network
Tailscale:
Connected = access only what your identity permits
enforced at the packet level
logged in the admin console
revocable instantly

This is the difference between a door that's locked or 
unlocked — and a door with a security guard who checks 
your ID, knows exactly which rooms you're allowed in, 
and keeps a record of every door you opened.

**For a business:** This isn't a convenience feature — 
it's the difference between "we have a VPN" and "we have 
a Zero Trust network." When a laptop is stolen, you revoke 
that device's identity. When an employee leaves, you 
remove them from Entra. Access is gone immediately — 
not when someone remembers to change a shared password.

---

### When Traditional VPN Still Makes Sense

Tailscale is not always the answer:

| Scenario | Better choice |
|---|---|
| Just need geo-unblocking for streaming | Commercial VPN |
| Privacy from your own ISP, no corporate use | Commercial VPN |
| Quick personal privacy, no maintenance | Commercial VPN |
| Team/corporate access with identity control | Tailscale |
| Site-to-site between offices | Tailscale |
| Developer accessing cloud resources | Tailscale |
| Zero Trust security model | Tailscale |
| Self-hosting services securely | Tailscale |

---

### Summary — Why Tailscale Won

For Will's use case:

1. **Stopped ISP throttling** ✅ — same as wgeasy
2. **No monthly VPS cost** ✅ — Azure VM already running
3. **Identity-driven access** ✅ — Entra ID SSO
4. **Least privilege ACL** ✅ — only permitted traffic passes
5. **Subnet routing** ✅ — reach home LAN from Azure VM
6. **Zero config WireGuard** ✅ — no manual key management
7. **Audit trail** ✅ — every connection logged

wgeasy solved one problem. Tailscale solved the same 
problem and added an entire Zero Trust security layer 
on top — for free.

> *"Tailscale is what happens when you stop thinking about 
> VPN as a privacy tool and start thinking about it as 
> an identity-aware network."*

---

## Tailscale's WireGuard Mesh

Unlike a traditional VPN where all traffic routes through 
one central server, Tailscale builds a direct 
peer-to-peer (P2P) encrypted mesh between your devices:
Traditional VPN:
Device A ──→ VPN server ──→ Device B
(bottleneck)
Tailscale mesh:
Device A ─────────────────→ Device B
(direct encrypted tunnel)

Every Tailscale connection uses WireGuard — the same 
protocol wgeasy uses — but Tailscale automates key 
exchange, peer discovery, and NAT traversal. You never 
touch a config file.

**When direct P2P isn't possible** (double NAT, strict 
firewalls), Tailscale falls back to DERP relay servers — 
but traffic remains end-to-end encrypted even through 
the relay.

---

## Why This Matters for Customers

### Use case 1 — Remote access without a traditional VPN

Traditional corporate VPN: all traffic routes through a 
central VPN server, creating a bottleneck. Slow, complex 
to manage, single point of failure.

Tailscale: direct encrypted P2P connections between 
devices. No central bottleneck. Access your office 
resources from anywhere as if you're on the local network.

### Use case 2 — Site-to-site connectivity

Two offices, each with their own network. Traditionally 
requires expensive hardware VPN appliances and complex 
configuration.

Tailscale: install on one device per site, advertise the 
subnet, approve in the admin console. Done. Both sites 
can reach each other's devices transparently.

### Use case 3 — Zero Trust access control

Traditional VPN: once connected, you're on the network — 
access everything.

Tailscale ACL: once connected, you can only reach what 
your identity is explicitly permitted to reach. Access 
is enforced at the packet level, not the session level.

### Use case 4 — ISP throttling bypass

As demonstrated above — exit node routes your traffic 
through a different ISP connection. Your ISP sees 
encrypted WireGuard, not the real destination.

---

## Key Concepts Summary

| Term | Plain English |
|---|---|
| **ISP** | Your internet provider — the roads your packets travel on |
| **Packet** | A small chunk of data sent across the internet |
| **WireGuard** | The encrypted tunnel protocol Tailscale uses |
| **Exit node** | A device that forwards all your internet traffic — makes your ISP see only encrypted packets |
| **Subnet router** | A device that lets Tailscale reach non-Tailscale devices on a local network |
| **DERP relay** | Tailscale's fallback relay when direct P2P isn't possible — still end-to-end encrypted |
| **Tailnet** | Your private Tailscale network — only your devices |
| **ACL** | Rules controlling which Tailnet devices can reach which others |
| **VPS** | A rented Linux server in a data centre with its own public IP |
| **Mesh network** | Direct device-to-device connections instead of routing through a central server |

---

## Official References

| Topic | URL |
|---|---|
| How Tailscale works | https://tailscale.com/blog/how-tailscale-works |
| WireGuard overview | https://tailscale.com/wireguard-vpn |
| Exit nodes | https://tailscale.com/docs/features/exit-nodes |
| DERP relay | https://tailscale.com/blog/how-tailscale-works#encrypted-tcp-relays-derp |
| Subnet routers | https://tailscale.com/docs/features/subnet-routers |
| ACL policy | https://tailscale.com/docs/features/access-control |