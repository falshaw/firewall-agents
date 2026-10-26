# Azure Security Quick Reference

**Scope:** Azure security engineering (AZ-500 level), enterprise hybrid deployments. Azure AD is now Microsoft Entra ID; MCAS is now Defender for Cloud Apps.

---

## Azure Security Services Map

| Category       | Service                      | Purpose                                |
| -------------- | ---------------------------- | -------------------------------------- |
| **Network**    | NSG                          | L4 ACL on subnets/NICs                 |
|                | Azure Firewall               | L7 managed firewall, FQDN filtering    |
|                | WAF (App Gateway)            | L7 web app protection                  |
|                | DDoS Protection              | Volumetric attack mitigation           |
|                | Private Link                 | Private connectivity to PaaS           |
|                | Bastion                      | Secure RDP/SSH without public IP       |
| **Identity**   | Azure AD / Entra ID          | Identity provider, SSO, MFA            |
|                | Conditional Access           | Context-based access policies          |
|                | PIM                          | Just-in-time privileged access         |
|                | Managed Identities           | App-to-service auth without secrets    |
| **Data**       | Key Vault                    | Secrets, keys, certificates            |
|                | Storage Service Encryption   | Encryption at rest (always on)         |
|                | Azure Information Protection | Data classification and labeling       |
|                | Disk Encryption              | BitLocker (Windows) / DM-Crypt (Linux) |
| **Threat**     | Microsoft Defender for Cloud | CSPM + CWPP                            |
|                | Sentinel                     | Cloud-native SIEM + SOAR               |
|                | Defender for Endpoint        | EDR                                    |
|                | Defender for Identity        | Identity threat detection              |
| **Governance** | Azure Policy                 | Enforce standards, compliance          |
|                | Blueprints (deprecated)      | Use Template Specs / Deployment Stacks |
|                | Management Groups            | Hierarchy for policy inheritance       |
|                | Resource Locks               | Prevent accidental deletion            |

---

## Network Security

### NSG (Network Security Group)

**Processing Order:** Lower number = higher priority (100-4096)

```
Priority  Direction   Action   Source         Dest           Port
100       Inbound     Allow    10.0.0.0/24   VirtualNetwork  443
200       Inbound     Allow    AzureLB        VirtualNetwork  *
65000     Inbound     Allow    VNet           VNet            *  (default)
65001     Inbound     Allow    AzureLB        *               *  (default)
65500     Inbound     Deny     *              *               *  (default)
```

**Key Points:**

- Applied at subnet OR NIC level (both = both evaluated)
- Stateful - return traffic allowed automatically
- Service Tags: `Internet`, `VirtualNetwork`, `AzureLoadBalancer`, `Storage`, `Sql`
- Application Security Groups (ASG): Group VMs logically instead of by IP

### Azure Firewall

```
Rule Processing Order:
1. NAT Rules (DNAT)
2. Network Rules (L3/L4)
3. Application Rules (L7, FQDN)

Key Features:
- FQDN filtering (allow *.microsoft.com)
- Threat Intelligence integration
- IDPS (Premium SKU)
- TLS Inspection (Premium SKU)
- Forced tunneling support
- DNS Proxy
```

### Hub-Spoke Architecture

```
                    ┌─────────────┐
                    │   On-Prem   │
                    │   Network   │
                    └──────┬──────┘
                           │ ExpressRoute/VPN
                    ┌──────┴──────┐
                    │    Hub VNet  │
                    │ Azure FW    │
                    │ VPN GW      │
                    │ Bastion     │
                    └──┬───┬───┬──┘
               ┌───────┘   │   └───────┐
        ┌──────┴──┐  ┌─────┴───┐  ┌───┴──────┐
        │Spoke-Web│  │Spoke-App│  │Spoke-Data│
        │  NSG    │  │  NSG    │  │  NSG     │
        └─────────┘  └─────────┘  └──────────┘
```

**UDR (User-Defined Routes):** Force spoke traffic through Azure Firewall in hub

```
Route: 0.0.0.0/0 → Next Hop: Azure Firewall private IP
Applied to: Spoke subnet route tables
```

---

## Identity & Access

### Conditional Access Policy Components

```
Assignments (WHO + WHAT):
├── Users and groups (include/exclude)
├── Cloud apps or actions
└── Conditions
    ├── Sign-in risk (Identity Protection)
    ├── Device platforms (iOS, Windows, Android)
    ├── Locations (named locations, trusted IPs)
    ├── Client apps (browser, mobile, desktop)
    └── Device state (compliant, hybrid joined)

Access Controls (THEN):
├── Grant
│   ├── Block access
│   ├── Require MFA
│   ├── Require compliant device
│   ├── Require Hybrid Azure AD join
│   └── Require approved client app
└── Session
    ├── App enforced restrictions
    ├── Conditional Access App Control (MCAS)
    └── Sign-in frequency
```

### PIM (Privileged Identity Management)

- Just-in-time access to Azure AD and Azure roles
- Time-bound activation (e.g., 4 hours)
- Approval workflow for critical roles
- Audit trail for all activations
- Access reviews for regular cleanup

### Managed Identities

| Type            | Use Case                                       |
| --------------- | ---------------------------------------------- |
| System-assigned | Tied to resource lifecycle, 1:1                |
| User-assigned   | Shared across resources, independent lifecycle |

```
# App Service → Key Vault (no secrets in code)
App Service (Managed Identity) → Key Vault (Access Policy) → Secret
```

---

## Data Protection

### Key Vault

```
Objects stored:
- Keys (RSA, EC) - HSM-backed optional
- Secrets (connection strings, passwords, API keys)
- Certificates (TLS/SSL)

Access:
- RBAC (recommended) or Vault Access Policies
- Network rules (VNet integration, Private Endpoint)
- Soft delete + Purge protection (recover deleted secrets)

Best Practices:
- One Key Vault per environment (dev/staging/prod)
- Use Managed Identities to access (no secrets for secrets)
- Enable logging to Monitor / Sentinel
- Enable soft-delete and purge protection
```

### Encryption

| Layer           | Service                    | Default                    |
| --------------- | -------------------------- | -------------------------- |
| Storage at rest | Storage Service Encryption | Always on (Microsoft keys) |
| Disk            | Azure Disk Encryption      | BitLocker/DM-Crypt         |
| Database        | TDE (SQL/Cosmos)           | Always on                  |
| In transit      | TLS 1.2+                   | Enforced                   |
| Key management  | Key Vault / Managed HSM    | Customer choice            |

---

## Threat Protection

### Microsoft Defender for Cloud

**CSPM (Cloud Security Posture Management):**

- Secure Score (0-100%)
- Security recommendations
- Regulatory compliance dashboard (ISO, PCI, SOC2)
- Attack path analysis

**CWPP (Cloud Workload Protection):**
| Plan | Protects |
|------|----------|
| Defender for Servers | VMs, Arc-enabled servers |
| Defender for App Service | Web apps |
| Defender for SQL | SQL DBs |
| Defender for Storage | Blob, Files |
| Defender for Containers | AKS, container registries |
| Defender for Key Vault | Key Vault operations |
| Defender for DNS | DNS anomalies |

### Microsoft Sentinel (SIEM + SOAR)

```
Data Flow:
Data Connectors → Log Analytics Workspace → Analytics Rules → Incidents → Playbooks

Key Components:
- Data Connectors: Azure AD, M365, AWS, Syslog, CEF
- Workbooks: Visualization dashboards
- Analytics Rules: KQL-based detection
- Hunting Queries: Proactive threat hunting
- Playbooks: Logic Apps for automated response
- Watchlists: Reference data (IOCs, VIP users)
```

**KQL Quick Reference:**

```kql
// Failed sign-ins last 24h
SigninLogs
| where TimeGenerated > ago(24h)
| where ResultType != "0"
| summarize FailedCount=count() by UserPrincipalName, IPAddress
| where FailedCount > 10
| sort by FailedCount desc

// Suspicious resource creation
AzureActivity
| where OperationNameValue endswith "write"
| where ActivityStatusValue == "Succeeded"
| where TimeGenerated > ago(1h)
| summarize count() by Caller, ResourceGroup
```

---

## Governance

### Azure Policy

```
Definition → Assignment → Evaluation

Effects:
- Deny: Block non-compliant resource creation
- Audit: Log non-compliance (no block)
- DeployIfNotExists: Auto-remediate
- Modify: Add/change tags or properties
- Disabled: Policy off

Common Policies:
- Require tag on resources
- Allowed locations
- Allowed VM SKUs
- Require encryption on storage accounts
- Deny public IP creation
```

### Management Group Hierarchy

```
Root Management Group
├── Production
│   ├── Subscription-Prod-EU
│   └── Subscription-Prod-US
├── Non-Production
│   ├── Subscription-Dev
│   └── Subscription-Staging
└── Sandbox
    └── Subscription-Sandbox
```

Policies at higher levels cascade down.

---

## Common Patterns

**Regulated hub-spoke:**

- Hub VNet with Azure Firewall; spokes peer to the hub and route egress through it (UDR 0.0.0.0/0)
- Conditional Access: location + device compliance + MFA
- Sentinel for monitoring across Azure and on-prem sources

**Hybrid identity:**

- Microsoft Entra ID synced with on-prem AD (Entra Connect / Cloud Sync)
- Key Vault for application secrets
- NSG + ASG for microsegmentation inside a VNet

---

**See also:** `../iso27001/` (compliance mapping), `../togaf/` (architecture framework)

_Reference for Azure security design and AZ-500 study. Maps to enterprise hybrid deployments._
