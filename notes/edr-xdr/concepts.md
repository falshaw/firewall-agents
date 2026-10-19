# EDR / XDR Concepts for Network & Firewall Engineers

**Scope:** Vendor-neutral endpoint detection and response concepts, with how they connect to firewalls, SIEM and SOAR. Product examples (SentinelOne Singularity, CrowdStrike Falcon, Microsoft Defender for Endpoint, Palo Alto Cortex XDR, Trellix) are named only to anchor terminology.

---

## EPP vs EDR vs XDR

| Layer   | Question it answers                         | Typical capabilities                                        |
| ------- | ------------------------------------------- | ----------------------------------------------------------- |
| **EPP** | Can we stop it before it runs?              | Signature + static ML scanning, exploit prevention, device control |
| **EDR** | What happened after it ran, and how do we respond? | Process/file/registry/network telemetry, behavioural detection, kill/quarantine/remediate |
| **XDR** | What happened across endpoints, network, cloud and identity? | EDR + firewall/NDR + cloud + identity telemetry correlated in one data lake |

---

## Detection Without Signatures

| Engine            | When it runs     | What it looks at                                                   |
| ----------------- | ---------------- | ------------------------------------------------------------------ |
| Static ML         | Before execution | File features (headers, imports, entropy) scored by a local model   |
| Behavioural       | At runtime       | Process injection, privilege escalation, credential access, lateral movement, mass file encryption |
| Cloud reputation  | Either           | Hash/URL/IP reputation lookups (optional on some agents)            |

**Design point:** agents that keep both engines local still detect and respond when the endpoint is offline or air-gapped. Agents that depend on cloud lookups degrade when disconnected — ask the vendor which detections need connectivity.

---

## Attack-Story Correlation

Most modern EDRs link every related event (parent/child processes, file writes, registry changes, network connections) into a single graph or timeline per incident. SentinelOne calls this *Storyline*; other vendors use process trees or "incident graphs".

Why it matters for responders:

- One alert instead of hundreds of disconnected events
- Clear root cause (initial process, persistence, spread)
- Response actions can target the whole chain, not one process

---

## Response Actions

| Action             | Effect                                                    | Network relevance                           |
| ------------------ | --------------------------------------------------------- | ------------------------------------------- |
| Kill               | Terminate malicious process tree                          | —                                           |
| Quarantine         | Move file to protected store                              | —                                           |
| Remediate          | Revert registry, file and service changes made by the chain | —                                         |
| Rollback           | Restore files from OS snapshots (e.g. VSS) after ransomware | —                                         |
| Network quarantine | Agent-enforced host isolation; only the management console stays reachable | Complements switch/NAC/firewall isolation |

**Autonomous vs analyst-driven:** policies decide whether the agent acts on its own (detect-and-protect) or only alerts (detect-only). Start new deployments in detect-only, tune exclusions, then enable protect mode by group.

---

## Custom Detections

- Rule engines run analyst-written queries continuously against incoming telemetry (SentinelOne STAR, Defender custom detections, Falcon custom IOAs).
- Each rule can alert only or trigger a response (kill, isolate).
- Good practice: version-control rules, map each to a MITRE ATT&CK technique, test against replayed telemetry before enabling response.

---

## Data Lake & Retention

- Telemetry lands in a vendor data lake or the SIEM; retention depends on licence tier (commonly 14–30 days hot, longer on higher tiers or via SIEM export).
- Threat hunting needs retention long enough to cover dwell time — plan for at least 90 days somewhere searchable.
- Natural-language query assistants (e.g. Purple AI, Copilot for Security) translate questions into the native query language; always review the generated query.

---

## Identity Threat Detection (ITDR)

Detects attacks on Active Directory / Entra ID:

- Credential theft and Pass-the-Hash / Pass-the-Ticket
- Kerberoasting, AS-REP roasting
- DCSync, Golden/Silver Ticket
- Deception: decoy accounts and credentials planted on endpoints

---

## MITRE ATT&CK

- Knowledge base of adversary tactics, techniques and procedures (TTPs).
- Use it to map detection coverage, not as a product scorecard. Vendor participation in the public evaluations changes year to year; read the raw results, not marketing summaries.

---

## Deployment Checklist

1. Inventory OS versions and kernel versions (Linux agents are kernel-sensitive)
2. Pilot group per OS, detect-only policy
3. Deploy via existing tooling (GPO, Intune/SCCM, Ansible, Chef/Puppet, golden images)
4. Exclusions for backup, database and AV-conflicting software — document each with an owner
5. Group-based policies with inheritance (servers vs workstations vs VDI)
6. Move to protect mode in waves; monitor helpdesk tickets
7. Verify agent health dashboards and tamper protection

---

## Integration with Network Security

| Integration     | Typical mechanism                                       | Use case                                         |
| --------------- | ------------------------------------------------------- | ------------------------------------------------ |
| SIEM            | Syslog/CEF, API pull, native connectors                 | Correlate endpoint alerts with firewall/proxy logs |
| SOAR            | REST API playbooks                                      | Auto-isolate host + block IOC on firewall        |
| Firewall        | External dynamic lists / threat feeds, tag-based dynamic groups | Push malicious IPs/URLs from EDR to NGFW blocklists; tag infected hosts for quarantine policy |
| NAC             | Posture/compliance API                                  | Deny network access to hosts without a healthy agent |
| Identity / IdP  | Conditional access signals                              | Block sign-in from compromised devices           |
| Ticketing       | ServiceNow / Jira connectors                            | Incident tracking                                |

**Firewall pattern:** EDR detects C2 → SOAR adds the C2 IP to an external dynamic list → NGFW blocks it for every host, including those without an agent (IoT, OT, unmanaged).

---

**See also:** `../azure-security/` (Defender + Sentinel), `../networking/hardening.md` (logging and monitoring)
