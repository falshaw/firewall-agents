# TOGAF 9 Quick Reference

**Scope:** TOGAF 9 concepts applied to security architecture work. (TOGAF 10 keeps the ADM; terminology below still applies.)

---

## What is TOGAF?

- **The Open Group Architecture Framework**
- Standard for Enterprise Architecture (EA) development
- Provides the Architecture Development Method (ADM) - iterative cycle
- Not prescriptive (framework, not methodology) - adapt to organization

---

## ADM Phases

```
         ┌──────────────┐
         │  Preliminary  │
         │   Phase       │
         └──────┬───────┘
                │
         ┌──────┴───────┐
         │ A: Architecture│
         │    Vision      │
         └──────┬───────┘
     ┌──────────┼──────────┐
     │          │          │
┌────┴────┐┌───┴───┐┌────┴────┐
│B: Business││C: IS  ││D: Tech  │
│Architecture││Arch   ││Arch     │
└────┬────┘└───┬───┘└────┬────┘
     │          │          │
     └──────────┼──────────┘
         ┌──────┴───────┐
         │ E: Opportunities│
         │  & Solutions    │
         └──────┬───────┘
         ┌──────┴───────┐
         │ F: Migration  │
         │   Planning    │
         └──────┬───────┘
         ┌──────┴───────┐
         │ G: Implementation│
         │   Governance     │
         └──────┬───────┘
         ┌──────┴───────┐
         │ H: Architecture│
         │ Change Mgmt    │
         └───────────────┘

     Requirements Management
     (continuous, center of cycle)
```

### Phase Details

| Phase           | Name                             | Key Activities                                           | Key Deliverables                          |
| --------------- | -------------------------------- | -------------------------------------------------------- | ----------------------------------------- |
| **Preliminary** | Framework & Principles           | Establish EA capability, define principles, tailor TOGAF | Architecture Principles, Org Model for EA |
| **A**           | Architecture Vision              | Stakeholder analysis, define scope, create vision        | Architecture Vision, Statement of Work    |
| **B**           | Business Architecture            | Model business processes, capabilities, value streams    | Business Architecture                     |
| **C**           | Information Systems Architecture | Data architecture + Application architecture             | Data & Application Architecture           |
| **D**           | Technology Architecture          | Platforms, networks, infrastructure                      | Technology Architecture                   |
| **E**           | Opportunities & Solutions        | Gap analysis, solution building blocks, roadmap          | Implementation Strategy                   |
| **F**           | Migration Planning               | Prioritize projects, create migration plan               | Implementation & Migration Plan           |
| **G**           | Implementation Governance        | Architecture compliance review                           | Compliance assessments                    |
| **H**           | Architecture Change Management   | Manage changes to architecture                           | Change requests, updated architecture     |
| **RM**          | Requirements Management          | Continuous requirements throughout all phases            | Requirements repository                   |

---

## Core Concepts

### Architecture Domains (BDAT)

| Domain          | Covers                                            |
| --------------- | ------------------------------------------------- |
| **Business**    | Strategy, organization, processes, capabilities   |
| **Data**        | Data models, data management, data flows          |
| **Application** | Application landscape, interactions, capabilities |
| **Technology**  | IT infrastructure, platforms, networks, hosting   |

### Building Blocks

| Type                                  | Description             | Example                            |
| ------------------------------------- | ----------------------- | ---------------------------------- |
| **ABB** (Architecture Building Block) | Conceptual component    | "Identity Management System"       |
| **SBB** (Solution Building Block)     | Specific implementation | "Azure AD with Conditional Access" |

ABB → defines what → SBB → defines how

### Enterprise Continuum

```
Foundation    Common Systems    Industry    Organization
Architecture → Architecture  → Architecture → Architecture
(most generic)                              (most specific)

Foundation → Common → Industry → Org-specific
```

### Architecture Repository

| Component                  | Purpose                         |
| -------------------------- | ------------------------------- |
| Architecture Metamodel     | Defines structure               |
| Architecture Landscape     | Baseline + Target architectures |
| Reference Library          | Guidelines, templates, patterns |
| Standards Information Base | Standards in use                |
| Governance Log             | Decision records                |

---

## Key Deliverables

### Architecture Vision (Phase A)

- Problem statement
- Stakeholder map and concerns
- Business scenario(s)
- High-level architecture (baseline → target)
- Value proposition and ROI
- Scope and constraints

### Gap Analysis (Phase E)

```
         Target Architecture
         ┌────┬────┬────┬────┐
         │ T1 │ T2 │ T3 │ T4 │
    ┌────┼────┼────┼────┼────┤
 B1 │ ✓  │    │    │    │ → Retained (B1=T1)
    ├────┼────┼────┼────┼────┤
 B2 │    │ ~  │    │    │ → Modified (B2→T2)
    ├────┼────┼────┼────┼────┤
 B3 │    │    │    │    │ → Eliminated
    ├────┼────┼────┼────┼────┤
    │    │    │ NEW│ NEW│ → New (T3, T4)
    └────┴────┴────┴────┴────┘
```

### Architecture Principles (Example)

| Principle                          | Statement                                         | Implication                      |
| ---------------------------------- | ------------------------------------------------- | -------------------------------- |
| Security by Design                 | Security built into every solution from inception | All designs include threat model |
| Technology Independence            | Architecture not dependent on specific vendor     | Use open standards, APIs         |
| Data is an Asset                   | Data managed with same rigor as financial assets  | Data governance, classification  |
| Reuse Before Buy, Buy Before Build | Maximize existing investments                     | Solution assessment checklist    |

---

## Governance

### Architecture Governance Framework

```
Policy → Process → Organization → Tools
```

### Architecture Board Responsibilities

- Review and approve architecture designs
- Ensure compliance with principles and standards
- Manage architecture change requests
- Resolve architecture conflicts
- Communicate architecture decisions

### Compliance Assessment

| Level              | Description                          |
| ------------------ | ------------------------------------ |
| **Conformant**     | Fully compliant with architecture    |
| **Compliant**      | Meets intent, minor deviations       |
| **Non-Conformant** | Significant deviations, needs review |
| **Dispensation**   | Temporary exception granted          |

---

## TOGAF for Security Consulting

### Mapping Security to ADM Phases

| ADM Phase          | Security Activity                                                |
| ------------------ | ---------------------------------------------------------------- |
| A: Vision          | Security principles, risk tolerance, compliance requirements     |
| B: Business        | Security-relevant business processes, data classification policy |
| C: IS Architecture | Security architecture (IAM, encryption, DLP), data flow security |
| D: Technology      | Firewalls, IDS/IPS, SIEM, network segmentation, cloud security   |
| E: Solutions       | Security product selection, vendor assessment                    |
| F: Migration       | Security implementation sequencing, risk-based prioritization    |
| G: Governance      | Security compliance reviews, penetration testing                 |
| H: Change Mgmt     | Security impact assessment for changes, threat landscape updates |

### Security Architecture Building Blocks

| ABB (Conceptual)     | SBB (Implementation)               |
| -------------------- | ---------------------------------- |
| Perimeter Security   | Enterprise NGFW cluster, Azure Firewall |
| Identity Management  | Azure AD, Okta, AD on-prem         |
| Data Protection      | AES-256, TDE, Key Vault            |
| Threat Detection     | Microsoft Sentinel, Splunk         |
| Endpoint Protection  | CrowdStrike, Defender for Endpoint |
| Network Segmentation | VRF, NSG, Microsegmentation        |

---

## TOGAF vs Other Frameworks

| Framework | Scope                    | Relationship                 |
| --------- | ------------------------ | ---------------------------- |
| TOGAF     | Enterprise Architecture  | Overarching EA method        |
| Zachman   | Classification framework | Complementary taxonomy       |
| SABSA     | Security Architecture    | Security-focused EA (layers) |
| NIST CSF  | Cybersecurity            | Maps to Phase D controls     |
| ITIL      | IT Service Management    | Operational delivery         |
| COBIT     | IT Governance            | Governance framework         |

---

## Stakeholder Management

### Stakeholder Map

| Stakeholder   | Concerns                        | Architecture View        |
| ------------- | ------------------------------- | ------------------------ |
| CEO/Board     | Risk, compliance, cost          | Strategic overview       |
| CTO/CIO       | Technology roadmap, integration | Technology architecture  |
| CISO          | Security, compliance, threat    | Security architecture    |
| Business Unit | Capability, performance         | Business process view    |
| Operations    | Availability, support           | Infrastructure view      |
| Developers    | APIs, standards, tools          | Application architecture |
| Auditors      | Compliance evidence             | Controls view            |

---

## Key Terms (Quick Recall)

- **Stakeholder** = anyone who has a concern
- **View** = representation of architecture for a stakeholder
- **Viewpoint** = template for creating views
- **Concern** = interest in the system
- **Architecture Landscape** = the architectural assets (Baseline + Target)
- **Transition Architecture** = intermediate state between Baseline and Target
- **Architecture Contract** = agreement between development partners and sponsors

---

**See also:** `../iso27001/` (ISMS governance), `../azure-security/` (Phase D cloud implementation)

_Reference for Enterprise Architecture in security consulting. Maps TOGAF to real-world security work._
