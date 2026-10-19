# PCI-DSS Quick Reference

**Version:** PCI DSS v4.0.1 (v3.2.1 retired March 2024; v4.0 future-dated requirements mandatory from 31 March 2025). Always confirm against the current PCI SSC documents.

---

## The 12 Requirements

### Build and Maintain a Secure Network

| #   | Requirement                                          | Key Controls                                                     |
| --- | ---------------------------------------------------- | ---------------------------------------------------------------- |
| 1   | Install and maintain network security controls       | Firewalls, NSCs, segmentation between CDE and other networks     |
| 2   | Apply secure configurations to all system components | No vendor defaults, disable unnecessary services, harden configs |

### Protect Account Data

| #   | Requirement                                                          | Key Controls                                                                    |
| --- | -------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| 3   | Protect stored account data                                          | Encryption at rest (AES-256), tokenization, truncation, data retention policies |
| 4   | Protect cardholder data with strong cryptography during transmission | TLS 1.2+, no SSL/early TLS, certificate management                              |

### Maintain a Vulnerability Management Program

| #   | Requirement                                      | Key Controls                                              |
| --- | ------------------------------------------------ | --------------------------------------------------------- |
| 5   | Protect all systems against malware              | AV/EDR on all systems, regular updates, periodic scans    |
| 6   | Develop and maintain secure systems and software | Patch management, secure SDLC, WAF for public-facing apps |

### Implement Strong Access Control Measures

| #   | Requirement                                                 | Key Controls                                                        |
| --- | ----------------------------------------------------------- | ------------------------------------------------------------------- |
| 7   | Restrict access to cardholder data by business need to know | RBAC, least privilege, access reviews                               |
| 8   | Identify users and authenticate access                      | Unique IDs, MFA for CDE access, password policy (12+ chars in v4.0) |
| 9   | Restrict physical access to cardholder data                 | Badges, cameras, visitor logs, media destruction                    |

### Regularly Monitor and Test Networks

| #   | Requirement                                                         | Key Controls                                                              |
| --- | ------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| 10  | Log and monitor all access to network resources and cardholder data | Centralized logging, SIEM, daily log reviews, NTP sync                    |
| 11  | Test security of systems and networks regularly                     | Quarterly ASV scans, annual pen tests, IDS/IPS, file integrity monitoring |

### Maintain an Information Security Policy

| #   | Requirement                                                            | Key Controls                                                         |
| --- | ---------------------------------------------------------------------- | -------------------------------------------------------------------- |
| 12  | Support information security with organizational policies and programs | Security policy, security awareness training, incident response plan |

---

## Compliance Levels

Levels are defined by each card brand; the table below follows the common Visa/Mastercard merchant model.

| Level | Criteria                            | Validation                               |
| ----- | ----------------------------------- | ---------------------------------------- |
| **1** | >6M transactions/year OR any breach | Annual ROC by QSA + quarterly ASV        |
| **2** | 1-6M transactions/year              | Annual SAQ + quarterly ASV               |
| **3** | 20K-1M e-commerce transactions/year | Annual SAQ + quarterly ASV               |
| **4** | <20K e-commerce OR <1M other        | Annual SAQ + quarterly ASV (recommended) |

---

## SAQ Types

| SAQ      | Applies To                                        | Scope                             |
| -------- | ------------------------------------------------- | --------------------------------- |
| **A**    | Card-not-present, fully outsourced                | Minimal - redirect/iframe only    |
| **A-EP** | E-commerce with website impacting payment         | Website security controls         |
| **B**    | Imprint machines or standalone dial-out terminals | Very limited                      |
| **B-IP** | Standalone IP-connected PTS terminals             | Terminal + network controls       |
| **C**    | Payment app systems connected to internet         | App + network controls            |
| **C-VT** | Virtual terminal, manual entry                    | Virtual terminal only             |
| **D**    | All others / full CDE control                     | Full 12 requirements              |
| **P2PE** | Hardware-encrypted PTS terminals                  | Minimal - P2PE solution validated |

---

## Network Segmentation

**Purpose:** Reduce PCI scope by isolating CDE from rest of network

### CDE Boundaries

```
┌─────────────────────────────────────┐
│           Corporate Network          │
│  ┌──────────────────────────┐       │
│  │    CDE (Cardholder Data  │       │
│  │       Environment)       │       │
│  │  ┌────────┐ ┌────────┐  │       │
│  │  │Payment │ │Card DB │  │       │
│  │  │Terminal│ │Server  │  │       │
│  │  └────────┘ └────────┘  │       │
│  └──────────────────────────┘       │
│         Firewall/NSC                │
│  ┌──────────┐ ┌──────────┐         │
│  │ Out of   │ │Connected │         │
│  │ Scope    │ │ Systems  │         │
│  └──────────┘ └──────────┘         │
└─────────────────────────────────────┘
```

**Segmentation Rules:**

- Firewall between CDE and all other networks
- Deny-all default, explicit allow rules only
- Log all traffic crossing CDE boundary
- Annual segmentation pen test required
- Connected systems (access CDE but don't store data) are IN scope

### Field-Proven Segmentation Pattern

- VRF-based segmentation on core switches
- NGFW enforcing CDE zone policies with application-aware rules
- Separate management VLAN for payment terminals
- Jump host for admin access to CDE - no direct access

---

## Key v4.0 Changes

| Change                                                   | Impact                                  |
| -------------------------------------------------------- | --------------------------------------- |
| Targeted risk analysis for flexible requirements         | More flexibility, more documentation    |
| MFA for all CDE access (not just remote)                 | Broader MFA deployment needed           |
| Passwords: 12 chars minimum (was 7)                      | Password policy update                  |
| Automated log review mechanisms                          | Can't rely on manual daily review alone |
| Internal vulnerability scans - authenticated             | More thorough scanning required         |
| WAF mandatory for public-facing apps (was best practice) | Req 6.4.2 now mandatory                 |
| Customized approach option                               | Alternative to defined approach         |

---

## Scoping Checklist

1. Identify all locations where CHD is stored, processed, transmitted
2. Map data flows end-to-end
3. Identify all connected systems
4. Validate segmentation controls
5. Document scope in network diagram
6. Get QSA agreement on scope before assessment

**Scope Reduction Strategies:**

- Tokenization (replace PAN with non-reversible token)
- P2PE (point-to-point encryption from terminal)
- Outsource payment processing (SAQ-A eligible)
- Network segmentation (isolate CDE)

---

## Assessment Timeline

| Activity                          | Frequency                          |
| --------------------------------- | ---------------------------------- |
| External vulnerability scan (ASV) | Quarterly                          |
| Internal vulnerability scan       | Quarterly                          |
| External penetration test         | Annual + after significant changes |
| Internal penetration test         | Annual + after significant changes |
| Segmentation test                 | Every 6 months (service providers) / Annual (merchants) |
| Security awareness training       | Annual                             |
| Access review                     | Every 6 months                     |
| Firewall rule review              | Every 6 months                     |
| Risk assessment                   | Annual                             |
| Incident response test            | Annual                             |

---

## Common Audit Findings

| Finding                        | Fix                                            |
| ------------------------------ | ---------------------------------------------- |
| Default credentials on devices | Hardening checklist, automated scanning        |
| Missing log review evidence    | SIEM + documented daily review process         |
| No segmentation validation     | Quarterly seg testing with evidence            |
| Weak password policy           | Update to v4.0 requirements (12 chars)         |
| Missing patches                | Patch management with SLAs (critical: 30 days) |
| Incomplete network diagram     | Maintain with every change, date and version   |
| No file integrity monitoring   | Deploy FIM on critical CDE systems             |

---

## PCI Quick Wins

1. **Scope first** - Document and minimize scope before any technical work
2. **Network diagrams** - Keep them current, QSAs always check these first
3. **Evidence folders** - Maintain quarterly evidence ready for audit
4. **Compensating controls** - Document properly when you can't meet a requirement directly
5. **Segmentation testing** - Do it yourself before the QSA does

---

**See also:** `../iso27001/` (integrated management systems), `../networking/` (segmentation deep-dive), `../segmentation/` (CDE isolation patterns)

_Study notes, not a substitute for the PCI DSS standard._
