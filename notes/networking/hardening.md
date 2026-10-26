# Network Hardening & Zero Trust

Each config block names its OS. Nexus (NX-OS) and Catalyst (IOS-XE) syntax differ; do not paste one into the other.

## AAA (Authentication, Authorization, Accounting)

### TACACS+ (Preferred for Network Devices)

```
! NX-OS (Nexus 9000)
! TACACS+ server configuration
feature tacacs+
tacacs-server host 10.0.0.50 key 7 ENCRYPTED
tacacs-server host 10.0.0.51 key 7 ENCRYPTED

aaa group server tacacs+ CORP_TACACS
  server 10.0.0.50
  server 10.0.0.51

! AAA configuration
aaa authentication login default group CORP_TACACS     ! local fallback is automatic if servers are unreachable
aaa authorization commands default group CORP_TACACS local
aaa accounting default group CORP_TACACS

! Local fallback user (if TACACS down) — type 5 = SHA-256 hash (NX-OS has no type 7 user passwords)
username admin-local password 5 HASHED_PASSWORD role network-admin
```

**TACACS+ vs RADIUS:**
| Feature | TACACS+ | RADIUS |
|---------|---------|--------|
| Protocol | TCP 49 | UDP 1812/1813 |
| Encryption | Full packet | Password only |
| Authorization | Per-command | Per-session |
| Best for | **Network devices** | 802.1X/WiFi |
| Vendor | Cisco ISE / ACS | ISE / FreeRADIUS |

### SNMPv3 (Only Acceptable Version)

```
! NX-OS (Nexus 9000)
! SNMPv3 with auth + privacy (authPriv). NX-OS uses roles, not IOS-style snmp-server groups:
! network-operator = read-only
snmp-server user MONITOR_USER network-operator auth sha AUTH_KEY priv aes-128 PRIV_KEY
snmp-server globalEnforcePriv

! Send traps to the NMS only, via the management VRF
snmp-server host 10.0.0.100 traps version 3 priv MONITOR_USER
snmp-server host 10.0.0.100 use-vrf management

! Disable v1/v2c completely
no snmp-server community public
no snmp-server community private
```

## 802.1X / Network Access Control

### ISE Integration

```
! IOS-XE (Catalyst 9000 access switch), classic Auth Manager syntax
aaa new-model
radius server ISE-01
 address ipv4 10.0.0.60 auth-port 1812 acct-port 1813
 key RADIUS_KEY
aaa authentication dot1x default group radius
aaa authorization network default group radius
dot1x system-auth-control
!
interface GigabitEthernet1/0/30
 switchport mode access
 switchport access vlan 100
 ! 802.1X first, MAB fallback for devices without a supplicant
 authentication host-mode multi-auth
 authentication order dot1x mab
 authentication priority dot1x mab
 authentication port-control auto
 mab
 dot1x pae authenticator
 dot1x timeout tx-period 10
 dot1x max-reauth-req 2
```

Unknown or failed devices: let ISE authorise the MAB result (VLAN or dACL). The local restricted VLAN (`authentication event fail action authorize vlan`) supports single-host mode only, so it does not fit a multi-auth port.

### TrustSec / SGT (Security Group Tags)

```
! IOS-XE (Catalyst 9000)
! SGT assignment via ISE
! ISE assigns SGT based on user/device identity after 802.1X
!
! Example SGT mapping:
! SGT 10 = Payment_Terminals (CDE)
! SGT 20 = SWIFT_Operators
! SGT 30 = Corp_Users
! SGT 40 = Admins
! SGT 50 = Guests

! SGACL (Security Group ACL) — enforced at egress
! Defined in ISE, pushed to switches:
! permit Payment_Terminals -> Payment_Server (TCP 443, 8443)
! deny Payment_Terminals -> Corp_Users (all)
! deny Guests -> everything except Internet

! CTS (Cisco TrustSec) on switch
cts authorization list CTS_LIST
cts role-based enforcement
cts role-based sgt-map 10.1.0.0/16 sgt 10    ! Static SGT for CDE subnet
```

**SGT + ACI mapping:**

- SGT tags map conceptually to ACI EPGs
- Both provide micro-segmentation
- ACI contracts = SGACL policies
- In hybrid environments, ISE can feed SGT info to APIC

## Control Plane Protection

### CoPP (Control Plane Policing)

```
! NX-OS (Nexus 9000)
! Protect switch CPU from DoS
! NX-OS has default CoPP policy — verify and customize

show copp status
show policy-map interface control-plane

! Custom CoPP class for BGP (critical — police generously)
class-map type control-plane match-any COPP_BGP
  match access-group name COPP_BGP_ACL

policy-map type control-plane COPP_POLICY
  class COPP_BGP
    police cir 50000 kbps bc 5000000 bytes conform transmit violate drop
  class COPP_DEFAULT
    police cir 500 kbps bc 50000 bytes conform transmit violate drop

! Apply
control-plane
  service-policy input COPP_POLICY
```

### Management Plane Hardening

```
! NX-OS (Nexus 9000)
! SSH only (telnet is off by default; make sure it stays off)
no feature telnet
feature ssh
ssh key rsa 2048                    ! NX-OS 10.x range is 768-2048 bits

! Restrict management access
ip access-list MGMT_ACCESS
  permit ip 10.0.0.0/24 any        ! Management subnet only
  deny ip any any log

line vty
  ip access-class MGMT_ACCESS in
  exec-timeout 15                   ! minutes
  session-limit 5

! Console security
line console
  exec-timeout 5

! Banner
banner motd #
*** AUTHORIZED ACCESS ONLY ***
This system is monitored. Unauthorized access is prohibited.
#

! Disable unused services
no feature nxapi              ! NX-API (HTTPS 443) is on by default since 9.2(1); off if unused

interface Ethernet1/48        ! untrusted port
  no cdp enable
  no lldp transmit
  no lldp receive
```

### Logging & Monitoring

```
! NX-OS (Nexus 9000)
! Syslog to SIEM
logging server 10.0.0.70 5 use-vrf management    ! Severity 5 = notification
logging source-interface loopback0
logging timestamp milliseconds
logging level authpriv 6                          ! Detailed auth logs
logging origin-id hostname

! NTP (critical for log correlation + compliance)
ntp server 10.0.0.80 use-vrf management
ntp authentication-key 1 md5 NTP_SECRET
ntp trusted-key 1
ntp authenticate

! NetFlow/IPFIX (traffic visibility)
feature netflow
flow record CORP_FLOW
  match ipv4 source address
  match ipv4 destination address
  match transport source-port
  match transport destination-port
  collect counter bytes
  collect counter packets
  collect timestamp sys-uptime first
  collect timestamp sys-uptime last

flow exporter CORP_EXPORT
  destination 10.0.0.70
  transport udp 2055
  version 9

flow monitor CORP_MONITOR
  record CORP_FLOW
  exporter CORP_EXPORT

interface Ethernet1/1
  ip flow monitor CORP_MONITOR input
```

## Zero Trust Mapping to ACI

| Zero Trust Principle       | ACI Implementation                                   |
| -------------------------- | ---------------------------------------------------- |
| Never trust, always verify | Default deny (no contract = no communication)        |
| Least privilege            | Contracts with specific filters (port/protocol)      |
| Micro-segmentation         | EPGs (endpoints grouped by function, not subnet)     |
| Continuous verification    | ISE + ACI integration, dynamic EPG membership        |
| Assume breach              | Per-EPG policy, lateral movement blocked by default  |
| Identity-based access      | ESGs (Endpoint Security Groups) in ACI 5.x+          |
| Visibility                 | APIC health scores, traffic analytics, span sessions |

## ACI + Firewall Zero Trust Architecture

```
                    ┌─────────────────────────────────────┐
                    │           APIC Cluster               │
                    │    (Policy + Visibility + Health)     │
                    └─────────────┬───────────────────────┘
                                  │
                    ┌─────────────┴───────────────────────┐
                    │         Spine Switches               │
                    │    (IS-IS Underlay + COOP)           │
                    └──┬──────────┬──────────────┬────────┘
                       │          │              │
              ┌────────┴──┐ ┌────┴─────┐ ┌──────┴────────┐
              │  Leaf-1   │ │  Leaf-2  │ │  Leaf-3       │
              │  (CDE)    │ │  (SWIFT) │ │  (Corp+DMZ)   │
              └─────┬─────┘ └────┬─────┘ └──────┬────────┘
                    │            │               │
              ┌─────┴─────┐     │         ┌─────┴─────┐
              │ NGFW pair │     │         │ NGFW pair │
              │ (E-W FW)  │     │         │ (N-S FW)  │
              └───────────┘     │         └───────────┘
                                │
                    Service Graph + PBR redirects
                    inter-EPG traffic through firewall

Zero Trust flow:
1. Endpoint connects → 802.1X/ISE authenticates
2. ISE assigns SGT → maps to ACI EPG via APIC-ISE integration
3. Endpoint placed in correct EPG (micro-segment)
4. Any cross-EPG traffic → contract required
5. Sensitive contracts → service graph → firewall inspection
6. Firewall applies L7 policy (IPS, URL filtering, threat prevention)
7. APIC monitors health, logs flows, alerts on anomalies
```

## Hardening Checklist for Regulated Networks

**Physical:**

- [ ] Unused ports shutdown + assigned to black-hole VLAN
- [ ] Console port password protected + timeout
- [ ] BIOS/firmware passwords set on all network devices

**Layer 2:**

- [ ] Native VLAN changed from 1 to unused VLAN (e.g., 999)
- [ ] DTP disabled on all ports (`switchport nonegotiate`)
- [ ] BPDU guard on all access ports
- [ ] Root guard on all non-root ports
- [ ] DHCP snooping enabled on all VLANs
- [ ] DAI enabled on all VLANs
- [ ] IP Source Guard on access ports
- [ ] Storm control configured
- [ ] Port security with sticky MAC
- [ ] Private VLANs for DMZ segments

**Layer 3:**

- [ ] OSPF/BGP authentication on ALL links (MD5 minimum, SHA preferred)
- [ ] Prefix lists on all BGP peers
- [ ] Route filtering — no default route redistribution
- [ ] URPF (Unicast Reverse Path Forwarding) on edge interfaces
- [ ] ACLs on all L3 interfaces (explicit deny + log)

**Management:**

- [ ] SSH only (RSA 4096 or Ed25519)
- [ ] TACACS+ for all admin access
- [ ] SNMPv3 authPriv only
- [ ] NTP authenticated
- [ ] Syslog to SIEM (TLS if possible)
- [ ] CoPP configured
- [ ] HTTP/HTTPS server disabled
- [ ] CDP/LLDP disabled on untrusted ports
- [ ] Banners on all VTY and console lines
- [ ] Session timeouts configured (15 min VTY, 5 min console)

**ACI Specific:**

- [ ] APIC out-of-band management only
- [ ] RBAC configured (no admin accounts shared)
- [ ] Contract logging enabled for audit
- [ ] Endpoint learning disabled on unused ports
- [ ] Rogue endpoint detection enabled
- [ ] Fabric access policies locked down (AAEP, interface profiles)
- [ ] APIC cluster: 3 nodes minimum, no single point of failure

---
