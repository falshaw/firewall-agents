# Playbook: Check Point → Palo Alto PAN-OS

Read `00-methodology.md` first. Agent: `paloalto-expert` (target), `firewall-architect` (planning).

## Export

- Management API: `mgmt_cli show access-rulebase name "<layer>" details-level full --format json`, `show nat-rulebase`, `show objects`
- Or the Show Package / policy export tooling for R8x
- Export per policy package and per gateway installation target — one package may install to several gateways with different interfaces

## Object Mapping

| Check Point                         | PAN-OS                                  | Notes |
| ----------------------------------- | --------------------------------------- | ----- |
| Host                                | Address (ip-netmask /32)                | Hosts may carry automatic NAT — capture it before converting |
| Network                             | Address (ip-netmask)                    | |
| Address range                       | Address (ip-range)                      | |
| Group                               | Address group                           | |
| Group with exclusion                | No direct equivalent                    | Split into explicit groups, or use negate in the rule (only if the whole cell is negated) |
| Domain object (FQDN mode)           | FQDN address object                     | Non-FQDN mode (reverse-lookup matching) behaves differently — test |
| Dynamic object                      | Dynamic address group / External Dynamic List | |
| Updatable objects (cloud services)  | EDL or predefined application / EDL feeds | |
| Access role (Identity Awareness)    | User-ID user/group in source-user       | Requires User-ID sources on PAN-OS |
| Zone object                         | Zone                                    | Only if zones were used; otherwise derive zones from topology |
| Service TCP/UDP                     | Service                                 | |
| Service with protocol handler / resource | App-ID / URL filtering / file blocking | Decide per service |
| Service-specific session timeout    | Custom service timeout or application override | |

## Rulebase Mapping

1. **Implied rules** (Global Properties) — control connections, ICMP, outgoing from gateway, etc. are not visible in the rulebase. List the ones enabled and recreate what the target actually needs (management, logging, VPN, dynamic routing) as explicit rules or interface management profiles.
2. **Layers** (R80+ ordered and inline layers) — PAN-OS has a single rulebase per vsys. Flatten:
   - Ordered layers: traffic must be accepted by **every** layer. Flatten to the intersection (or keep layer 1 as zone/network policy and fold application layer into App-ID + profiles).
   - Inline layers: parent rule match + child rule → expand into one PAN-OS rule per child, inheriting the parent's match criteria.
3. **Negated cells** → PAN-OS `negate-source` / `negate-destination`.
4. **Install On** column → split rules per target gateway/vsys.
5. **Zones:** Check Point rules usually have no zones. Derive source/destination zones from the gateway topology (which interface each network sits behind). This is the most labour-intensive step; script it from the routing table.
6. **Actions:** Accept → allow; Drop → drop; Reject → reset-both / deny; Ask/Inform (UserCheck) → URL filtering continue/override pages.
7. **Track** (Log / Detailed / Extended) → log at session end; Accounting → no direct equivalent.
8. **Time objects** → schedules.
9. **Cleanup rule** → PAN-OS interzone-default override with logging.

## NAT Mapping

| Check Point                                | PAN-OS                                           |
| ------------------------------------------ | ------------------------------------------------ |
| Automatic Hide NAT (on network/host)       | Source NAT dynamic-ip-and-port                   |
| Automatic Static NAT                        | Two rules on Check Point (source + destination) → PAN-OS bi-directional static or separate source and destination rules |
| Manual NAT rules                           | NAT rules in the same order                      |
| Hide behind gateway                        | dynamic-ip-and-port to egress interface address  |
| NAT rule order (manual first, then automatic: static before hide) | Recreate order explicitly — PAN-OS is first match |

**Security rule addresses:** on PAN-OS, security rules for inbound DNAT use the **public (pre-NAT) address** and the **post-NAT zone**. Review every rule whose destination is an object with automatic static NAT: decide whether the PAN-OS rule needs the public object, the private object, or both (flows from inside may hit the private address directly).

**Proxy ARP:** Check Point automatic NAT adds proxy ARP automatically (and manual NAT needs `local.arp` entries). On PAN-OS, ensure upstream routing or interface-subnet addresses for every public NAT IP.

## VPN Mapping

| Check Point                              | PAN-OS                                     |
| ---------------------------------------- | ------------------------------------------ |
| Star / mesh community                    | One IKE gateway + IPsec tunnel per peer    |
| Encryption domain (domain-based VPN)     | Proxy IDs per subnet pair, or route-based with peer agreement |
| Per-community crypto settings            | IKE / IPsec crypto profiles                |
| "One tunnel per gateway pair" vs per subnet pair | Matters for third-party peers — match exactly |
| Remote access (Endpoint Security VPN)    | GlobalProtect                              |

Check Point supernetting of encryption domains can produce proxy IDs that the peer does not expect. Capture the actual negotiated SAs on the source (`vpn tu`) before migrating each peer.

## Security Blades → PAN-OS

| Check Point blade         | PAN-OS                                   |
| ------------------------- | ---------------------------------------- |
| Application Control       | App-ID                                   |
| URL Filtering             | URL Filtering profile                    |
| IPS                       | Vulnerability Protection profile         |
| Anti-Virus / Anti-Bot     | Antivirus / Anti-Spyware (+ DNS Security) |
| Threat Emulation          | WildFire                                 |
| HTTPS Inspection          | Decryption policy + profiles             |
| Identity Awareness        | User-ID                                  |

## Verification

| Purpose | Check Point (source)                    | PAN-OS (target)                         |
| ------- | --------------------------------------- | --------------------------------------- |
| Policy  | Logs view, `fw ctl zdebug drop`         | `test security-policy-match ...`        |
| NAT     | `fw tab -t fwx_alloc`, logs             | `test nat-policy-match ...`             |
| Sessions | `fw ctl conntab`                       | `show session all filter ...`           |
| VPN     | `vpn tu`, `cpview`                      | `show vpn ipsec-sa`, `show vpn ike-sa`  |

## Gotchas Checklist

- [ ] Implied rules enumerated and recreated where needed
- [ ] Layers flattened and reviewed by a second engineer
- [ ] Groups-with-exclusion resolved
- [ ] Zones derived for every rule
- [ ] Automatic NAT expanded into explicit, correctly ordered NAT rules
- [ ] Inbound security rules use public address + post-NAT zone
- [ ] Encryption domains translated to proxy IDs that match the peer
- [ ] Install On targets split correctly
