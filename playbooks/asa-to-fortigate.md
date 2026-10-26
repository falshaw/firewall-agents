# Playbook: Cisco ASA → Fortinet FortiGate

Read `00-methodology.md` first. Agents: `cisco-security-expert` (source), `fortinet-expert` (target).

## Decide First

| Decision        | Options                                                   |
| --------------- | --------------------------------------------------------- |
| NAT mode        | **Policy-based NAT** (default): VIPs as policy destination, SNAT per policy. **Central NAT**: SNAT table + DNAT/VIP table, policies use real addresses — closer to ASA semantics |
| Interface model | Interfaces direct in policies, or **zones** grouping interfaces (simpler policy count) |
| Inspection mode | Flow-based (default) vs proxy-based per policy            |

Tooling: FortiConverter handles objects, policies, NAT and VPN for ASA sources; review its output against this playbook.

## Object Mapping

| ASA                          | FortiOS                                    |
| ---------------------------- | ------------------------------------------ |
| `object network` host/subnet/range/fqdn | `firewall address` (ipmask/iprange/fqdn) |
| `object-group network`       | `firewall addrgrp`                         |
| `object service`, `object-group service` | `firewall service custom`, `firewall service group` |
| `tcp-udp` service            | One custom service with both `tcp-portrange` and `udp-portrange` |
| ICMP types                   | Custom service protocol ICMP with type/code |
| `time-range`                 | `firewall schedule recurring/onetime`      |
| `remark`                     | Policy `comments`                          |

## Policy Mapping

1. Each inbound interface ACL → policies with `srcintf` = that interface (or zone), `dstintf` = interface(s) toward the destination by routing.
2. A FortiGate policy can list multiple `srcintf`/`dstintf` values; avoid `any` interface where possible (it complicates troubleshooting and some features).
3. Security-level implicit permits → explicit accept policies.
4. ASA implicit deny → FortiGate implicit deny (policy ID 0). Enable logging:

```
config log setting
    set fwpolicy-implicit-log enable
end
```

5. Policy order: FortiGate evaluates top-down by **sequence**, not by policy ID. Use `move <id> before <id>` to order.

## NAT Mapping (policy-based NAT mode)

| ASA                                         | FortiOS                                                        |
| ------------------------------------------- | -------------------------------------------------------------- |
| Static NAT / port forward for a server      | `firewall vip` (extip → mappedip, optional portforward); inbound policy `dstaddr` = **VIP object** |
| Dynamic PAT to interface                    | Policy `set nat enable` (uses egress interface IP)             |
| Dynamic NAT/PAT to pool                     | `firewall ippool` (overload / one-to-one / fixed-port-range) + policy `set ippool enable` `set poolname` |
| Static 1:1 outbound for the same server     | VIP with `set nat-source-vip enable`, or explicit ippool one-to-one |
| NAT exemption (VPN)                         | VPN policy with `set nat disable` — nothing else needed        |
| Twice NAT (source + destination together)   | VIP + ippool on the same policy                                |

**Address semantics:** in policy-based mode the inbound policy references the **VIP**, not the real server address; ASA ACLs reference the real IP. In central NAT mode the policy references the real (mapped) address — the same as ASA.

**Proxy ARP:** FortiGate answers ARP for VIP external IPs and ippool addresses automatically (`arp-reply enable`, default).

**Hairpin (internal clients to public IP):** set the VIP `extintf` to `any`, then add one policy from the client interface to the server interface with the VIP as destination. With the VIP bound to `wan1`, the traffic needs a policy from internal to `wan1` plus a policy from `wan1` to the server interface with the VIP as destination. ASA needed a same-security/hairpin config for this too.

### Example

```
config firewall vip
    edit "VIP-WEB-01-HTTPS"
        set extip 203.0.113.10
        set extintf "wan1"
        set mappedip "10.20.0.10"
        set portforward enable
        set extport 443
        set mappedport 443
    next
end
config firewall policy
    edit 0
        set name "ALLOW-WEB-01-IN"
        set srcintf "wan1"
        set dstintf "dmz"
        set srcaddr "all"
        set dstaddr "VIP-WEB-01-HTTPS"
        set action accept
        set schedule "always"
        set service "HTTPS"
        set logtraffic all
        set utm-status enable
        set ips-sensor "default"
    next
end
```

## VPN Mapping

| ASA                                   | FortiOS                                                     |
| ------------------------------------- | ----------------------------------------------------------- |
| IKE policy + tunnel-group PSK         | `vpn ipsec phase1-interface` (route-based)                  |
| Transform set / IPsec proposal + PFS  | `vpn ipsec phase2-interface`                                |
| Crypto ACL lines                      | Phase 2 selectors (`src-subnet`/`dst-subnet`) — one phase2 per ACL line when the peer is policy-based; `0.0.0.0/0` selectors only if the peer agrees |
| Routes via crypto map                 | Static routes to the tunnel interface + accept policies to/from the tunnel |
| AnyConnect                            | FortiClient SSL-VPN or IPsec dial-up (check current FortiOS SSL-VPN support status for the target version), or ZTNA |

## Inspection

- ASA `inspect` → FortiGate session helpers (`config system session-helper`) and ALGs. Review SIP: FortiGate's SIP ALG/helper is often disabled when the PBX handles NAT.
- Add security profiles after cutover stability (AV, IPS, web filter, application control).

## Verification

| Purpose  | ASA                                          | FortiOS                                                              |
| -------- | -------------------------------------------- | -------------------------------------------------------------------- |
| Policy   | `packet-tracer ...`                          | `diagnose firewall iprope lookup <src> <sport> <dst> <dport> <proto> <srcintf>` |
| Flow     | `capture ... type asp-drop`                  | `diagnose debug flow` (filter, trace start, enable)                  |
| NAT      | `show xlate`                                 | `show firewall vip`, `diagnose sys session filter ...` + `diagnose sys session list` (shows NAT) |
| VPN      | `show crypto ipsec sa`                       | `diagnose vpn tunnel list`, `get vpn ipsec tunnel summary`           |

## Gotchas Checklist

- [ ] NAT mode chosen and documented before conversion
- [ ] Inbound policies use VIP objects (policy-based mode)
- [ ] Policy sequence verified (not ID order)
- [ ] Implicit-deny logging enabled
- [ ] Phase 2 selectors match peers' crypto ACLs exactly
- [ ] Hairpin policies added where internal clients use public names
- [ ] Session helpers reviewed (SIP, FTP, DNS)
- [ ] Interface `allowaccess` restricted (no admin on WAN)
