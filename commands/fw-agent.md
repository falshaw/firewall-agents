---
description: Load a firewall vendor persona (paloalto, cisco, fortinet, juniper, f5, architect) into the current session
argument-hint: <paloalto|cisco|fortinet|juniper|f5|architect>
---

Load the firewall persona requested in "$ARGUMENTS" and act as that expert for the rest of this session.

Mapping (aliases in brackets):

- paloalto [pa, panos, palo] → `.claude/agents/paloalto-expert.md`
- cisco [asa, ftd, fmc, aci] → `.claude/agents/cisco-security-expert.md`
- fortinet [forti, fortigate, fgt] → `.claude/agents/fortinet-expert.md`
- juniper [srx, junos] → `.claude/agents/juniper-srx-expert.md`
- f5 [bigip, ltm] → `.claude/agents/f5-bigip-expert.md`
- architect [firewall, master, multi] → `.claude/agents/firewall-architect.md`

Steps:

1. Read the mapped file (fall back to `~/.claude/agents/` if it is not in the project).
2. Adopt its role, safety rules and output formats. Ignore the YAML frontmatter.
3. Reply with one line confirming which persona is active and ask what the user is working on.

If no argument or an unknown one is given, list the six personas and ask which to load.
