# ISO 27001 Quick Reference

**Scope:** ISMS implementation, gap assessments, audit preparation (Lead Implementer level)

---

## What is ISO 27001?

- International standard for Information Security Management Systems (ISMS)
- Framework for managing information security risks
- Certifiable standard (external audit by accredited body)
- Current version: ISO 27001:2022 (revised Annex A with 93 controls)

---

## ISMS Structure

### Plan-Do-Check-Act (PDCA)

```
┌─────────────────────────────────────┐
│             PLAN                     │
│  Context, Scope, Risk Assessment,   │
│  Statement of Applicability         │
├─────────────────────────────────────┤
│             DO                       │
│  Implement controls, training,      │
│  awareness, operations              │
├─────────────────────────────────────┤
│             CHECK                    │
│  Internal audit, monitoring,        │
│  management review, metrics         │
├─────────────────────────────────────┤
│             ACT                      │
│  Corrective actions, continual      │
│  improvement, update risk treatment │
└─────────────────────────────────────┘
```

### Mandatory Documents (Clauses 4-10)

| Clause | Document                             |
| ------ | ------------------------------------ |
| 4.3    | Scope of the ISMS                    |
| 5.2    | Information Security Policy          |
| 6.1.2  | Risk Assessment Methodology          |
| 6.1.3  | Risk Treatment Plan                  |
| 6.1.3d | Statement of Applicability (SoA)     |
| 6.2    | Information Security Objectives      |
| 7.2    | Evidence of Competence               |
| 8.1    | Operational Planning & Control       |
| 8.2    | Risk Assessment Results              |
| 8.3    | Risk Treatment Results               |
| 9.1    | Monitoring & Measurement Results     |
| 9.2    | Internal Audit Program & Results     |
| 9.3    | Management Review Results            |
| 10.2   | Nonconformities & Corrective Actions |

---

## Risk Assessment Process

### Steps

1. **Establish context** - Scope, criteria, risk appetite
2. **Identify risks** - Assets, threats, vulnerabilities
3. **Analyze risks** - Likelihood x Impact → Risk level
4. **Evaluate risks** - Compare against criteria, prioritize
5. **Treat risks** - Select controls from Annex A or custom

### Risk Treatment Options

| Option   | Description                 | Example                        |
| -------- | --------------------------- | ------------------------------ |
| Mitigate | Reduce likelihood or impact | Implement firewall, encryption |
| Transfer | Shift to third party        | Cyber insurance, outsourcing   |
| Avoid    | Eliminate the risk source   | Discontinue risky activity     |
| Accept   | Tolerate within appetite    | Low-risk item documented       |

### Risk Matrix (Example)

```
Impact →    Low(1)  Medium(2)  High(3)  Critical(4)
Likelihood
High(4)       4        8        12        16
Medium(3)     3        6         9        12
Low(2)        2        4         6         8
Very Low(1)   1        2         3         4

Risk Appetite: Accept ≤ 4, Mitigate 5-11, Escalate ≥ 12
```

---

## Annex A Controls (ISO 27001:2022)

### 4 Themes (93 Controls)

| Theme                    | Count | Scope                                |
| ------------------------ | ----- | ------------------------------------ |
| **Organizational** (A.5) | 37    | Policies, roles, threat intel, cloud |
| **People** (A.6)         | 8     | Screening, awareness, remote working |
| **Physical** (A.7)       | 14    | Perimeter, offices, equipment, media |
| **Technological** (A.8)  | 34    | Access, crypto, network, development |

### Key Organizational Controls (A.5)

| Control | Title                           | Key Actions                     |
| ------- | ------------------------------- | ------------------------------- |
| A.5.1   | Information Security Policies   | Document, communicate, review   |
| A.5.2   | IS Roles & Responsibilities     | Define and assign               |
| A.5.7   | Threat Intelligence             | Collect and analyze threat info |
| A.5.8   | IS in Project Management        | Security from project start     |
| A.5.9   | Inventory of Information Assets | Maintain asset register         |
| A.5.10  | Acceptable Use of Information   | Define rules, communicate       |
| A.5.15  | Access Control                  | Need-to-know, least privilege   |
| A.5.23  | IS for Cloud Services           | Cloud security policy           |
| A.5.24  | IR Management Planning          | Incident response plan          |
| A.5.29  | IS During Disruption            | Business continuity             |
| A.5.31  | Legal & Regulatory Requirements | Identify and comply             |
| A.5.34  | Privacy & PII Protection        | GDPR, data protection           |
| A.5.36  | Conformance with Policies       | Monitor and enforce             |

### Key Technological Controls (A.8)

| Control | Title                        | Key Actions                      |
| ------- | ---------------------------- | -------------------------------- |
| A.8.1   | User Endpoint Devices        | MDM, encryption, policy          |
| A.8.2   | Privileged Access Rights     | PIM, MFA, monitoring             |
| A.8.5   | Secure Authentication        | MFA, password policy             |
| A.8.7   | Protection Against Malware   | AV/EDR, updates                  |
| A.8.8   | Technical Vulnerability Mgmt | Scan, patch, mitigate            |
| A.8.9   | Configuration Management     | Hardening, baselines             |
| A.8.10  | Information Deletion         | Secure disposal                  |
| A.8.12  | Data Leakage Prevention      | DLP tools and policies           |
| A.8.15  | Logging                      | Centralized, protected, retained |
| A.8.16  | Monitoring Activities        | SIEM, alerting                   |
| A.8.20  | Network Security             | Segmentation, firewall, IDS      |
| A.8.21  | Security of Network Services | SLAs, monitoring                 |
| A.8.23  | Web Filtering                | URL filtering, proxy             |
| A.8.24  | Use of Cryptography          | Encryption policy                |
| A.8.25  | Secure Development Lifecycle | SSDLC                            |
| A.8.28  | Secure Coding                | OWASP, code review               |

---

## Statement of Applicability (SoA)

The SoA is the **most important document** in an ISMS.

| Column                | Content                           |
| --------------------- | --------------------------------- |
| Control #             | A.5.1, A.5.2, etc.                |
| Control Title         | From Annex A                      |
| Applicable            | Yes/No                            |
| Justification         | Why included or excluded          |
| Implementation Status | Implemented / Partially / Not yet |
| Control Description   | How it's implemented              |
| Evidence              | Where to find proof               |

---

## Audit Process

### Internal Audit

1. **Plan**: Audit program covering all ISMS areas over cycle (typically annual)
2. **Prepare**: Audit criteria, scope, schedule, checklist
3. **Execute**: Document review, staff walkthroughs, observation, evidence collection
4. **Report**: Findings (conformity, nonconformity, observation, OFI)
5. **Follow-up**: Corrective actions, verification

### Certification Audit (External)

| Stage           | Purpose                                    | Duration |
| --------------- | ------------------------------------------ | -------- |
| Stage 1         | Document review, readiness assessment      | 1-2 days |
| Stage 2         | Full implementation audit, evidence review | 3-5 days |
| Surveillance    | Annual check (Year 1 & 2)                  | 1-2 days |
| Recertification | Full re-audit (Year 3)                     | 2-3 days |

### Nonconformity Types

| Type            | Definition                                             | Action Required                                         |
| --------------- | ------------------------------------------------------ | ------------------------------------------------------- |
| **Major**       | Failure to implement requirement, systematic breakdown | Corrective action + root cause + evidence (before cert) |
| **Minor**       | Single observed failure, isolated incident             | Corrective action (by surveillance audit)               |
| **Observation** | Not a nonconformity, but risk of one                   | Consider for improvement                                |
| **OFI**         | Opportunity for improvement                            | Voluntary                                               |

---

## Common Gap Assessment Findings

| Area              | Typical Gap                              |
| ----------------- | ---------------------------------------- |
| Asset Management  | No complete asset register               |
| Risk Assessment   | Informal/undocumented                    |
| Access Control    | No regular access reviews                |
| Logging           | Logs exist but no centralized monitoring |
| Incident Response | Plan exists but never tested             |
| Supplier Security | No security requirements in contracts    |
| Awareness         | No regular security training             |
| BCP/DRP           | Not tested                               |
| Change Management | Informal process                         |
| Policy Review     | Policies outdated                        |

---

## ISO 27001 vs Other Standards

| Standard  | Scope                           | Relationship to 27001      |
| --------- | ------------------------------- | -------------------------- |
| ISO 27002 | Control implementation guidance | Supporting standard        |
| ISO 27005 | Risk management                 | Risk methodology           |
| ISO 27017 | Cloud security                  | Cloud-specific controls    |
| ISO 27018 | PII in cloud                    | Privacy extension          |
| ISO 27701 | Privacy (PIMS)                  | GDPR alignment             |
| ISO 22301 | Business Continuity             | Complementary              |
| SOC 2     | Service org controls            | Overlap with many controls |
| PCI-DSS   | Payment card data               | Specific industry standard |
| NIST CSF  | Cybersecurity framework         | Mapping available          |

---

## Implementation Timeline (Typical)

| Phase                     | Activities                            | Duration    |
| ------------------------- | ------------------------------------- | ----------- |
| 1. Gap Assessment         | Current state vs. requirements        | Month 1-2   |
| 2. Risk Assessment        | Asset register, threats, controls     | Month 2-3   |
| 3. Policy Development     | Write/update policies and procedures  | Month 3-5   |
| 4. Control Implementation | Technical and organizational controls | Month 4-8   |
| 5. Training & Awareness   | Staff training, security culture      | Month 6-8   |
| 6. Internal Audit         | Full audit cycle                      | Month 8-9   |
| 7. Management Review      | Board/exec review of ISMS             | Month 9     |
| 8. Certification Audit    | Stage 1 + Stage 2                     | Month 10-12 |

---

**See also:** `../pci-dss/` (integrated PCI+ISO approach), `../togaf/` (EA governance alignment)

_Reference for ISO 27001 consulting and ISMS implementation. Lead Implementer level._
