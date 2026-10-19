# Cutover & Rollback Playbook

Applies to any firewall replacement or major policy change. Fill in the bracketed fields per change.

## 1. Preparation (T-14 to T-1 days)

- [ ] Change approved; window, approver and rollback decision-maker named
- [ ] Config backups of **every** device touched (old firewall, new firewall, switches, routers, VPN peers) stored off-box
- [ ] Target config staged and reviewed; diff archived
- [ ] Critical-flow test list agreed with app owners (who tests what, how, expected result)
- [ ] VPN peers: third parties informed, new peer IP / crypto settings agreed, contact on call
- [ ] DNS TTLs lowered (if any public IPs change) at least 2× the old TTL beforehand
- [ ] Upstream ARP behaviour understood (see section 3)
- [ ] Out-of-band access to all devices verified (console server, mgmt network)
- [ ] Licences, content updates and HA sync verified on the target
- [ ] Monitoring (syslog, SNMP, SIEM parser) pointed at the target
- [ ] Change freeze on source rulebase from T-3 days (or delta-sync procedure agreed)

## 2. Go / No-Go (T-0, start of window)

| Check                                        | Owner      | Status |
| -------------------------------------------- | ---------- | ------ |
| Target config matches approved version        | Firewall   |        |
| Rollback steps rehearsed / documented         | Firewall   |        |
| App testers available                         | App owners |        |
| Third-party VPN contacts reachable            | Network    |        |
| No open P1/P2 incidents on affected services  | Ops        |        |

Any "no" → postpone.

## 3. Execution Patterns

### Physical port swing (same IPs)

1. Pre-cable the target where possible; label every cable with old and new port
2. Shut source interfaces (or disconnect) → bring up target interfaces
3. **ARP:** neighbours cache the old firewall MAC. Either
   - Clear ARP on upstream/downstream routers and L3 switches, or
   - Rely on gratuitous ARP from the target (most platforms send GARP when interfaces come up / on HA failover), or
   - Temporarily reduce ARP timeouts beforehand
4. Dynamic routing: confirm adjacencies (OSPF/BGP) re-form; BGP peers may need a soft reset
5. Run critical-flow tests

### Routing swing (new IPs, parallel run)

1. Target live in parallel on new transit IPs
2. Move traffic per segment by changing next-hop / route preference
3. Rollback = revert the route change (fast, low risk)

### VPN peer moves

1. One peer at a time; peer updates their gateway IP if it changes
2. Confirm phase 1 and phase 2 (all proxy-ID / selector pairs) are up
3. Test traffic both directions

## 4. Session Impact

- Existing sessions do **not** transfer between different vendors' firewalls. Long-lived TCP (databases, mainframe, VPN-carried apps, SSH) will reset — warn app owners and schedule app restarts if needed.
- Stateful asymmetry: if only one direction moves first, the new firewall drops mid-session packets. Move both directions of a segment together.

## 5. Validation During the Window

Run `validation-checklist.md` section "Live". Minimum:

- [ ] All critical flows tested and signed off by their owners
- [ ] Deny logs reviewed for unexpected drops on production subnets
- [ ] NAT translations visible for published services
- [ ] All VPN tunnels up with traffic counters increasing
- [ ] Routing table matches the pre-change snapshot (prefix count ± documented changes)
- [ ] Management, logging and monitoring working on the target

## 6. Rollback

### Triggers (decide before the window)

- Any critical flow failing with no fix within **[30] minutes**
- More than **[N]** unexplained denies per minute on production traffic
- Loss of management access to the target
- Window end minus **[60] minutes** reached with open failures

### Steps

1. Announce rollback on the bridge; stop further changes
2. Reverse the execution steps (port swing back / revert routes / re-point VPN peers)
3. Clear ARP on neighbours (or wait for GARP)
4. Verify critical flows on the **source** firewall
5. Record what failed, logs and captures, for the post-mortem

### Platform safety nets for config changes

| Platform     | Mechanism                                                            |
| ------------ | -------------------------------------------------------------------- |
| Junos        | `commit confirmed <minutes>` — auto-rollback unless confirmed with `commit` |
| FortiOS      | `config system global` → `set cfg-save revert` + `set cfg-revert-timeout <s>`; confirm with `execute cfg save` |
| PAN-OS       | Named config snapshot before change; `load config version <n>` or `load config from <file>` then commit; `revert config` discards uncommitted candidate |
| Cisco IOS-XE | `configure replace flash:<file> time <s>` (confirm with `configure confirm`) or `reload in <min>` |
| NX-OS        | `checkpoint <name>` / `rollback running-config checkpoint <name>`     |
| Cisco ASA    | Changes are live immediately; keep previous config on flash, `reload in <min>` only with care (drops all traffic) |
| FMC / FTD    | Policy deployment history → roll back to previous deployment          |
| F5 BIG-IP    | UCS archive before change; `load sys config` from saved config or restore UCS |

## 7. Close-Out

- [ ] Hypercare owner and duration named (e.g. 5 business days)
- [ ] Source device kept intact and cable-ready for the rollback period
- [ ] Temporary rules listed with expiry dates
- [ ] Documentation updated (diagrams, CMDB, runbooks)
- [ ] Post-implementation review scheduled
