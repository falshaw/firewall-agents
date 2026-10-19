# Playbook: Cisco ASA → Palo Alto PAN-OS

Read `00-methodology.md` first. Agents: `cisco-security-expert` (source), `paloalto-expert` (target).

## Object Mapping

| ASA                                       | PAN-OS                                | Notes |
| ----------------------------------------- | ------------------------------------- | ----- |
| `object network X` `host` / `subnet` / `range` | Address object (ip-netmask / ip-range) | Name limit 63 chars |
| `object network X` `fqdn`                 | Address object type FQDN              | Resolution interval differs |
| `object-group network`                    | Address group (static)                | Nested groups supported |
| `object service` / `object-group service` | Service / service group               | `tcp-udp` → create two services |
| `eq`, `range`, `gt`, `lt` port operators  | Service port / port range             | `gt 1023` → `1024-65535` |
| ICMP in ACL (`icmp`, `echo`)              | Application `icmp` / `ping`, service `application-default` | |
| `time-range`                              | Schedule                              | |
| `remark`                                  | Rule description                      | Preserve change refs |
| `inactive` ACE                            | Disabled rule                         | |

## Policy Mapping

ASA ACLs are bound **per interface, per direction**; PAN-OS rules are **zone pair** based.

1. Map each ASA `nameif` to a zone (one-to-one first; merge later if desired).
2. For each inbound interface ACL: `access-group X in interface Y` → rules with `from` = zone(Y), `to` = zone determined by the route to the destination.
   - An ACE with destination `any` can reach several egress zones — either use `to any` or split into one rule per real egress zone (preferred for clarity).
3. Global ACL (`access-group X global`) → `from any to any` rules, placed after interface rules (ASA evaluates interface ACLs first).
4. Implicit permits from **security levels** (higher → lower with no ACL bound) do not exist on PAN-OS. Enumerate them as explicit rules.
5. `same-security-traffic permit inter-interface` / `intra-interface` → explicit intrazone/interzone rules (PAN-OS intrazone is allowed by default; decide whether to override).
6. ASA implicit deny → PAN-OS `interzone-default` deny; override it to enable logging.

## NAT Mapping

### Order

PAN-OS NAT is a single top-down list (first match). Recreate ASA order:

1. Section 1 — twice (manual) NAT, in config order
2. Section 2 — object (auto) NAT: static before dynamic, then longest prefix first
3. Section 3 — twice NAT `after-auto`

### Address semantics (the critical difference)

| Item                    | ASA 8.3+                     | PAN-OS                                   |
| ----------------------- | ---------------------------- | ---------------------------------------- |
| ACL destination for DNAT | Real (private) IP           | Security rule: **pre-NAT (public) IP**   |
| Zone in rule            | Ingress interface ACL        | Security rule: **post-NAT zone** (zone of the real server) |
| NAT rule zones          | `(real_ifc,mapped_ifc)`      | NAT rule: pre-NAT source zone and pre-NAT destination zone (zone the public IP routes to) |

**Worked example — published web server**

ASA:

```
object network SRV-WEB-01
 host 10.20.0.10
 nat (dmz,outside) static 203.0.113.10 service tcp https https
access-list OUTSIDE_IN extended permit tcp any object SRV-WEB-01 eq https
```

PAN-OS:

```
set address SRV-WEB-01 ip-netmask 10.20.0.10/32
set address SRV-WEB-01-PUB ip-netmask 203.0.113.10/32

set rulebase nat rules DNAT-WEB-01 from untrust to untrust source any destination SRV-WEB-01-PUB service service-https destination-translation translated-address SRV-WEB-01

set rulebase security rules ALLOW-WEB-01-IN from untrust to dmz source any destination SRV-WEB-01-PUB application [ ssl web-browsing ] service service-https action allow
```

Note: NAT rule `to untrust` (the public IP routes out untrust); security rule `to dmz` and destination = the **public** object.

### Other NAT constructs

| ASA                                                         | PAN-OS                                         |
| ----------------------------------------------------------- | ---------------------------------------------- |
| `nat (inside,outside) dynamic interface`                    | Source translation: dynamic-ip-and-port, interface address |
| Dynamic NAT to pool                                         | dynamic-ip (pool) or dynamic-ip-and-port (pool) |
| Static 1:1 (object NAT static)                              | Static IP source translation, `bi-directional yes` (or separate DNAT rule — clearer) |
| NAT exemption / identity NAT (`source static X X destination static Y Y`) | Usually nothing needed (no NAT rule = no translation). Add a **no-NAT rule above** any broader source-NAT rule that would otherwise match |
| `no-proxy-arp`, `route-lookup`                              | Not applicable; see proxy ARP note below       |
| Port forwarding (`service tcp 8443 443`)                    | Destination translation with translated port   |

**Proxy ARP:** ASA proxy-ARPs for mapped addresses by default. PAN-OS answers ARP for NAT addresses in the interface's subnet; for public addresses outside the interface subnet, the upstream router needs a route to the firewall.

## VPN Mapping

| ASA                         | PAN-OS                                         |
| --------------------------- | ---------------------------------------------- |
| `crypto ikev1/ikev2 policy` | IKE crypto profile                             |
| `crypto ipsec ikev2 ipsec-proposal` / transform-set | IPsec crypto profile             |
| `tunnel-group <peer> ipsec-attributes` (PSK) | IKE gateway                   |
| `crypto map` entry + match ACL | IPsec tunnel + tunnel interface + **proxy IDs** matching the ACL (when the peer stays policy-based) + static routes into the tunnel |
| VTI (`interface Tunnel`)     | Tunnel interface, no proxy IDs needed if both sides route-based |
| AnyConnect                  | GlobalProtect portal + gateway (separate workstream: client rollout, auth profile, HIP) |

Each crypto ACL line becomes a proxy ID pair on PAN-OS. Mismatched proxy IDs are the most common cause of "phase 1 up, phase 2 down" after migration.

## Inspection / Application Layer

- ASA `inspect` (FTP, SIP, DNS, ICMP) → App-ID + ALGs on PAN-OS. Check SIP/H.323 behaviour explicitly; disable the SIP ALG if the PBX/SBC handles NAT itself.
- Plan App-ID adoption **after** cutover: migrate as port-based (`application any`, explicit service), then use Policy Optimizer to convert rules to App-ID with `application-default`.

## Verification Commands

| Purpose           | ASA (source)                                            | PAN-OS (target)                                                 |
| ----------------- | ------------------------------------------------------- | --------------------------------------------------------------- |
| Policy            | `packet-tracer input outside tcp 198.51.100.20 40000 203.0.113.10 443 detailed` | `test security-policy-match from untrust to dmz source 198.51.100.20 destination 203.0.113.10 protocol 6 destination-port 443` |
| NAT               | `show nat detail`, `show xlate`                         | `test nat-policy-match from untrust to untrust source 198.51.100.20 destination 203.0.113.10 protocol 6 destination-port 443` |
| Sessions          | `show conn address 10.20.0.10`                          | `show session all filter destination 203.0.113.10`              |
| VPN               | `show crypto ipsec sa peer 198.51.100.1`                | `show vpn ipsec-sa tunnel <name>`, `test vpn ipsec-sa tunnel <name>` |

## Gotchas Checklist

- [ ] Every DNAT rule rewritten with public destination + post-NAT zone
- [ ] Security-level implicit permits enumerated
- [ ] `tcp-udp` services split
- [ ] Global ACL placed after interface-derived rules
- [ ] NAT exemption replaced by no-NAT rules only where needed
- [ ] Proxy IDs created for every policy-based VPN peer
- [ ] Interzone default rule overridden to log
- [ ] Rule descriptions carry the original ACL name/line for traceability
- [ ] Object names within 63 characters, no illegal characters
