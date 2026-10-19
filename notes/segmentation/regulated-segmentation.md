# Segmentation for Regulated Environments

## Banking / PCI Context

### PCI Zones in ACI

| PCI Zone                          | ACI Construct                               |
| --------------------------------- | ------------------------------------------- |
| Cardholder Data Environment (CDE) | Dedicated Tenant or VRF                     |
| CDE App Tiers                     | EPGs (Web, App, DB) with explicit contracts |
| PCI DMZ                           | Separate BD + EPG                           |
| Out-of-scope                      | Separate Tenant or VRF                      |
| Shared services                   | EPGs in "common" tenant                     |

**ACI is deny-all by default = PCI Requirement 1 satisfied inherently.**

### What QSA Auditors Want

1. Every contract has documented business justification
2. No "allow all" contracts — exact ports/protocols in filters
3. Segmentation validation testing (pentest)
4. All denied traffic logged
5. Change management (CAB) for every contract change
6. Network diagrams showing tenants/VRFs/BDs/EPGs
7. Contracts reviewed every 6 months (PCI 4.0)

### SWIFT Zone Isolation

- Dedicated Tenant or VRF for SWIFT
- **Zero internet connectivity** (no contract to internet EPGs)
- SWIFT-specific protocols only between EPGs
- Operator PCs in separate EPG with GUI-port-only contracts

### Banking Segmentation Patterns

- **ATM:** Dedicated EPG per group, contracts only to controller EPG
- **Trading floor:** EPG per desk, low-latency = may skip FW insertion
- **Core banking platform:** EPGs per tier (presentation/app/DB/batch)
- **DMZ:** Service graph with FW, intra-EPG isolation enforced

### DORA (In Force Since Jan 2025)

- Mandatory network segmentation
- Complete ICT asset inventory including network components
- Continuous network monitoring (IDS/IPS)
- Incident isolation capability (ability to disconnect zones)
- Third-party access must be segmented and audited
- TLPT (Threat-Led Penetration Testing) — segmentation will be tested

### Key Regulations Stack

| Regulation      | Network Impact                                         |
| --------------- | ------------------------------------------------------ |
| **DORA**        | Mandatory segmentation, monitoring, incident isolation |
| **PCI-DSS 4.0** | CDE isolation, segmentation testing, contract review   |
| **SWIFT CSP**   | Secure zone architecture, zero internet                |
| **National ICT-risk circulars** | ICT risk management, documented architecture |
| **TIBER-EU (national variants)** | Red team testing of segmentation |
| **NIS2**        | Network security requirements                          |

---
## Network Segmentation Deep Dive

### Flat vs Segmented

- Flat = all devices in same broadcast domain, unrestricted lateral movement
- Segmented = isolated zones with controlled paths between them
- **Banking:** compromised printer must never reach core banking system

### Macro vs Micro Segmentation

| Aspect      | Macro                 | Micro                    |
| ----------- | --------------------- | ------------------------ |
| Granularity | Network/zone level    | Per-workload             |
| Mechanisms  | VLANs, VRFs, FW zones | EPGs, contracts, ESGs    |
| Controls    | North-South primarily | East-West primarily      |
| ACI mapping | VRFs, Bridge Domains  | EPGs, Contracts, Filters |

Banks use **both**. Macro defines major zones. Micro enforces within/between app tiers.

### Zero Trust in ACI

ACI is **inherently Zero Trust**:

- Default deny between all EPGs
- Least privilege via specific contracts
- Identity-based via ESGs (tag-based, not VLAN-based)
- Distributed enforcement at every leaf switch
- No central chokepoint

### East-West Traffic

- **70-80% of DC traffic is East-West** — traditional perimeter FW is blind to it
- ACI enforces on every E-W flow at fabric level in hardware (leaf TCAM)
- No performance penalty — this is a key selling point

---

## PCI DSS v4.x Network Requirements

### Requirement 1: Network Security Controls

v4.0 broadened from "firewalls" to **"Network Security Controls"** — includes SDN policy (ACI contracts).

| Req   | What                              | ACI Mapping                       |
| ----- | --------------------------------- | --------------------------------- |
| 1.2.1 | NSC configuration standards       | Contract/filter standards in APIC |
| 1.2.4 | Accurate network diagram          | APIC topology view                |
| 1.2.5 | All permitted services identified | Contracts + Filters               |
| 1.3.1 | Inbound to CDE restricted         | Consumer contracts on CDE EPGs    |
| 1.3.2 | Outbound from CDE restricted      | Provider contracts                |

### CDE Isolation Pattern

```
Tenant: Bank-PCI
  VRF: CDE-VRF          (hard L3 isolation)
  VRF: Corporate-VRF    (separate routing domain)

  Application Profile: Payment-Processing
    EPG: Web-Tier        (payment gateway)
    EPG: App-Tier        (processing logic)
    EPG: DB-Tier         (cardholder data)

  Contracts:
    Web-to-App: TCP 8443 only
    App-to-DB:  TCP 1521 only (Oracle)
    No contract = no communication
```

Separate VRFs = hard boundary. Even misconfigured EPG can't leak across VRFs.

### Segmentation Testing (11.4.5/11.4.6)

- Annual for merchants, **6-monthly for service providers**
- Must test after ANY change to segmentation controls
- Must attempt to cross each boundary
- ACI audit logs prove contract changes for documentation

### Scope Reduction

Without segmentation: **every system in scope**. With proper segmentation: only CDE + connected systems. Dramatically reduces audit cost and risk.

### What QSAs Want

1. Documented contracts with business justification
2. No "allow all" — exact ports/protocols
3. Pentest results proving isolation
4. Logging of denied traffic
5. CAB process for contract changes
6. Network diagrams (tenants/VRFs/BDs/EPGs)
7. 6-monthly contract reviews

---

## Banking Architecture — Complete Example

```
Tenant: Bank-Production

+-- VRF: CDE-VRF                     (PCI: payment processing)
|   +-- EPG: Payment-Gateway-Web
|   +-- EPG: Payment-Processing-App
|   +-- EPG: CHD-Database
|   Contracts: strict, minimal ports

+-- VRF: SWIFT-Secure-VRF            (SWIFT CSCF)
|   +-- EPG: Messaging-Interface
|   +-- EPG: SWIFTNet-Link-HSM
|   No internet L3Out

+-- VRF: CoreBanking-VRF             (Crown jewels)
|   +-- EPG: CBS-Application
|   +-- EPG: CBS-Database
|   +-- EPG: API-Gateway

+-- VRF: ATM-VRF                     (Isolated, intra-EPG isolation ON)
|   +-- EPG: ATM-Endpoints
|   +-- EPG: ATM-Host-Switch

+-- VRF: Trading-VRF                 (Low-latency)
|   +-- EPG: OMS-Servers
|   +-- EPG: Market-Data-Feed
|   +-- EPG: FIX-Gateway

+-- VRF: E-Banking-DMZ-VRF           (Internet-facing)
|   +-- EPG: Web-Servers
|   +-- EPG: WAF-Cluster
|   L3Out to internet (via FW service graph)

+-- VRF: Corporate-VRF               (Out of PCI scope)
|   +-- EPG: Workstations
|   +-- EPG: Email-Servers

+-- VRF: Management-VRF              (Admin access)
    +-- EPG: PAW-Workstations
    +-- EPG: Jump-Servers
    +-- EPG: Monitoring (SIEM, NMS)
    Intra-EPG isolation on PAWs

Inter-VRF:
- E-Banking DMZ --> CoreBanking: via Service Graph (FW inline)
- Management --> All VRFs: narrow contracts (SSH/RDP only)
- Corporate --> NOTHING sensitive (no contracts to CDE/SWIFT/Core)
- ATM --> CoreBanking: ATM host protocol only
```

This satisfies PCI, SWIFT CSP, DORA, ISO 27001, and SOX simultaneously.

### Compliance Framework Overlap

| Control         | PCI DSS    | DORA     | ISO 27001 | NIS2      | SWIFT CSP | SOX  |
| --------------- | ---------- | -------- | --------- | --------- | --------- | ---- |
| Segmentation    | Req 1      | Pillar 1 | A.8.22    | Art 21    | CSCF 1.1  | ITGC |
| Pen testing     | Req 11.4   | Pillar 3 | A.8.8     | Art 21    | CSCF 7.1  | -    |
| Third-party     | Req 12.8   | Pillar 4 | A.5.19    | Art 21(d) | CSCF 2.8A | ITGC |
| Incident        | Req 12.10  | Pillar 2 | A.5.24    | Art 23    | Advisory  | -    |
| Asset inventory | Req 12.5.1 | Art 8    | A.5.9     | Art 21(a) | CSCF 2.1  | ITGC |
| Access control  | Req 7-8    | Art 9    | A.5.15    | Art 21(i) | CSCF 4.1  | ITGC |

---

