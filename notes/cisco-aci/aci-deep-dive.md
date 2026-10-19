# Cisco ACI Deep Dive

## Part 1: ACI Architecture — Spine/Leaf, APIC

### What It Is

Cisco ACI is an intent-based, policy-driven SDN for data centres built on Nexus 9000 hardware. Instead of configuring individual switches, you define application policies centrally on the APIC controller, and the fabric renders them across all switches.

### Traditional Mapping

| Traditional 3-Tier                | ACI Spine/Leaf                                                 |
| --------------------------------- | -------------------------------------------------------------- |
| Core switches                     | Spine switches (interconnect only, never connect to endpoints) |
| Aggregation/Distribution          | Does not exist — collapsed into fabric                         |
| Access switches                   | Leaf switches (all endpoints connect here)                     |
| Spanning Tree for loop prevention | No STP inside fabric — ECMP across all spine paths             |
| North-South traffic dominant      | East-West optimized (any leaf to any leaf in 2 hops max)       |
| CLI per device                    | APIC centralized policy controller (GUI, REST API, CLI)        |
| VLAN-based segmentation           | EPG/Contract-based segmentation                                |

### Key Facts

- **Spine switches** never connect to endpoints. Only interconnect leafs. Every leaf connects to every spine (full mesh).
- **Leaf switches** are the access layer. Servers, firewalls, routers, storage — everything connects to leaf ports.
- **APIC** = cluster of 3+ appliances (NOT in data path). Manages policy, automation, health monitoring. If all APICs go down, fabric continues forwarding with last-known policy.
- **Overlay**: iVXLAN (Cisco proprietary VXLAN) internally. Endpoints see normal VLANs.
- **Control plane**: IS-IS for underlay, COOP on spines for endpoint mapping, MP-BGP for external routes.

### Key Commands

```
acidiag fnvread                    # Fabric node status (from APIC)
moquery -c fabricNode              # All fabric nodes
show isis adjacency                # IS-IS on switch
acidiag avread                     # APIC cluster health
show lldp neighbors                # Topology discovery
```

### Common Misconfigs

- Partial mesh (not every leaf to every spine)
- TEP pool too small
- Running STP inside fabric
- APIC cluster with even number of nodes (need odd for quorum)
- Mismatched NX-OS/APIC firmware versions

---

## Part 2: Tenants, VRFs, Bridge Domains

### Tenants

Top-level container. Everything lives inside a tenant. Like a separate admin domain.

**Default tenants (cannot delete):**

- **common** — shared services (DNS, DHCP, L3Outs). Reusable across tenants.
- **infra** — internal fabric operations. No application workloads here.
- **mgmt** — in-band and out-of-band management.

### VRFs

Layer 3 routing domain within a tenant. Own routing table. Overlapping IPs across VRFs supported. Same as traditional VRF.

**Policy enforcement modes (critical):**

| Mode                   | Behavior                                                 |
| ---------------------- | -------------------------------------------------------- |
| **Enforced** (default) | All inter-EPG traffic needs Contracts. Default deny-all. |
| **Unenforced**         | All EPGs communicate freely. Contracts ignored.          |
| **Preferred Group**    | Members communicate freely; non-members need contracts.  |

### Bridge Domains (BD)

Layer 2 forwarding domain. Associated with exactly one VRF. Contains subnets.

| Traditional VLAN             | ACI Bridge Domain                           |
| ---------------------------- | ------------------------------------------- |
| Single broadcast domain      | Single broadcast domain                     |
| One subnet per VLAN          | Multiple subnets per BD                     |
| SVI for routing              | Anycast Gateway (same IP/MAC on every leaf) |
| Flooding for unknown unicast | Hardware Proxy (spine lookup, no flooding)  |
| ARP broadcast storms         | ARP optimization (fabric proxy-responds)    |

**Anycast Gateway MAC:** `00:22:BD:F8:19:FF` (shared on all leafs)

### Key Commands

```
moquery -c fvTenant              # Show tenants
moquery -c fvCtx                 # Show VRFs
moquery -c fvBD                  # Show bridge domains
moquery -c fvSubnet              # Show subnets
show endpoint vrf <tenant>:<vrf> # Endpoints in a VRF
show vlan extended               # BD on a leaf
```

### Common Misconfigs

- Not enabling Unicast Routing on L2 BDs
- Wrong unknown unicast mode (Flood vs Hardware Proxy)
- BD not associated with VRF
- Subnet scope missing "Advertised Externally" for L3Out reachability

---

## Part 3: EPGs (Endpoint Groups)

### What It Is

Fundamental unit of policy. Groups endpoints sharing common policy. **Within EPG = allowed by default. Between EPGs = denied by default** (needs Contract).

### Traditional Mapping

| Traditional                | ACI EPG                            |
| -------------------------- | ---------------------------------- |
| VLAN + ACL                 | EPG + Contract                     |
| Security zone (firewall)   | EPG (more granular — per app tier) |
| Port-based VLAN assignment | Static port binding or VMM dynamic |

**Key difference:** Traditional = security tied to topology (VLAN/subnet). ACI = security decoupled. Two servers on same subnet can be in different EPGs.

### Classification Methods

1. **Static binding** — leaf port + VLAN to EPG (most common for bare metal)
2. **VMM integration** — dynamic from vCenter/HyperV
3. **Microsegmentation (uSeg)** — classify by IP, MAC, VM attributes
4. **ESG (Endpoint Security Groups)** — ACI 5.0+, decouples security from forwarding entirely

### Key Commands

```
moquery -c fvAEPg               # All EPGs
moquery -c fvCEp                # Endpoints in EPG
show endpoint                   # All learned endpoints
show endpoint ip <ip>           # Endpoint detail
moquery -c fvRsBd               # EPG-to-BD mapping
```

### Common Misconfigs

- No static path binding (EPG exists but no traffic flows)
- Wrong encapsulation VLAN
- EPG not linked to BD
- VLAN pool mismatch
- **Broken access policy chain** (see Part 5)

---

## Part 4: Contracts and Filters

### What They Are

ACI's firewall rules between EPGs. **Default deny** — no contract = no traffic.

### Traditional Mapping

| Traditional         | ACI                   |
| ------------------- | --------------------- |
| ACL / firewall rule | Contract              |
| ACL entries         | Filters               |
| Rule grouping       | Subject               |
| Source zone         | Consumer EPG          |
| Destination zone    | Provider EPG          |
| Implicit deny       | No contract = blocked |

### Hierarchy

```
Contract
  +-- Subject (groups filters, defines direction)
        +-- Filter (match criteria)
              +-- Filter Entry (protocol, ports, etc.)
```

### Provider-Consumer Model

- **Consumer = SOURCE** (initiates connection)
- **Provider = DESTINATION** (offers service)
- E.g., Web server EPG _provides_ HTTP. Client EPG _consumes_ it.

### Contract Types

| Type                | Purpose                                                         |
| ------------------- | --------------------------------------------------------------- |
| **Standard**        | Whitelist. Provider + Consumer. Most common.                    |
| **Taboo**           | Blacklist. Higher priority. Discouraged in production.          |
| **vzAny**           | Apply to ALL EPGs in VRF. Saves TCAM. Good for shared services. |
| **Preferred Group** | Free communication among members. No contracts needed.          |

### Scope

| Scope               | Coverage                                |
| ------------------- | --------------------------------------- |
| Application Profile | Within single App Profile               |
| VRF (default)       | Across App Profiles in VRF              |
| Tenant              | Inter-VRF within tenant (route leaking) |
| Global              | Inter-tenant                            |

### Key Commands

```
moquery -c vzBrCP              # All contracts
moquery -c vzSubj              # Subjects
moquery -c vzFilter            # Filters
moquery -c fvRsProv            # Provider relationships
moquery -c fvRsCons            # Consumer relationships
show zoning-rule               # Rendered contracts on leaf
```

### Common Misconfigs

- Swapped provider/consumer (number one mistake)
- Missing return traffic (not using "Apply Both Directions")
- Contract scope too narrow
- TCAM exhaustion (too many granular contracts)
- Using Taboo contracts in production

---

## Part 5: Access Policies (The Chain)

### The Access Policy Chain — MOST IMPORTANT OPERATIONAL CONCEPT

```
VLAN Pool -> Domain -> AAEP -> Interface Policy Group -> Interface Profile -> Switch Profile
```

**If ANY link is broken, nothing reaches the port.** This is the #1 operational issue in ACI.

### Traditional Mapping

| Traditional                         | ACI                                      |
| ----------------------------------- | ---------------------------------------- |
| `switchport trunk allowed vlan 100` | VLAN Pool (100) -> Domain -> AAEP -> IPG |
| `speed 10000`                       | Link-Level Policy in IPG                 |
| `cdp enable`                        | CDP Policy in IPG                        |
| `channel-group 1 mode active`       | Port Channel Policy (LACP) in IPG        |
| `vpc domain`                        | vPC Explicit Protection Group            |

### Interface Policy Group Types

| Type                       | Use                                   |
| -------------------------- | ------------------------------------- |
| Access (single port)       | Bare metal, single-homed              |
| PC (Port Channel)          | LACP on single leaf                   |
| vPC (Virtual Port Channel) | LACP across two leafs (active-active) |

### APIC GUI Navigation

```
Fabric > Access Policies > Pools > VLAN                    (VLAN Pools)
Fabric > Access Policies > Physical and External Domains   (Domains)
Fabric > Access Policies > Global Policies > AAEP          (AAEP)
Fabric > Access Policies > Interfaces > Policy Groups      (IPG)
Fabric > Access Policies > Interfaces > Profiles           (Interface Profiles)
Fabric > Access Policies > Switches > Profiles             (Switch Profiles)
```

### Common Misconfigs

- **Broken policy chain** — #1 issue. Missing any link = no config on port.
- VLAN pool allocation mode wrong (Static for bare metal, Dynamic for VMM)
- Forgetting vPC Explicit Protection Group
- Duplicate interface profile associations

---

## ACI Object Model — Quick Reference

```
Fabric
  +-- Tenant
        +-- VRF
        +-- Bridge Domain (-> VRF)
        |     +-- Subnet
        +-- Application Profile
        |     +-- EPG (-> BD)
        |           +-- Static Path Binding (leaf/port/vlan)
        |           +-- Contract (provides/consumes)
        +-- Contract
        |     +-- Subject -> Filter -> Filter Entry
        +-- L3Out (-> VRF)
        +-- ESG (newer, -> VRF directly)

Access Policies (Fabric-level)
  +-- VLAN Pool -> Domain -> AAEP -> IPG -> Interface Profile -> Switch Profile
```

---

## Top 10 Key Points

1. ACI is **default deny** between EPGs
2. Access policy chain must be complete or nothing works
3. Bridge Domain is NOT a VLAN — multiple subnets, anycast gateway, hardware proxy
4. **Consumer = source, Provider = destination**
5. APIC is out-of-band — fabric survives APIC failure
6. Spine-leaf = always 2 hops max
7. vzAny saves TCAM for shared services
8. ESGs are the evolution of EPGs (ACI 5.0+)
9. COOP on spines = endpoint location database
10. `show zoning-rule` = verify rendered contracts on leaf

---

## Part 6: Firewall Integration in ACI

### Service Graphs

ACI's mechanism for inserting L4-L7 services (firewalls, LBs) into traffic path. Instead of manually configuring VLANs/routes/PBR, you define a logical graph in APIC.

**Three Insertion Modes:**

| Mode           | Description                                   | When to Use                           |
| -------------- | --------------------------------------------- | ------------------------------------- |
| **Go-Through** | Transparent/bridged firewall                  | Legacy migrations                     |
| **Go-To**      | Routed firewall, VRF sandwich (multiple VRFs) | Classic perimeter, creates complexity |
| **PBR**        | Routed firewall, single VRF, ACI redirects    | **Modern recommended approach**       |

**Managed vs Unmanaged:**

- **Unmanaged is the norm for firewalls.** You configure the firewall yourself (Panorama/FMC). APIC only handles fabric plumbing.
- Managed requires vendor device package. Rare for firewalls.

### PBR (Policy-Based Redirect)

Single-VRF alternative to VRF sandwich. ACI redirects traffic matching a contract to a firewall **without making the firewall the default gateway**.

**Traditional mapping:** Like `route-map` with `set ip next-hop` but applied at contract level, fabric-wide.

**Critical config:**

1. Create service Bridge Domains — **disable endpoint dataplane learning** (most common misconfiguration)
2. Create PBR Policy (IP + MAC of firewall interface)
3. Register firewall as unmanaged device
4. Create service graph template, attach to contract
5. Enable symmetric PBR for active-active HA (resilient hashing)

**Key gotchas:**

- Forgetting to disable EP learning on service BDs = traffic bypasses firewall intermittently
- Use HA virtual MAC (not physical MAC) in PBR policy
- Health probe failure = traffic bypass (not blackhole) — security risk if unexpected

### L3Out (External Connectivity)

How ACI connects to anything outside the fabric (WAN, internet, external firewalls).

| ACI Concept  | Traditional                                      |
| ------------ | ------------------------------------------------ |
| L3Out        | Uplink + routing protocol on border switch       |
| Border Leaf  | The switch peering with external router/FW       |
| External EPG | Prefix-list defining external subnets for policy |

**Key fact:** BD subnets must have "Advertised Externally" flag set or external devices have no route back. #1 L3Out troubleshooting issue.

### Palo Alto Integration with ACI

**Data Plane:** Unmanaged device + PBR service graph.
**Control Plane:** Panorama ACI Plugin monitors APIC, populates Dynamic Address Groups (DAGs) from EPG membership.

**How DAGs work:**

1. Panorama plugin connects to APIC REST API
2. Monitors endpoint-to-EPG mappings
3. Tags IPs with EPG names on firewalls
4. Firewall policies use DAGs (match tag = EPG name)
5. New VM in EPG = automatic policy update

**Verify:** `show object registered-ip all` on firewall

### East-West vs North-South

|                | North-South                               | East-West                                        |
| -------------- | ----------------------------------------- | ------------------------------------------------ |
| What           | Traffic entering/leaving fabric           | Traffic between internal EPGs                    |
| Without FW     | L3Out + External EPG + contracts          | ACI contracts (stateless L3/L4 ACLs)             |
| With FW        | FW outside fabric (L3Out) or inside (PBR) | Service graph + PBR between EPGs                 |
| When to add FW | Always for perimeter                      | When you need stateful, App-ID, IPS, SSL decrypt |

**vzAny + PBR = ALL inter-EPG traffic through firewall.** Be careful — can overwhelm the firewall. Be selective.

---
