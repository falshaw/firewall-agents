# Cisco ACI Quick Start (for Firewall Engineers)

**Purpose:** Learning guide + cheat sheet for engineers who know firewalls/networking but are new to ACI. Deeper material: `aci-deep-dive.md`.

---

## What Is ACI? (Plain English)

ACI = Cisco's SDN solution for data centers. Instead of configuring each switch individually (CLI, box-by-box), you define **policies** on a central controller (APIC) and the fabric pushes them to all switches automatically.

**Think of it like a firewall manager (Panorama, FMC, FortiManager) for switches.** APIC does the same for the entire data center network fabric.

| Traditional DC                | ACI DC                      |
| ----------------------------- | --------------------------- |
| Configure each switch via CLI | Configure policies on APIC  |
| VLANs define segmentation     | EPGs define segmentation    |
| ACLs control traffic          | Contracts control traffic   |
| 3-tier (Core/Dist/Access)     | Spine-Leaf (2-tier, no STP) |
| Manual, error-prone           | Policy-driven, automated    |

---

## The ACI Object Model (Hierarchy)

This is the core thing to learn. Everything in ACI is an object in a tree.

```
Tenant                          ← Top container (like a VRF context on a firewall)
├── VRF (Context)               ← Routing domain (same concept as traditional VRF)
│   └── Bridge Domain (BD)      ← L2 domain (replaces VLAN concept)
│       └── Subnet              ← IP subnet within the BD
├── Application Profile (AP)    ← Groups EPGs for one application
│   └── EPG (Endpoint Group)    ← THE KEY CONCEPT - logical group of endpoints
│       ├── Static Bindings     ← Maps EPG to physical ports/VLANs
│       └── Contracts           ← Rules allowing traffic between EPGs
│           └── Subject         ← Groups filters within a contract
│               └── Filter      ← The actual L3/L4 rule (like an ACL entry)
└── L3Out                       ← External connectivity (WAN, Internet, legacy network)
```

### The Key Mental Model

**EPG = Security Zone.** Firewall engineers already think in zones (Inside, Outside, DMZ). In ACI, every application tier is its own zone (EPG).

| Firewall Concept             | ACI Equivalent                                    |
| ---------------------------- | ------------------------------------------------- |
| Security zone                | EPG                                               |
| Security policy/rule         | Contract + Filter                                 |
| Allow rule (src → dst, port) | Consumer EPG → Provider EPG, Contract with Filter |
| Deny all (default)           | Same! Default deny between EPGs                   |
| VLAN                         | Bridge Domain                                     |
| VRF                          | VRF (same concept)                                |
| NAT policy                   | Contract with Service Graph                       |
| Policy-based routing         | PBR (Policy-Based Redirect)                       |

---

## EPGs — The Core Concept

**Endpoint Group (EPG)** = a logical group of endpoints that share the same policy.

```
Traditional: Put web servers in VLAN 100, app servers in VLAN 200
ACI:         Put web servers in EPG "Web", app servers in EPG "App"
```

**Key difference:** In traditional networking, the VLAN defines both L2 and security boundary. In ACI, the EPG defines the security boundary independently of the VLAN/subnet.

### How Endpoints Join an EPG

| Method          | How                        | When               |
| --------------- | -------------------------- | ------------------ |
| Static binding  | Admin maps port+VLAN → EPG | Bare metal servers |
| VMM integration | vCenter tags VM → EPG      | Virtual machines   |
| IP-based        | Endpoint IP → EPG          | Dynamic assignment |
| MAC-based       | Endpoint MAC → EPG         | Legacy devices     |

### Intra-EPG Communication

- **Default:** Endpoints within the same EPG CAN talk to each other (no contract needed)
- **Intra-EPG isolation:** Can be enabled to block intra-EPG traffic (useful for PCI — e.g., POS terminals shouldn't talk to each other)

---

## Contracts — The Rules

**Contract = the policy that allows EPGs to communicate.**

Without a contract between two EPGs, traffic is BLOCKED (default deny).

```
Consumer EPG  ----[Contract]---→  Provider EPG
   (client)                         (server)

Example:
Web-EPG (Consumer) ---[Allow TCP 443]---→ App-EPG (Provider)
App-EPG (Consumer) ---[Allow TCP 1433]--→ DB-EPG (Provider)
```

### Contract Structure

```
Contract: "Web-to-App"
└── Subject: "HTTPS-Traffic"
    └── Filter: "TCP-443"
        ├── Protocol: TCP
        ├── Destination Port: 443
        └── Direction: both (or consumer-to-provider)
```

### Contract Types

| Type              | Scope             | Use Case                          |
| ----------------- | ----------------- | --------------------------------- |
| Standard          | Within one Tenant | Normal app-to-app                 |
| Shared (vzGlobal) | Cross-Tenant      | Shared services (DNS, NTP, AD)    |
| vzAny             | All EPGs in a VRF | Common services everyone needs    |
| Taboo             | Deny-specific     | Block specific traffic explicitly |

### Preferred Groups

EPGs in a "Preferred Group" can talk freely without contracts. Everything NOT in the group needs explicit contracts. Used for: migration scenarios, trusted internal services.

---

## Fabric Architecture

```
        ┌──────┐  ┌──────┐  ┌──────┐
        │Spine1│  │Spine2│  │Spine3│      ← Spine switches (Nexus 9500)
        └──┬───┘  └──┬───┘  └──┬───┘        Only connect to Leafs
           │╲    ╱│╲  │  ╱│╲    │╱           Full mesh: every Leaf to every Spine
        ┌──┴──┐┌──┴──┐┌──┴──┐┌──┴──┐
        │Leaf1││Leaf2││Leaf3││Leaf4│      ← Leaf switches (Nexus 9300)
        └──┬──┘└──┬──┘└──┬──┘└──┬──┘        Connect to servers, firewalls, routers
           │      │      │      │
        [Servers] [FW]  [Servers] [L3Out]

    ┌──────────────────────────────────┐
    │   APIC Cluster (3 controllers)   │  ← Management plane
    └──────────────────────────────────┘     Policy, config, monitoring
```

### Underlay vs Overlay

| Layer         | Protocol    | Purpose                                   |
| ------------- | ----------- | ----------------------------------------- |
| Underlay      | IS-IS       | Routing between Spine/Leaf (infra)        |
| Overlay       | VXLAN       | Encapsulates tenant traffic across fabric |
| Control plane | COOP        | Endpoint info (MAC/IP) from leafs to the spine database |
| Control plane | MP-BGP      | External (L3Out) routes inside the fabric; EVPN between pods/sites |

**Why no STP?** Spine-Leaf is inherently loop-free. Every Leaf connects to every Spine. No blocked ports. No convergence delays. ECMP load balancing across all Spine links.

---

## Firewall Integration

This is where firewall experience directly applies to ACI.

### Service Graph

A Service Graph inserts an L4-L7 device (firewall, load balancer) into the data path between EPGs.

```
Without Service Graph:
  EPG-A  ──────────────────→  EPG-B
         (direct, fabric only)

With Service Graph:
  EPG-A  ──→  [Firewall]  ──→  EPG-B
         (traffic redirected through firewall)
```

### PBR (Policy-Based Redirect)

PBR tells ACI: "For traffic matching this contract, redirect it to this firewall instead of forwarding directly."

| Deployment | Description                                           | When to Use                        |
| ---------- | ----------------------------------------------------- | ---------------------------------- |
| One-Arm    | Firewall on single interface, traffic in/out same leg | Simple insertion; routed (go-to) mode, as L3 PBR requires |
| Two-Arm    | Firewall with inside + outside interfaces             | Routed mode, NAT scenarios         |
| Multi-Node | Chain: Firewall → Load Balancer → Firewall            | Complex service chains             |

### Managed vs Unmanaged Device

| Type      | APIC Controls                                          | Trade-off                     |
| --------- | ------------------------------------------------------ | ----------------------------- |
| Managed   | APIC pushes config to firewall via device package      | Vendor-specific plugin needed |
| Unmanaged | APIC only redirects traffic, you configure FW manually | Simpler, more control         |

**Recommendation:** Unmanaged mode is the common choice — the firewall team keeps full control of firewall configuration (via its own manager) while ACI handles traffic steering.

---

## PCI Segmentation with ACI

Mapping a classic PCI zone design to ACI constructs:

### PCI Zone → ACI Design

```
PCI CDE Tenant
├── VRF: "CDE-VRF" (isolated routing domain)
│   ├── BD: "POS-Network"
│   │   └── EPG: "POS-Terminals"        ← POS devices
│   ├── BD: "Payment-Processing"
│   │   └── EPG: "Payment-Servers"       ← Transaction processing
│   ├── BD: "Card-Database"
│   │   └── EPG: "DB-Servers"            ← CHD storage
│   └── BD: "PCI-Management"
│       └── EPG: "Jump-Hosts"            ← Admin access via jump host only

Contracts:
  POS-Terminals → Payment-Servers: TCP 8443 (payment API)
  Payment-Servers → DB-Servers: TCP 1433 (MSSQL)
  Jump-Hosts → ALL PCI EPGs: TCP 22, TCP 3389 (SSH/RDP)
  Everything else: DENIED (default)

Intra-EPG Isolation:
  POS-Terminals: ENABLED (POS devices can't see each other)
```

### PCI Requirements Mapped to ACI

| PCI Req | What                          | ACI Implementation                          |
| ------- | ----------------------------- | ------------------------------------------- |
| 1.3.1-1.3.2 | Restrict traffic to/from CDE | Contracts = allow-list only             |
| 1.4.1-1.4.2 | NSCs between trusted and untrusted networks | L3Out contracts, no direct path |
| 1.5.1   | Controls on devices connecting to untrusted networks and CDE | Not ACI scope (endpoint) |
| 2.2.2   | Vendor default accounts managed | APIC default admin password changed       |
| 10.2.1  | Audit logs enabled            | Contract logging enabled                    |
| 10.6.1  | Time synchronisation          | NTP configured on APIC and fabric           |
| 11.4.5  | Segmentation penetration test | Validate contracts block unauthorized flows |

**Why it helps:** Classic designs use VRFs and physical firewalls. ACI adds granular microsegmentation inside the CDE, not just macro zones. Contracts are the allow-list, and default-deny can be evidenced to the QSA via the zoning rules on each leaf.

---

## APIC Key Operations

### GUI Navigation (What You'd See Day-1)

| Tab           | What's There                       |
| ------------- | ---------------------------------- |
| Dashboard     | Health scores, faults, topology    |
| Tenants       | Policy configuration               |
| Fabric        | Physical topology, inventory, pods |
| VM Networking | VMM domain integration             |
| Admin         | AAA, firmware, backups             |
| Operations    | Endpoint tracker, capacity         |

### Essential CLI Commands (on Leaf Switch)

```bash
# Show learned endpoints
show endpoint

# Show endpoints in specific EPG
show endpoint epg <epg-name>

# Show zoning rules (the actual programmed policies)
show zoning-rule

# Show VLAN to EPG mapping
show vlan extended

# Check fabric health
acidiag fnvread          # Fabric node vector - all nodes in fabric

# Check APIC cluster status (on APIC)
acidiag avread           # APIC cluster health
```

### APIC REST API (Quick Reference)

```
Base URL: https://<apic>/api/

# Login
POST /api/aaaLogin.json
Body: {"aaaUser":{"attributes":{"name":"admin","pwd":"password"}}}

# Get all tenants
GET /api/class/fvTenant.json

# Get all EPGs in a tenant
GET /api/mo/uni/tn-{tenant}/ap-{app-profile}.json?query-target=children&target-subtree-class=fvAEPg

# Get endpoints
GET /api/class/fvCEp.json

# Every object has a DN (Distinguished Name):
# uni/tn-Production/ap-WebApp/epg-WebServers
```

### APIC API Inspector

**Tip:** APIC GUI has a built-in API Inspector that shows the REST call for every GUI action. If you click "Create EPG" in the GUI, the Inspector shows the exact POST request. Great for learning the API and for automation.

---

## Troubleshooting Checklist

When an app team says "my app can't connect":

```
1. Is the endpoint learned?
   → APIC: Operational > Client Endpoints
   → CLI: show endpoint | grep <ip>
   → If NOT learned: check port, VLAN, physical connection

2. Is the endpoint in the right EPG?
   → Check static binding (port + VLAN → EPG mapping)
   → If wrong EPG: fix the binding

3. Is there a Contract between the EPGs?
   → Tenants > [tenant] > Application Profiles > [app] > EPGs
   → Check Consumer/Provider contracts
   → If NO contract: that is the problem (default deny)

4. Does the Contract Filter match the traffic?
   → Check protocol, port, direction
   → Common mistake: filter says TCP 80 but app uses TCP 8080

5. Is the Contract actually programmed on the Leaf?
   → CLI: show zoning-rule | grep <scope>
   → If not there: APIC might be out of sync with Leaf

6. Check the Health Score
   → Under 90 = investigate faults
   → Faults tab shows what's broken
```

---

## Micro-Segmentation (Advanced)

### uSeg EPGs (Micro-EPGs)

Standard EPGs group endpoints by port/VLAN. uSeg EPGs classify endpoints by attributes:

| Attribute        | Example     | Use Case               |
| ---------------- | ----------- | ---------------------- |
| IP address       | 10.0.1.0/24 | Subnet-based micro-seg |
| MAC address      | aa:bb:cc:\* | Device-type filtering  |
| VM name          | "PCI-\*"    | VMware integration     |
| Custom attribute | OS=Windows  | Tag-based policy       |

### Endpoint Security Groups (ESGs) — Newer

ESGs are the evolution of EPGs for microsegmentation. Key difference: ESGs decouple security policy from network topology. Endpoints can be in different subnets/BDs but share the same security policy via ESG membership.

**Note:** ESGs are newer and not present in every deployment; check the fabric version before designing around them.

---

## Multi-Site vs Multi-Pod

| Feature           | Multi-Pod          | Multi-Site                         |
| ----------------- | ------------------ | ---------------------------------- |
| APIC cluster      | Single (shared)    | Separate per site                  |
| Independence      | Shared fate        | Independent operation              |
| Latency tolerance | <50ms RTT          | Higher (100ms+)                    |
| Management        | One APIC domain    | NDO (Nexus Dashboard Orchestrator) |
| Use case          | Campus, nearby DCs | DR, geo-distributed DCs            |
| Stretched EPGs    | Yes                | Yes (but careful with L2)          |

**Rule of thumb:** Multi-Site for DR — each DC operates independently if the other goes down. Multi-Pod for nearby DCs that should behave as one fabric.

---

## ACI Automation

| Tool                   | Good For                        | Complexity |
| ---------------------- | ------------------------------- | ---------- |
| APIC REST API          | Direct calls, scripting         | Low        |
| Python + acitoolkit    | Pythonic wrapper for APIC       | Medium     |
| Python + Cobra SDK     | Official Cisco SDK (verbose)    | High       |
| Ansible cisco.aci      | Playbooks for repeatable config | Medium     |
| Terraform aci provider | Infrastructure-as-Code          | Medium     |

### Quick Ansible Example

```yaml
- name: Create EPG
  cisco.aci.aci_epg:
    host: "{{ apic_host }}"
    username: "{{ apic_user }}"
    password: "{{ apic_pass }}"
    tenant: Production
    ap: WebApp
    epg: WebServers
    bd: Web-BD
    state: present
```

---

## Common ACI Gotchas

| Gotcha                                    | What Happens                     | Fix                              |
| ----------------------------------------- | -------------------------------- | -------------------------------- |
| Missing Contract                          | Traffic blocked (default deny)   | Add contract between EPGs        |
| Consumer/Provider swapped                 | Wrong direction, traffic blocked | Check who initiates (consumer)   |
| Wrong VLAN in static binding              | Endpoint not learned in EPG      | Match VLAN to what's on the port |
| Bridge Domain not linked to VRF           | No routing for that BD           | Associate BD with correct VRF    |
| Subnet not marked "advertised externally" | External can't reach subnet      | Enable on BD subnet settings     |
| L3Out missing route-map                   | No routes exported/imported      | Configure route-map on L3Out     |
| Firmware mismatch                         | APIC and switches out of sync    | Check compatibility matrix       |

---

_Bridges firewall/PCI knowledge to ACI concepts._
