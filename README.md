# firewall-agents

Vendor expert personas, migration playbooks and study notes for network and firewall engineers who use [Claude Code](https://docs.anthropic.com/en/docs/claude-code).

- **Agents** — Palo Alto, Cisco, Fortinet, Juniper SRX, F5 BIG-IP and a multi-vendor architect, packaged as Claude Code subagents. You can also paste them into any LLM as plain prompts.
- **Playbooks** — how to move from one vendor to another: rule, NAT and object translation, cutover, rollback and validation.
- **Notes** — cheat sheets by topic: routing and switching, hardening, Cisco ACI, segmentation, PCI DSS, ISO 27001, Azure security, TOGAF and EDR/XDR.

Every agent works on the same rule: **it writes config for a human to review and never commits, saves, installs or reboots anything.**

---

## What's Inside

```
firewall-agents/
├── agents/                     Claude Code subagents (YAML frontmatter + prompt)
│   ├── firewall-architect.md
│   ├── paloalto-expert.md
│   ├── cisco-security-expert.md
│   ├── fortinet-expert.md
│   ├── juniper-srx-expert.md
│   └── f5-bigip-expert.md
├── commands/
│   └── fw-agent.md             Optional /fw-agent <vendor> slash command
├── playbooks/                  Vendor-to-vendor migration procedures
└── notes/<topic>/              Study notes and cheat sheets
```

### Agents

| Agent                   | Use it for                                                         |
| ----------------------- | ------------------------------------------------------------------ |
| `firewall-architect`    | Multi-vendor design, migration planning, NAT semantics, cutover/rollback, validation |
| `paloalto-expert`       | PAN-OS, Panorama, GlobalProtect, App-ID adoption, migrations onto PAN-OS |
| `cisco-security-expert` | ASA, Secure Firewall (FTD/FMC), IOS-XE security, ISE, ACI          |
| `fortinet-expert`       | FortiGate, FortiManager, SD-WAN, ZTNA, VIP/central NAT             |
| `juniper-srx-expert`    | Junos SRX policies, NAT, route-based VPN, chassis cluster          |
| `f5-bigip-expert`       | LTM, BIG-IP DNS, iRules, AS3, SSL profiles, Advanced WAF           |

### Playbooks

| Playbook                                                   | Covers                                           |
| ---------------------------------------------------------- | ------------------------------------------------ |
| [00-methodology](playbooks/00-methodology.md)              | Discover → clean → design → convert → validate → cutover → hypercare |
| [asa-to-panos](playbooks/asa-to-panos.md)                  | Cisco ASA → Palo Alto                            |
| [checkpoint-to-panos](playbooks/checkpoint-to-panos.md)    | Check Point → Palo Alto                          |
| [asa-to-ftd](playbooks/asa-to-ftd.md)                      | Cisco ASA → Secure Firewall Threat Defense       |
| [asa-to-fortigate](playbooks/asa-to-fortigate.md)          | Cisco ASA → FortiGate                            |
| [panos-to-fortigate](playbooks/panos-to-fortigate.md)      | Palo Alto → FortiGate                            |
| [to-juniper-srx](playbooks/to-juniper-srx.md)              | Palo Alto or Cisco ASA → Juniper SRX             |
| [cutover-and-rollback](playbooks/cutover-and-rollback.md)  | Window prep, go/no-go, ARP/session impact, rollback triggers, per-vendor safety nets |
| [validation-checklist](playbooks/validation-checklist.md)  | Baseline counts, static review, simulated policy tests, live checks |

### Topic Index (notes)

| Topic            | Files                                                                                         |
| ---------------- | --------------------------------------------------------------------------------------------- |
| Networking       | [cheatsheet](notes/networking/cheatsheet.md) · [cisco-switching-routing](notes/networking/cisco-switching-routing.md) · [hardening](notes/networking/hardening.md) |
| Juniper SRX      | [cheatsheet](notes/juniper-srx/cheatsheet.md)                                                 |
| Cisco ACI        | [aci-quickstart](notes/cisco-aci/aci-quickstart.md) · [aci-deep-dive](notes/cisco-aci/aci-deep-dive.md) |
| Segmentation     | [regulated-segmentation](notes/segmentation/regulated-segmentation.md) (PCI, DORA, SWIFT CSP, NIS2 mapping) |
| PCI DSS          | [cheatsheet](notes/pci-dss/cheatsheet.md)                                                     |
| ISO 27001        | [cheatsheet](notes/iso27001/cheatsheet.md)                                                    |
| Azure security   | [cheatsheet](notes/azure-security/cheatsheet.md)                                              |
| TOGAF            | [cheatsheet](notes/togaf/cheatsheet.md)                                                       |
| EDR / XDR        | [concepts](notes/edr-xdr/concepts.md)                                                         |

---

## Installing the Agents

Claude Code loads subagents from `.claude/agents/` (one project) or `~/.claude/agents/` (every project).

**Every project (user level):**

```bash
git clone https://github.com/<your-account>/firewall-agents.git
mkdir -p ~/.claude/agents ~/.claude/commands
cp firewall-agents/agents/*.md ~/.claude/agents/
cp firewall-agents/commands/fw-agent.md ~/.claude/commands/   # optional
```

**One project only:**

```bash
mkdir -p .claude/agents .claude/commands
cp /path/to/firewall-agents/agents/*.md .claude/agents/
cp /path/to/firewall-agents/commands/fw-agent.md .claude/commands/   # optional
```

Windows PowerShell: `Copy-Item firewall-agents\agents\*.md $HOME\.claude\agents\`

Restart Claude Code (or run `/agents`) to confirm they are listed.

### Using them

- **Automatic delegation:** describe the task ("convert this ASA config to PAN-OS set commands") and Claude Code picks the matching subagent from its `description`.
- **Explicit:** "Use the juniper-srx-expert agent to review this NAT config."
- **Persona for the whole session:** `/fw-agent paloalto`, `/fw-agent cisco`, `/fw-agent fortinet`, `/fw-agent juniper`, `/fw-agent f5` or `/fw-agent architect`. The command loads the persona into the main conversation instead of starting a subagent.
- **As a plain prompt:** drop the YAML block between the `---` lines and paste the rest as a system prompt or the first message in any LLM chat.

Each agent's `tools:` line allows reading, searching, writing local files and fetching web docs. It does **not** allow shell access, so an agent cannot reach a device by itself. Keep it that way for anything that can talk to production.

---

## Using the Playbooks

1. Read [00-methodology](playbooks/00-methodology.md) and fill in the source inventory.
2. Open the vendor-pair playbook and work through its mapping tables and gotchas checklist.
3. Give the agents the playbook and your sanitised config export. Example:
   > "Using playbooks/asa-to-panos.md, convert configs/edge-asa.txt to PAN-OS set commands. Produce a mapping file from source ACL line to target rule, and list every NAT rule you were unsure about."
4. Check the output against [validation-checklist](playbooks/validation-checklist.md).
5. Plan the window with [cutover-and-rollback](playbooks/cutover-and-rollback.md).

**Sanitise configs before you share them with any AI tool.** Remove or replace public IPs, hostnames, PSKs, SNMP communities, password hashes and certificates. The `.gitignore` blocks common config and capture file types so they do not get committed by accident.

---

## Conventions

- Example addresses use only RFC 1918 private ranges and the RFC 5737 documentation ranges (`192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24`). Example hostnames use `example.com`.
- Object names follow a generic pattern: `SRV-<role>-<nn>`, `NET-<zone>-<cidr>`, `GRP-<purpose>`.
- CLI syntax changes between software versions. Check each snippet against the vendor documentation for the version you run before you use it.

## Disclaimer

- The content is provided as is (see [LICENSE](LICENSE)). You are responsible for testing and approving any change you make to your own network.
- Palo Alto Networks, PAN-OS, Panorama, Cisco, ASA, Firepower, ACI, Fortinet, FortiGate, FortiOS, Juniper, Junos, SRX, F5, BIG-IP, Check Point, Microsoft, Azure, SentinelOne, TOGAF, PCI DSS, ISO and all other product names and trademarks belong to their respective owners. This project is independent. It is not affiliated with, endorsed by or sponsored by any vendor or standards body.
- The notes are original summaries written as study and field material. They are not copies of vendor documentation, standards text or course material. For anything authoritative, use the official documents.

## License

MIT, Copyright (c) 2026 Nick Falshaw. See [LICENSE](LICENSE).
