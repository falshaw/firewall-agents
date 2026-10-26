# Playbook: Palo Alto PAN-OS → Fortinet FortiGate

Read `00-methodology.md` first. Agents: `paloalto-expert` (source), `fortinet-expert` (target).

## Decide First

| Decision          | Why it matters                                                   |
| ----------------- | ---------------------------------------------------------------- |
| NGFW mode         | **Profile-based** (default): policies match ports; applications are controlled by an application control sensor. **Policy-based**: security policies match applications directly — closest to PAN-OS App-ID rules |
| NAT mode          | Policy-based NAT (VIPs in policy) or central NAT                 |
| Zones             | FortiOS `system zone` maps cleanly to PAN-OS zones; set `intrazone deny` to match PAN-OS intrazone behaviour if desired |
| Management        | Panorama device groups/templates → FortiManager ADOM, policy packages, device database / provisioning templates |

## Export

- Panorama: export device-group pre/post rules and shared objects (XML API `show` or `config` export) — remember the effective rulebase is **shared pre → DG pre → local → DG post → shared post**
- Firewall: `show config running` (XML), or `set cli config-output-format set` in operational mode, then `configure` and `show` for set-format output

## Object Mapping

| PAN-OS                          | FortiOS                                         |
| ------------------------------- | ----------------------------------------------- |
| Address (ip-netmask/range/fqdn) | `firewall address`                              |
| Address group (static)          | `firewall addrgrp`                              |
| Dynamic address group (tags)    | Dynamic addresses via fabric connectors (FSSO, SDN connectors) or rebuild as static |
| External Dynamic List           | `system external-resource` (threat feeds)       |
| Service / service group         | `firewall service custom` / `group`             |
| `application-default`           | No equivalent — use explicit ports, or rely on policy-based mode + application match |
| Application / application group / filter | Application control sensor entries or policy-based `application` / `app-category` |
| Tags                            | Object tags / comments                          |
| Region objects                  | Geography address objects (`set type geography`) |
| Schedule                        | `firewall schedule`                             |

Name length: PAN-OS allows 63 characters, FortiOS address names up to 79 — usually no truncation; check special characters.

## Policy Mapping

1. Zone pairs → `srcintf`/`dstintf` using FortiOS zones.
2. Order: flatten Panorama pre/local/post rules into one sequence before conversion.
3. Rules with App-ID:
   - **Policy-based mode:** `config firewall security-policy` with `set application` / `set app-category`; ports via `service`.
   - **Profile-based mode:** port-based `firewall policy` + application control sensor that allows the listed applications and blocks the rest for that policy.
4. `source-user` / User-ID → FSSO groups or FortiAuthenticator / RADIUS single sign-on groups (`set groups`).
5. HIP profiles (GlobalProtect) → EMS ZTNA tags.
6. Rule actions: allow → accept; deny/drop → deny; reset-client/server → deny with `send-deny-packet enable`.
7. Default intrazone allow / interzone deny → explicit policies + implicit deny with logging.

## Security Profile Mapping

| PAN-OS                         | FortiOS                                   |
| ------------------------------ | ----------------------------------------- |
| Antivirus                      | AntiVirus profile                         |
| Anti-Spyware (+ DNS Security)  | IPS sensor (botnet C&C) + DNS filter profile |
| Vulnerability Protection       | IPS sensor                                |
| URL Filtering                  | Web filter profile                        |
| File Blocking                  | File filter profile                       |
| WildFire Analysis              | AntiVirus with FortiSandbox / FortiGuard outbreak prevention |
| Data Filtering                 | DLP profile                               |
| Decryption policy + profile    | SSL/SSH inspection profile (`deep-inspection`) + exemptions |
| Security profile group         | Apply profiles per policy (or via FortiManager templates) |

## NAT Mapping

| PAN-OS NAT rule                          | FortiOS (policy-based NAT)                       |
| ---------------------------------------- | ------------------------------------------------ |
| Source dynamic-ip-and-port → interface   | Policy `set nat enable`                          |
| Source dynamic-ip-and-port → pool        | `firewall ippool type overload` + `set poolname` |
| Source dynamic-ip                        | `firewall ippool type one-to-one` or fixed-port-range |
| Source static, bi-directional            | VIP (static NAT) + `set nat-source-vip enable`   |
| Destination translation (+ port)         | VIP with `portforward enable`                    |
| U-turn NAT                               | VIP with `extintf any` + internal-to-internal policy using the VIP |
| No-NAT rule above broader rule           | Policy with `set nat disable` placed above        |

**Address semantics:** PAN-OS inbound security rules use the **public** address with the **post-NAT zone**. FortiOS (policy-based NAT) inbound policies use the **VIP object** as `dstaddr` with `dstintf` = interface/zone of the real server. In central NAT mode, use the real server address.

## VPN Mapping

| PAN-OS                              | FortiOS                                        |
| ----------------------------------- | ---------------------------------------------- |
| IKE gateway + IKE crypto profile    | `vpn ipsec phase1-interface`                   |
| IPsec tunnel + IPsec crypto profile | `vpn ipsec phase2-interface`                   |
| Proxy IDs                           | Phase 2 selectors (one phase2 per proxy-ID pair) |
| Tunnel interface + static/dynamic routes | Tunnel interface (auto-created) + routes  |
| GlobalProtect portal/gateway        | FortiClient + EMS (IPsec/SSL-VPN) or ZTNA      |

## Routing

- Virtual routers (or logical routers with Advanced Routing Engine) → FortiOS VRFs (`set vrf` on interfaces) or VDOMs if full separation is required
- PBF rules → `router policy` or SD-WAN rules
- Redistribution profiles → `router ospf/bgp` redistribute + route-maps

## Verification

| Purpose | PAN-OS (source)                      | FortiOS (target)                                  |
| ------- | ------------------------------------ | ------------------------------------------------- |
| Policy  | `test security-policy-match ...`     | `diagnose firewall iprope lookup ...`             |
| NAT     | `test nat-policy-match ...`          | `show firewall vip`, session list        |
| App     | `show session id <id>` (application) | Session list (`app_list`), FortiView Applications |
| VPN     | `show vpn ipsec-sa`                  | `diagnose vpn tunnel list`                        |

## Gotchas Checklist

- [ ] NGFW mode chosen (profile- vs policy-based) before conversion
- [ ] Panorama pre/post/shared rule order flattened correctly
- [ ] `application-default` rules given explicit ports or application matching
- [ ] DAGs/EDLs rebuilt as connector-based or static objects
- [ ] Inbound DNAT policies use VIPs (policy-based NAT)
- [ ] Decryption exemptions carried over
- [ ] User-ID sources replaced by FSSO/RSSO and tested
- [ ] GlobalProtect replacement tested per client OS
