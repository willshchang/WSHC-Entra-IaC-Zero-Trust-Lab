# Tailscale ACL, Tags and Access Control

**Document Type:** Admin Technical Reference  
**Author:** Will Chang, Tailscale Customer Success Engineer  
**Audience:** IT Administrator / Tailscale CSE Reference  
**Last Updated:** April 2026  
**Repository:** https://github.com/willshchang/WSHC-Entra-IaC-Zero-Trust-Lab  
**Official Reference:** https://tailscale.com/docs/features/access-control

---

## Overview

This document covers the design, implementation, and verification 
of Tailscale ACL (Access Control List) policy for the WSHC Zero 
Trust lab. It includes tag ownership, role-based grants, SSH 
access rules, and test validation.

**Core design principle:**
> Identity drives access, tags define infrastructure roles. 
> ACL rules mirror the Entra ID RBAC model — one source of 
> truth for roles, two enforcement layers (identity + network).

---

## Why ACLs Matter — Zero Trust Network Layer

Tailscale's default policy allows all devices on a tailnet to 
reach all other devices. This is convenient for personal use 
but violates Zero Trust principles in any team or production 
environment.

**Without ACLs:**
Any device → Any device = full access

**With ACLs (least privilege):**
ITOps identity → Infrastructure only
Infrastructure → Home LAN only
Everything else → Blocked by default

ACLs are the network enforcement layer that complements 
Entra ID's identity enforcement layer:

| Layer | Tool | What it enforces |
|---|---|---|
| Identity | Microsoft Entra ID | Who can authenticate and join the Tailnet |
| Network | Tailscale ACL | What authenticated devices can reach |

---

## ACL vs Entra ID — Two Layers, One Model

The Tailscale ACL policy mirrors the Entra ID RBAC model 
designed in this lab:

| Entra ID Role | Tailscale ACL Identity | Access |
|---|---|---|
| Global Administrator (ITOps) | `will.sh.chang@gmail.com` | Full infrastructure access |
| Infrastructure device | `tag:server` | Subnet access only |
| Network device | `tag:subnet-router` | Routes traffic, no direct access |

In a production multi-user tailnet, Entra ID groups would map 
directly to Tailscale groups:

```json
"groups": {
    "group:itops":    ["will@company.com", "alice@company.com"],
    "group:sre":      ["bob@company.com"],
    "group:finance":  ["carol@company.com"]
}
```

Each group mirrors the Entra ID dynamic group — same roles, 
enforced at both identity and network layers.

---

## Lab Device Topology

| Device | Tailscale Name | Tag | Role |
|---|---|---|---|
| Azure VM | `tinyco-vm` | `tag:server` | Cloud infrastructure |
| Living Room Apple TV | `iwilltvliving` | `tag:subnet-router` | Primary subnet router |
| Bedroom Apple TV | `iwilltvmaster` | `tag:subnet-router` | HA subnet router |
| Windows PC | `iwillwindows` | none | ITOps engineer device |
| iPad Pro | `iwill14pro` | none | ITOps engineer device |
| iPhone | `iwillprom4` | none | ITOps engineer device |

**Why user devices have no tags:**
User devices are identified by the Tailscale user identity 
(`will.sh.chang@gmail.com`) — not by tags. Tags are for 
infrastructure devices that aren't tied to a specific user.

---

## Tag Ownership

Tags in Tailscale require an owner — who can assign the tag 
to devices. Without `tagOwners` defined, tags cannot be used 
in grants.

```json
"tagOwners": {
    "tag:server":        ["will.sh.chang@gmail.com"],
    "tag:subnet-router": ["will.sh.chang@gmail.com"]
}
```

**What this means:**
Only `will.sh.chang@gmail.com` can assign `tag:server` or 
`tag:subnet-router` to devices. Prevents unauthorized devices 
from claiming infrastructure roles.

**Important — tagging transfers ownership:**
When a tag is applied to a device, ownership transfers from 
the user account to the tag. The user loses implicit access — 
access is only via explicit ACL grants.

**Real-time enforcement verified:**
Tag applied to tinyco-vm
↓ immediately
Access revoked.
Connection to tinyco-vm.hair-squeaker.ts.net closed.

This is Zero Trust working correctly — no grace period, 
no legacy access.

---

## ACL Syntax Reference

### `grants`
The main access rules. Replaces the older `acls` field.

```json
{
    "src": ["will.sh.chang@gmail.com"],
    "dst": ["tag:server"],
    "ip":  ["*"]
}
```

- `src` — who initiates the connection (user, group, tag, IP)
- `dst` — what they're trying to reach (tag, IP, subnet)
- `ip` — ports and protocols allowed
  - `["*"]` — all ports, all protocols
  - `["tcp:22"]` — SSH only
  - `["tcp:443", "tcp:80"]` — HTTPS and HTTP only

### `tagOwners`
Defines who can assign a tag to devices.

### `autogroup:self`
Built-in group — devices owned by the same user as the source.

### `autogroup:member`
Built-in group — all members of the tailnet.

### `autogroup:admin`
Built-in group — tailnet administrators only.

### Implicit deny
Tailscale is **default deny** — anything not explicitly 
permitted in `grants` is automatically blocked. No deny rules 
needed. Any unlisted `src`/`dst` combination is blocked.

### `tests`
Validates ACL rules every time the policy is saved. If a test 
fails, the save is rejected — prevents accidentally locking 
yourself out. Requires `hostname:port` format:

```json
"tests": [
    {
        "src":    "will.sh.chang@gmail.com",
        "accept": ["tag:server:22", "tag:subnet-router:80"],
        "deny":   []
    }
]
```

---

## Full ACL Policy

```json
{
  // ============================================================
  // WSHC Entra IaC Zero Trust Lab — Tailscale ACL Policy
  // Author: Will Chang
  // Last Updated: April 2026
  //
  // Design principle: Role-based access control mirroring
  // Entra ID group structure. User identity drives access,
  // tags define infrastructure roles.
  // ============================================================

  "tagOwners": {
    "tag:server":        ["will.sh.chang@gmail.com"],
    "tag:subnet-router": ["will.sh.chang@gmail.com"]
  },

  "grants": [
    // ITOps Engineer — full access to cloud infrastructure
    // Mirrors: TinyCo-ITOps → Global Administrator in Entra
    {
      "src": ["will.sh.chang@gmail.com"],
      "dst": ["tag:server"],
      "ip":  ["*"]
    },
    // ITOps Engineer — full access to subnet router devices
    {
      "src": ["will.sh.chang@gmail.com"],
      "dst": ["tag:subnet-router"],
      "ip":  ["*"]
    },
    // ITOps Engineer — access to home LAN via subnet router
    {
      "src": ["will.sh.chang@gmail.com"],
      "dst": ["192.168.1.0/24"],
      "ip":  ["*"]
    },
    // Cloud server → home LAN access
    // Azure VM can reach home subnet via Apple TV subnet router
    {
      "src": ["tag:server"],
      "dst": ["192.168.1.0/24"],
      "ip":  ["*"]
    }
    // IMPLICIT DENIES (Tailscale default — not written):
    // tag:server → user devices = BLOCKED
    // tag:subnet-router → anywhere = BLOCKED
    // Any unlisted src/dst = BLOCKED
  ],

  "ssh": [
    // ITOps Engineer SSH to cloud server
    // Identity-verified — no passwords, browser auth
    {
      "action": "accept",
      "src":    ["will.sh.chang@gmail.com"],
      "dst":    ["tag:server"],
      "users":  ["tinyco-admin", "iwill", "root"]
    }
  ],

  "tests": [
    {
      "src":    "will.sh.chang@gmail.com",
      "accept": ["tag:server:22", "tag:subnet-router:80", "192.168.1.1:80"],
      "deny":   []
    }
  ]
}
```

---

## Implementation — Step by Step

### Step 1 — Define tags in ACL first

Always write `tagOwners` in the ACL **before** assigning tags 
to devices. If you assign a tag before defining its owner, 
Tailscale may reject the policy.

### Step 2 — Tag infrastructure devices

**Azure VM — via Tailscale CLI:**

Note: `--advertise-tags` is not available on the free personal 
plan CLI. Tag via admin console:

Admin console → **Machines** → `tinyco-vm` → three dots → 
**Edit tags** → select `tag:server` → set owner to 
`will.sh.chang@gmail.com` → Save

**Apple TVs — admin console only (no tvOS CLI):**

Admin console → **Machines** → `iwilltvliving` → three dots → 
**Edit tags** → select `tag:subnet-router` → Save

Repeat for `iwilltvmaster`.

> **Warning:** Tagging a device immediately transfers ownership 
> from your user account to the tag. Access is revoked instantly 
> until ACL grants are saved.

### Step 3 — Replace default allow-all grant

Remove:
```json
{"src": ["*"], "dst": ["*"], "ip": ["*"]}
```

Replace with explicit role-based grants. Save.

### Step 4 — Verify with tests block

The `tests` block validates on every save. A failed test 
prevents saving — your safety net against lockout.

---

## Verification — Live Results

### SSH Access Revoked and Restored
Tag applied — access immediately revoked
Access revoked.
Connection to tinyco-vm.hair-squeaker.ts.net closed.
ACL not yet saved — SSH blocked by policy
$ tailscale ssh tinyco-admin@tinyco-vm
tailscale: tailnet policy does not permit you to SSH to this node
Connection closed by UNKNOWN port 65535
ACL saved with explicit grant — access restored
$ tailscale ssh tinyco-admin@tinyco-vm
Welcome to Ubuntu 24.04.4 LTS (GNU/Linux 6.17.0-1010-azure x86_64)

**What this demonstrates:**
- Zero Trust enforcement is real-time — no grace period
- ACL grants restore access precisely as designed
- Identity-based SSH — no passwords required

---

### Site-to-Site Subnet Routing Under ACL

After ACL policy applied, Azure VM successfully reaches 
home LAN devices via Apple TV subnet router:

```bash
# Telus modem (192.168.1.254) — non-Tailscale device
ping -c 4 192.168.1.254

PING 192.168.1.254 (192.168.1.254) 56(84) bytes of data.
64 bytes from 192.168.1.254: icmp_seq=1 ttl=64 time=315 ms
64 bytes from 192.168.1.254: icmp_seq=2 ttl=64 time=81.5 ms
64 bytes from 192.168.1.254: icmp_seq=3 ttl=64 time=83.4 ms
64 bytes from 192.168.1.254: icmp_seq=4 ttl=64 time=80.6 ms
--- 192.168.1.254 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss
```

```bash
# ASUS router in AP mode (192.168.1.59) — non-Tailscale device
ping -c 4 192.168.1.59

PING 192.168.1.59 (192.168.1.59) 56(84) bytes of data.
64 bytes from 192.168.1.59: icmp_seq=1 ttl=64 time=86.9 ms
64 bytes from 192.168.1.59: icmp_seq=2 ttl=64 time=82.7 ms
64 bytes from 192.168.1.59: icmp_seq=3 ttl=64 time=81.0 ms
64 bytes from 192.168.1.59: icmp_seq=4 ttl=64 time=82.1 ms
--- 192.168.1.59 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss
```

**Latency observation:**
First ping to `192.168.1.254` shows `315ms` — DERP relay 
negotiating the path after ACL was applied. Subsequent pings 
drop to `80-83ms` as the path is optimised. `192.168.1.59` 
pings are consistent at `81-86ms` — path was already cached 
from the previous ping sequence.

---

## Common Errors

### `test(s) failed — missing port in address`
Error: test(s) failed
address tag:server: missing port in address

**Cause:** The `tests` block `accept` values require `hostname:port` 
format — not just hostname.

**Fix:**
```json
"accept": ["tag:server:22", "tag:subnet-router:80"]
```

---

### `tailnet policy does not permit you to SSH`

**Cause:** Tag was applied before ACL grant was saved, or SSH 
rule is missing from the policy.

**Fix:** Verify SSH block exists in policy:
```json
"ssh": [
    {
        "action": "accept",
        "src":    ["will.sh.chang@gmail.com"],
        "dst":    ["tag:server"],
        "users":  ["tinyco-admin", "iwill", "root"]
    }
]
```

---

## Production Enhancements

---

## Production ACL Design — Multi-Site, Role and Function Based

The lab ACL uses a single identity (`will.sh.chang@gmail.com`) 
with full access — appropriate for a personal lab. In 
production, ACL policy becomes significantly more granular, 
reflecting the organisation's site structure, team roles, 
and specific application functions.

The design principle stays the same — identity drives access, 
tags define infrastructure roles. The difference is specificity.

---

### The Three-Layer Design Framework

Every ACL grant answers three questions:
WHO (src)         → WHAT (dst)          → HOW (ip/port)
group:engineering → tag:server          → tcp:22 only
group:finance     → tag:payroll         → tcp:8443 only
group:hq-staff    → 192.168.1.0/24     → all ports
group:itops       → everything          → everything

This maps directly to the Entra ID RBAC model:

| Entra ID Group | Tailscale Group | Access |
|---|---|---|
| `TinyCo-ITOps` | `group:itops` | All sites, all ports |
| `TinyCo-SRE` | `group:sre` | Servers only, SSH + HTTPS |
| `TinyCo-Finance` | `group:finance` | Payroll server, specific port |
| `TinyCo-HQ-Staff` | `group:hq-staff` | HQ subnet only |
| `TinyCo-Branch-Staff` | `group:branch-staff` | Branch subnet only |

Same roles defined once in Entra ID — enforced at both 
identity layer (Entra SSO) and network layer (Tailscale ACL).

---

### Site-Based Access Control

Different office sites have different subnets. Staff should 
only reach the subnet for their site — not all sites.

```json
"groups": {
    "group:itops":        ["will@tinyco.com"],
    "group:hq-staff":     ["alice@tinyco.com", "bob@tinyco.com"],
    "group:branch-staff": ["carol@tinyco.com", "dave@tinyco.com"],
    "group:sre":          ["eve@tinyco.com"]
},

"grants": [
    // ITOps — full access to all sites and infrastructure
    {
        "src": ["group:itops"],
        "dst": ["192.168.1.0/24", "10.10.0.0/24", "tag:server"],
        "ip":  ["*"]
    },
    // HQ staff — HQ subnet only
    // Can reach printers, APs, local devices at HQ
    {
        "src": ["group:hq-staff"],
        "dst": ["192.168.1.0/24"],
        "ip":  ["*"]
    },
    // Branch staff — branch subnet only
    // Cannot reach HQ subnet at all (implicit deny)
    {
        "src": ["group:branch-staff"],
        "dst": ["10.10.0.0/24"],
        "ip":  ["*"]
    },
    // SRE — servers only, SSH and HTTPS
    {
        "src": ["group:sre"],
        "dst": ["tag:server"],
        "ip":  ["tcp:22", "tcp:443"]
    }
]
```

---

### Function-Based Access — Specific Ports

Beyond site-level access, production ACLs restrict by 
application function — only the ports needed for the job.

```json
// Engineers — SSH and HTTPS to servers only
{
    "src": ["group:engineering"],
    "dst": ["tag:server"],
    "ip":  ["tcp:22", "tcp:443"]
},
// Finance — payroll application only, specific port
{
    "src": ["group:finance"],
    "dst": ["tag:payroll-server"],
    "ip":  ["tcp:8443"]
},
// HR — HR database only
{
    "src": ["group:hr"],
    "dst": ["tag:hr-database"],
    "ip":  ["tcp:5432"]
},
// All HQ staff — printing on HQ subnet
{
    "src": ["group:hq-staff"],
    "dst": ["192.168.1.50"],   // HQ printer IP
    "ip":  ["tcp:631"]         // IPP printing protocol
}
```

> **The printer example — Zero Trust in action:**
> Branch staff attempting to print to the HQ printer 
> (`192.168.1.50`) are blocked at the ACL level — before 
> the packet even reaches the subnet router. The connection 
> never touches the HQ network. This is Zero Trust enforced 
> at the packet level, not the session level.

---

### The Implicit Deny — Your Silent Security Layer

Tailscale's default deny means you never need to write 
block rules. Any combination not explicitly listed is 
automatically blocked:
Branch staff → HQ printer    = BLOCKED (not listed)
Finance → Engineering server = BLOCKED (not listed)
HR → Payroll server          = BLOCKED (not listed)
Engineering → HR database    = BLOCKED (not listed)

In a traditional VPN, once connected you reach everything. 
In Tailscale with explicit ACLs, being connected means 
nothing — you only reach what your identity explicitly permits.

---

### Lab vs Production Comparison

| | Lab (current) | Production (TinyCo) |
|---|---|---|
| **Users** | Single identity | Groups per team and site |
| **Sites** | Azure VM + home LAN | HQ + Branch + Cloud VPC |
| **Port control** | `["*"]` all ports | Specific ports per function |
| **Device tags** | `tag:server`, `tag:subnet-router` | `tag:server`, `tag:payroll`, `tag:hr-database`, `tag:printer` |
| **Subnet access** | All subnets to one user | Site-specific per group |
| **Printer access** | Not applicable | HQ staff → HQ printer only |
| **Implicit denies** | Basic | Comprehensive — finance can't reach engineering |

---

### Interview Framing

When a customer asks:
> *"How do I stop my branch office from accessing HQ 
> resources over Tailscale?"*

Your answer:
> *"By default Tailscale allows all tailnet devices to 
> reach each other. We replace that with explicit grants — 
> branch staff get a grant to their own subnet only. 
> HQ subnet is never listed as a destination for branch 
> staff, so it's implicitly denied. No block rules needed — 
> Tailscale's default deny handles it automatically."*

This demonstrates you understand both the product and the 
Zero Trust principle behind it. 🚀

---

### Multi-user groups mirroring Entra ID

```json
"groups": {
    "group:itops":       ["will@company.com"],
    "group:sre":         ["alice@company.com"],
    "group:engineering": ["bob@company.com", "carol@company.com"]
},
"grants": [
    {
        "src": ["group:itops"],
        "dst": ["tag:server", "tag:subnet-router"],
        "ip":  ["*"]
    },
    {
        "src": ["group:sre"],
        "dst": ["tag:server"],
        "ip":  ["tcp:22", "tcp:443"]
    },
    {
        "src": ["group:engineering"],
        "dst": ["tag:server"],
        "ip":  ["tcp:443"]
    }
]
```

### Port-specific grants (least privilege)

Instead of `["*"]` — restrict to specific ports per role:

```json
// SRE — SSH and HTTPS only
"ip": ["tcp:22", "tcp:443"]

// Engineering — HTTPS only
"ip": ["tcp:443"]

// Monitoring — specific port only
"ip": ["tcp:9090"]
```

### Terraform IaC for tags

```hcl
resource "tailscale_device_tags" "vm_tags" {
  device_id = data.tailscale_device.vm.id
  tags      = ["tag:server"]
}

resource "tailscale_acl" "policy" {
  acl = jsonencode({
    tagOwners = {
      "tag:server"        = ["will.sh.chang@gmail.com"]
      "tag:subnet-router" = ["will.sh.chang@gmail.com"]
    }
    grants = [...]
    ssh    = [...]
  })
}
```

This completes the IaC story — Entra ID managed by Terraform, 
Tailscale ACL managed by Terraform, one codebase for the 
entire Zero Trust stack.

---

## Key Lessons Learned

**1. Tag before you grant — but define grants before tagging**
Write the ACL with grants first, then tag devices. Tagging 
before grants exist causes immediate lockout.

**2. Tests block is your safety net**
Always include `tests` — it prevents saving a policy that 
locks you out. Failed tests = save rejected.

**3. `tagOwners` must be defined before tags can be used**
Tags without owners can't be referenced in grants. Always 
define `tagOwners` first.

**4. Implicit deny is your friend**
You don't write deny rules — Tailscale denies everything 
not explicitly permitted. This is the correct Zero Trust 
default.

**5. ACL and Entra ID should mirror each other**
Same roles, same access model, two enforcement layers. 
If a user loses their Entra group, they should lose Tailscale 
access too. Design both systems together.

---

## Official References

| Topic | URL |
|---|---|
| ACL policy syntax | https://tailscale.com/docs/reference/syntax/policy-file |
| Tags | https://tailscale.com/kb/1068/acl-tags |
| Access control | https://tailscale.com/docs/features/access-control |
| ACL examples | https://tailscale.com/docs/reference/examples/acls |
| Grant examples | https://tailscale.com/docs/reference/examples/grants |
| Groups in ACL | https://tailscale.com/kb/1337/acl-syntax#groups |
| Port-based grants | https://tailscale.com/kb/1337/acl-syntax#grants |
| Default deny | https://tailscale.com/blog/access-control-best-practices |