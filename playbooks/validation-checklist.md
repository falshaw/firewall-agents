# Migration Validation Checklist

Use in three passes: **Static** (config review before the window), **Simulated** (on the staged target), **Live** (during and after cutover).

## Baseline (capture from source before conversion)

| Metric                         | Source value | Target value | Delta explained? |
| ------------------------------ | ------------ | ------------ | ---------------- |
| Address objects                |              |              |                  |
| Address groups                 |              |              |                  |
| Service objects / groups       |              |              |                  |
| Security rules (enabled)       |              |              |                  |
| Security rules (disabled)      |              |              |                  |
| NAT rules                      |              |              |                  |
| Site-to-site VPN peers         |              |              |                  |
| Phase 2 SAs / proxy-ID pairs   |              |              |                  |
| Routes (static / dynamic)      |              |              |                  |
| Interfaces / sub-interfaces    |              |              |                  |
| Peak concurrent sessions       |              |              |                  |
| Peak throughput / CPS          |              |              |                  |

Deltas are expected after cleanup — each one must be listed in the mapping file.

## Pass 1 — Static Review

### Objects

- [ ] No duplicate objects with different names for the same value
- [ ] No truncated or renamed objects without a mapping entry
- [ ] FQDN objects resolve from the target's DNS servers
- [ ] Ranges and wildcard masks converted correctly (wildcard masks are a common converter bug)
- [ ] Services: `tcp-udp`, port ranges, ICMP types/codes converted correctly

### Rules

- [ ] Every source rule maps to ≥ 1 target rule in the mapping file (or is listed as removed with reason)
- [ ] No new `any any` allow rules introduced by the converter
- [ ] Rule order preserved where it matters (deny-before-allow pairs)
- [ ] Shadow analysis run on the target rulebase
- [ ] Every allow rule has logging + security profiles per the design
- [ ] Default/implicit deny logs

### NAT

- [ ] NAT rule order matches source semantics
- [ ] Security rules for DNAT use the correct address form for the target platform (pre-NAT vs post-NAT — see `agents/firewall-architect.md`)
- [ ] No-NAT / exemption rules placed above broader source NAT
- [ ] Proxy ARP / ARP reply configured for every public NAT address on the target
- [ ] Hairpin / U-turn NAT for internal clients using public names

### VPN

- [ ] Crypto parameters (IKE version, encryption, integrity, DH group, lifetimes, PFS) match each peer
- [ ] Proxy IDs / traffic selectors exactly match policy-based peers
- [ ] Routes into tunnels present; security rules permit tunnel traffic both directions
- [ ] DPD / keepalive settings compatible

### Platform

- [ ] Management access restricted (source subnets, protocols)
- [ ] AAA, NTP, syslog, SNMPv3 configured and reachable
- [ ] HA configured, synced, failover tested in the staging phase
- [ ] Content/threat updates current; licences active

## Pass 2 — Simulated

Run a policy-match and NAT-match test for every critical flow on the staged target.

| Vendor   | Policy test                                                   | NAT test                                  |
| -------- | ------------------------------------------------------------- | ----------------------------------------- |
| PAN-OS   | `test security-policy-match from <z> to <z> source <ip> destination <ip> protocol 6 destination-port <p>` | `test nat-policy-match ...`               |
| ASA/FTD  | `packet-tracer input <if> tcp <src> <sport> <dst> <dport> detailed` | (included in packet-tracer)          |
| FortiOS  | `diagnose firewall iprope lookup <src> <sport> <dst> <dport> tcp <srcintf>` | `diagnose firewall vip list`     |
| Junos    | `show security match-policies from-zone <z> to-zone <z> source-ip <ip> destination-ip <ip> destination-port <p> protocol tcp` | `show security nat ... rule all` |

Flow test table:

| # | Flow (app / owner)    | Source         | Destination      | Port   | Expected | Target rule hit | Result |
| - | --------------------- | -------------- | ---------------- | ------ | -------- | --------------- | ------ |
| 1 | Web portal / AppTeam-A | any           | 203.0.113.10     | 443    | allow    |                 |        |
| 2 | DB replication / DBA  | 10.20.1.0/24   | 10.30.1.0/24     | 1433   | allow    |                 |        |
| 3 | Guest to internal     | 192.168.100.0/24 | 10.0.0.0/8     | any    | deny     |                 |        |

Include negative tests (flows that must stay blocked), especially across compliance boundaries (e.g. CDE).

## Pass 3 — Live

### During the window

- [ ] Interfaces up, no errors/drops incrementing
- [ ] ARP entries on neighbours point to the target MAC
- [ ] Routing adjacencies up; prefix counts match baseline
- [ ] All VPN tunnels up; traffic counters increasing in both directions
- [ ] Every critical flow tested by its owner and signed off
- [ ] Published services reachable from an external test point
- [ ] Deny log reviewed: no unexpected production drops
- [ ] Logs arriving in SIEM; parsers working

### Hypercare (24–72 h)

- [ ] Deny log review at agreed intervals; fixes via emergency change
- [ ] Session count and CPU/memory compared to baseline
- [ ] Threat/IPS false positives reviewed before tightening profiles
- [ ] Helpdesk tickets correlated with firewall logs

### After 30 / 90 days

- [ ] Temporary rules removed
- [ ] Zero-hit rules on the target reviewed and removed
- [ ] Optimisation backlog started (App-ID, identity, decryption)
- [ ] Source device decommissioned and wiped
