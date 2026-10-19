# Playbook: Palo Alto PAN-OS or Cisco ASA → Juniper SRX

Read `00-methodology.md` first. Agents: `paloalto-expert` / `cisco-security-expert` (source), `juniper-srx-expert` (target).

## SRX Order of Operations

```
screens → static NAT → destination NAT → route lookup → zone lookup → security policy → reverse static NAT → source NAT
```

Security policies therefore match the **post-DNAT (real) destination** and the **zone the real address routes to**.

| Source platform | Destination address in source rule | What to write on SRX |
| --------------- | ---------------------------------- | -------------------- |
| Cisco ASA 8.3+  | Real IP                            | Same real IP — direct mapping |
| PAN-OS          | **Public** (pre-NAT) IP            | **Rewrite to the real IP** (the DNAT target) |

This one difference breaks most PAN-OS → SRX conversions that are done by search-and-replace.

## Object Mapping

| PAN-OS / ASA                          | Junos SRX                                     |
| ------------------------------------- | --------------------------------------------- |
| Address / `object network`            | `security address-book global address`       |
| Address group / `object-group network`| `address-set`                                 |
| FQDN address                          | `address <name> dns-name <fqdn>`              |
| Service / `object service`            | `applications application` (protocol, destination-port) |
| Service group                         | `applications application-set`                |
| Predefined services (HTTP/HTTPS/DNS)  | `junos-http`, `junos-https`, `junos-dns-udp` … |
| App-ID                                | AppSecure: unified policies with `dynamic-application` (Junos 18.2+), or AppFW rule sets |
| Schedule / time-range                 | `schedulers scheduler` + policy `scheduler-name` |

Use the **global address book** for new builds — it is required for global policies and avoids duplicating objects per zone.

## Policy Mapping

| Source                                   | SRX                                             |
| ---------------------------------------- | ----------------------------------------------- |
| PAN-OS zone pair rule                    | `security policies from-zone A to-zone B policy <name>` |
| PAN-OS `from any` / `to any`             | Global policy (`security policies global`) with `from-zone`/`to-zone` match lists, or one policy per zone pair |
| ASA interface ACL (inbound)              | One zone-pair context per (ingress zone, egress zone by routing) |
| ASA security-level implicit permit       | Explicit policies                               |
| Rule order                               | Top-down **within a zone pair**; zone-pair policies before global policies |
| Disabled rule                            | `deactivate security policies ...`              |
| Log at session end                       | `then log session-close`                        |
| Profiles (AV/IPS/URL)                    | `then permit application-services` (IDP, UTM/content-security, SSL proxy) |
| User-ID / identity                       | `source-identity` with JIMS / Identity Management |

Default: SRX has an implicit `default-policy deny-all`. Add an explicit global deny with `log session-init` to see drops.

## NAT Mapping

| PAN-OS / ASA                              | Junos SRX                                                        |
| ----------------------------------------- | ---------------------------------------------------------------- |
| Source PAT to interface                   | `security nat source rule-set ... then source-nat interface`      |
| Source NAT to pool                        | `source pool` + `then source-nat pool <name>` (`port no-translation` for 1:1-style) |
| Destination NAT / port forward            | `destination pool` (address + port) + destination rule-set       |
| Static 1:1 bi-directional                 | `security nat static rule-set` (`static-nat prefix`)             |
| No-NAT / NAT exemption                    | `then source-nat off` rule placed above the broader rule         |
| Rule order                                | The most specific rule-set context wins (interface > zone > routing-instance); within a rule-set, rules are first match in order — use `insert ... before` |

**Proxy ARP is not automatic.** For any NAT address that is in the interface subnet but is not the interface IP:

```
set security nat proxy-arp interface xe-0/0/0.0 address 203.0.113.10/32
```

ASA and PAN-OS did this implicitly; forgetting it on SRX shows up as "NAT rule correct, no traffic arrives".

### Worked example — published web server from PAN-OS

PAN-OS source:

```
NAT:      from untrust to untrust  dst 203.0.113.10  → translate dst to 10.20.0.10
Security: from untrust to dmz      dst 203.0.113.10  app ssl  allow
```

SRX target:

```
set security address-book global address SRV-WEB-01 10.20.0.10/32
set security nat destination pool DP-WEB-01 address 10.20.0.10/32 port 443
set security nat destination rule-set RS-UNTRUST-IN from zone untrust
set security nat destination rule-set RS-UNTRUST-IN rule DNAT-WEB-01 match destination-address 203.0.113.10/32
set security nat destination rule-set RS-UNTRUST-IN rule DNAT-WEB-01 match destination-port 443
set security nat destination rule-set RS-UNTRUST-IN rule DNAT-WEB-01 then destination-nat pool DP-WEB-01
set security nat proxy-arp interface xe-0/0/0.0 address 203.0.113.10/32

set security policies from-zone untrust to-zone dmz policy ALLOW-WEB-01-IN match source-address any
set security policies from-zone untrust to-zone dmz policy ALLOW-WEB-01-IN match destination-address SRV-WEB-01
set security policies from-zone untrust to-zone dmz policy ALLOW-WEB-01-IN match application junos-https
set security policies from-zone untrust to-zone dmz policy ALLOW-WEB-01-IN then permit
set security policies from-zone untrust to-zone dmz policy ALLOW-WEB-01-IN then log session-close
```

Destination in the policy is the **real** server `SRV-WEB-01`, not the public address.

## VPN Mapping

| Source                                   | SRX                                                       |
| ---------------------------------------- | --------------------------------------------------------- |
| IKE crypto profile / IKE policy          | `security ike proposal` + `ike policy`                    |
| IPsec crypto profile / transform set     | `security ipsec proposal` + `ipsec policy`                |
| IKE gateway / tunnel-group               | `security ike gateway`                                    |
| Tunnel interface                         | `st0.<unit>` bound to the VPN, placed in a zone           |
| Proxy IDs / crypto ACL                   | `traffic-selector` (route-based, multiple) or `proxy-identity` per VPN |
| GlobalProtect / AnyConnect               | Juniper Secure Connect (separate workstream)              |

## Routing

| Source                                    | SRX                                          |
| ----------------------------------------- | -------------------------------------------- |
| PAN-OS virtual router / ASA routing table | `routing-instances` (virtual-router) or inet.0 |
| PBF / ASA PBR                             | Filter-based forwarding (firewall filter + routing instance) |
| OSPF/BGP                                  | `protocols ospf/bgp` + `policy-options` for redistribution |

Remember to allow routing protocols in `host-inbound-traffic protocols` on the zone/interface.

## HA

- PAN-OS HA pair / ASA failover → SRX **chassis cluster** (control link, fabric link, redundancy groups, reth interfaces)
- Interface numbering on node 1 is offset by platform — map physical ports before writing config
- Plan the cluster build as its own change, before policy migration

## Verification

```
show security match-policies from-zone untrust to-zone dmz source-ip 198.51.100.20 destination-ip 10.20.0.10 destination-port 443 protocol tcp
show security nat destination rule all
show security nat source rule all
show security flow session destination-prefix 10.20.0.10
show security policies hit-count
show security ike security-associations
show security ipsec security-associations
show log messages | match RT_FLOW_SESSION_DENY
```

Note: `match-policies` takes the **post-NAT** destination.

## Gotchas Checklist

- [ ] PAN-OS public destinations rewritten to real addresses in every DNAT-related policy
- [ ] Proxy ARP entries for every NAT address in an interface subnet
- [ ] `host-inbound-traffic` allows IKE, routing protocols, management where needed
- [ ] Global address book used; no per-zone duplicates
- [ ] Explicit global deny with logging
- [ ] Traffic selectors match policy-based peers
- [ ] `commit confirmed` used for every cutover commit
