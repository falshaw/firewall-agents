# Juniper SRX Quick Reference

**Scope:** Junos / SRX at JNCIA-Junos to JNCIS-SEC level

---

## JunOS CLI Essentials

### Modes

| Mode          | Prompt      | Purpose                             |
| ------------- | ----------- | ----------------------------------- |
| Operational   | `user@srx>` | Show, monitor, troubleshoot         |
| Configuration | `user@srx#` | Edit config                         |
| Shell         | `%`         | FreeBSD shell (avoid in production) |

### Navigation

```
configure                    # Enter config mode
configure exclusive          # Lock config (single editor)
configure private            # Private candidate config
edit security policies       # Navigate to policies
up / top                     # Navigate up / to root
exit                         # Leave current level or mode
```

### Show Commands

```
show | compare               # Diff candidate vs active
show | display set           # Set-command format
show | display xml           # XML format (for NETCONF)
show | match "pattern"       # Filter output (like grep)
show | except "pattern"      # Exclude matching lines
show | count                 # Count output lines
show | no-more               # Disable paging
```

### Commit Options

```
commit check                 # Validate only
commit confirmed 5           # Auto-rollback in 5 min
commit comment "<change-ref>" # Attach comment
commit synchronize           # Sync dual-RE
commit at "2026-02-01 02:00" # Scheduled commit
rollback ?                   # Show available rollbacks (0-49)
rollback 1                   # Revert to previous
```

---

## Interface Configuration

### Physical

```
set interfaces xe-0/0/0 unit 0 family inet address 10.0.0.1/24
set interfaces xe-0/0/0 description "UPLINK-ISP"
set interfaces xe-0/0/0 unit 0 family inet filter input PROTECT-RE
```

### VLAN Tagging (Sub-interfaces)

```
set interfaces xe-0/0/1 vlan-tagging
set interfaces xe-0/0/1 unit 100 vlan-id 100
set interfaces xe-0/0/1 unit 100 family inet address 10.1.0.1/24
set interfaces xe-0/0/1 unit 200 vlan-id 200
set interfaces xe-0/0/1 unit 200 family inet address 10.2.0.1/24
```

### Loopback

```
set interfaces lo0 unit 0 family inet address 10.255.0.1/32
```

### Aggregate (LAG)

```
set chassis aggregated-devices ethernet device-count 2
set interfaces ae0 aggregated-ether-options lacp active
set interfaces xe-0/0/0 ether-options 802.3ad ae0
set interfaces xe-0/0/1 ether-options 802.3ad ae0
set interfaces ae0 unit 0 family inet address 10.0.0.1/24
```

---

## Security Zones

```
# Create zones
set security zones security-zone trust
set security zones security-zone untrust
set security zones security-zone dmz

# Assign interfaces
set security zones security-zone trust interfaces xe-0/0/1.100
set security zones security-zone untrust interfaces xe-0/0/0.0
set security zones security-zone dmz interfaces xe-0/0/2.0

# Host-inbound traffic (management access TO the SRX)
set security zones security-zone trust interfaces xe-0/0/1.100 host-inbound-traffic system-services ssh
set security zones security-zone trust interfaces xe-0/0/1.100 host-inbound-traffic system-services ping
set security zones security-zone trust interfaces xe-0/0/1.100 host-inbound-traffic system-services https
set security zones security-zone trust interfaces xe-0/0/1.100 host-inbound-traffic protocols ospf
```

---

## Security Policies

### Policy Structure

```
from-zone <src> to-zone <dst> policy <name>
  match: source-address, destination-address, application
  then: permit|deny|reject + log + count
```

### Examples

```
# Allow web traffic outbound
set security policies from-zone trust to-zone untrust policy allow-web match source-address any
set security policies from-zone trust to-zone untrust policy allow-web match destination-address any
set security policies from-zone trust to-zone untrust policy allow-web match application [junos-http junos-https junos-dns-udp]
set security policies from-zone trust to-zone untrust policy allow-web then permit
set security policies from-zone trust to-zone untrust policy allow-web then log session-close

# Deny all (default - implicit, but explicit is clearer)
set security policies from-zone trust to-zone untrust policy deny-rest match source-address any
set security policies from-zone trust to-zone untrust policy deny-rest match destination-address any
set security policies from-zone trust to-zone untrust policy deny-rest match application any
set security policies from-zone trust to-zone untrust policy deny-rest then deny
set security policies from-zone trust to-zone untrust policy deny-rest then log session-init
```

### Address Book (per-zone in SRX)

```
set security zones security-zone trust address-book address web-srv 10.1.1.10/32
set security zones security-zone trust address-book address app-srv 10.1.1.20/32
set security zones security-zone trust address-book address-set servers address web-srv
set security zones security-zone trust address-book address-set servers address app-srv

# Global address book (shared across zones)
set security address-book global address partner-net 172.16.0.0/16
```

---

## Routing

### Static Routes

```
set routing-options static route 0.0.0.0/0 next-hop 203.0.113.1
set routing-options static route 192.168.0.0/16 next-hop 10.0.0.2
set routing-options static route 10.10.0.0/24 next-hop st0.0   # via VPN tunnel
```

### OSPF

```
set protocols ospf area 0.0.0.0 interface xe-0/0/1.100
set protocols ospf area 0.0.0.0 interface lo0.0 passive
set protocols ospf area 0.0.0.1 interface xe-0/0/2.0
set protocols ospf area 0.0.0.1 stub    # Stub area

# Verification
show ospf neighbor
show ospf database
show ospf interface
show route protocol ospf
```

### BGP

```
set routing-options autonomous-system 65001
set protocols bgp group ebgp-isp type external
set protocols bgp group ebgp-isp peer-as 65000
set protocols bgp group ebgp-isp neighbor 203.0.113.1
set protocols bgp group ebgp-isp export send-default
set protocols bgp group ebgp-isp import accept-all

# Route Policy
set policy-options policy-statement send-default term 1 from route-filter 0.0.0.0/0 exact
set policy-options policy-statement send-default term 1 then accept

# Verification
show bgp summary
show bgp neighbor <ip>
show route protocol bgp
show route advertising-protocol bgp <peer>
show route receive-protocol bgp <peer>
```

### Routing Instances (VRF)

```
set routing-instances MGMT instance-type virtual-router
set routing-instances MGMT interface xe-0/0/3.0
set routing-instances MGMT routing-options static route 0.0.0.0/0 next-hop 10.100.0.1

# Show routes in VRF
show route table MGMT.inet.0
```

---

## NAT

### Source NAT (PAT)

```
set security nat source rule-set trust-to-untrust from zone trust
set security nat source rule-set trust-to-untrust to zone untrust
set security nat source rule-set trust-to-untrust rule internet-pat match source-address 10.0.0.0/8
set security nat source rule-set trust-to-untrust rule internet-pat then source-nat interface
```

### Destination NAT

```
set security nat destination pool web-pool address 10.1.1.10/32 port 443
set security nat destination rule-set untrust-dnat from zone untrust
set security nat destination rule-set untrust-dnat rule web-inbound match destination-address 203.0.113.10/32
set security nat destination rule-set untrust-dnat rule web-inbound match destination-port 443
set security nat destination rule-set untrust-dnat rule web-inbound then destination-nat pool web-pool
```

### Static NAT (1:1)

```
set security nat static rule-set static-nat from zone untrust
set security nat static rule-set static-nat rule server-nat match destination-address 203.0.113.20/32
set security nat static rule-set static-nat rule server-nat then static-nat prefix 10.1.1.20/32
```

### Verification

```
show security nat source summary
show security nat destination summary
show security nat static rule all
show security flow session nat
```

---

## IPSec VPN

### Quick Setup Summary

1. IKE Proposal (crypto params)
2. IKE Policy (proposal + PSK/cert)
3. IKE Gateway (peer address + policy)
4. IPSec Proposal (ESP params)
5. IPSec Policy (proposal + PFS)
6. IPSec VPN (gateway + policy + bind-interface)
7. st0 interface
8. Route to remote via st0
9. Security policy to allow traffic

### Verification

```
show security ike security-associations
show security ike security-associations detail
show security ipsec security-associations
show security ipsec statistics
show security ipsec next-hop-tunnels
ping <remote-ip> source <local-tunnel-ip>
```

---

## Chassis Cluster (HA)

### Quick Reference

```
# Status
show chassis cluster status
show chassis cluster interfaces
show chassis cluster statistics

# Failover
request chassis cluster failover redundancy-group 1 node 1
request chassis cluster failover reset redundancy-group 1

# Key concepts
- Redundancy Group 0 = RE (control plane)
- Redundancy Group 1+ = data plane
- reth = redundant ethernet interface
- fab = fabric link between nodes
```

---

## Troubleshooting Cheat Sheet

| Symptom             | Command                                                                                             | Look For                 |
| ------------------- | --------------------------------------------------------------------------------------------------- | ------------------------ |
| Traffic dropped     | `show security flow session`                                                                        | Missing sessions         |
| Policy not matching | `show security match-policies from-zone X to-zone Y source-ip A dest-ip B dest-port P protocol tcp` | Which policy hits        |
| Interface down      | `show interfaces terse`                                                                             | Status/link              |
| Routing issue       | `show route <ip>`                                                                                   | Next-hop, protocol       |
| VPN down            | `show security ike sa`                                                                              | State, proposal mismatch |
| HA failover         | `show chassis cluster status`                                                                       | Priority, state          |
| High CPU            | `show chassis routing-engine`                                                                       | CPU %, memory            |
| Drops/counters      | `show security flow statistics`                                                                     | Drop reasons             |

### Flow Debugging (use carefully in production)

```
set security flow traceoptions file flow-debug
set security flow traceoptions flag basic-datapath
set security flow traceoptions packet-filter match-1 source-address 10.0.0.5
set security flow traceoptions packet-filter match-1 destination-address 198.51.100.53
# View: show log flow-debug | last 100
# REMOVE after troubleshooting: delete security flow traceoptions
```

---

**See also:** `../../agents/juniper-srx-expert.md` (full config examples), `../networking/` (protocol theory), `../../playbooks/to-juniper-srx.md` (migration)

_Quick reference for Juniper SRX deployments. JNCIA/JNCIS level._
