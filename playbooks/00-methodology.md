# Firewall Migration Methodology

Vendor-neutral process for moving a rulebase from one firewall platform to another. The vendor-pair playbooks in this folder add the platform-specific mappings and gotchas.

## Principles

1. **Translate intent, not syntax.** A rule exists to allow a business flow. Converters copy syntax, including mistakes; humans must confirm intent.
2. **Clean before you convert.** Every unused rule you migrate becomes permanent technical debt on the new platform.
3. **Like-for-like first, optimise second.** Cutover with equivalent policy; tighten (App-ID, profiles, identity) after the platform is stable. Doing both at once makes failures impossible to attribute.
4. **NAT semantics differ per vendor.** Re-check every NAT-dependent rule against the target platform's order of operations (see the table in `agents/firewall-architect.md`).
5. **Every step has a rollback.**

---

## Phase 1 — Discover

| Item                       | How to collect                                           |
| -------------------------- | -------------------------------------------------------- |
| Full config export         | Native backup + text export (set/display set/XML)        |
| Rule hit counts            | ASA `show access-list`; PAN-OS rule usage; FortiOS policy stats; SRX `show security policies hit-count`; Check Point hit count column |
| NAT table                  | Running NAT policy + live translations (`show xlate`, sessions) |
| Routing table              | Full RIB, including dynamic routing neighbours            |
| VPN inventory              | Peers, crypto settings, proxy IDs / encryption domains, owners |
| Interfaces / VLANs / zones | Physical map, sub-interfaces, HA links                   |
| Management integrations    | Syslog, SNMP, NTP, AAA/RADIUS/TACACS, identity sources, SIEM |
| Critical flows             | List of business apps with owners and a test method for each |
| Remote-access VPN          | Users, groups, posture, MFA, client versions              |

Deliverable: **source inventory** (counts of objects, rules, NAT rules, VPNs, routes) — this becomes the baseline for validation.

## Phase 2 — Clean

- Rules with zero hits over a full business cycle (recommend ≥ 90 days, covering month-end/quarter-end) → disable first, delete later
- Shadowed and redundant rules (broader rule above narrower one)
- Duplicate objects (same value, different names) → pick one canonical name
- Unused objects and groups
- `any` in service or destination on allow rules → document or tighten
- Expired temporary rules (check descriptions / change references)

Get sign-off on the cleanup list from rule owners. Apply cleanup on the **source** device where possible, so the conversion input is already clean.

## Phase 3 — Design

| Decision                 | Notes                                                    |
| ------------------------ | -------------------------------------------------------- |
| Zone model               | Map source interfaces/security levels to target zones; avoid one-zone-per-interface if the target supports grouping |
| NAT model                | E.g. FortiOS policy-based vs central NAT; decide once    |
| Naming standard          | Respect target name-length limits (e.g. PAN-OS 63 chars) and allowed characters |
| Rule granularity         | One rule per business flow; group by application or owner |
| Logging & profiles       | Default profile group / UTM profile for every allow rule |
| HA design                | Active/passive vs active/active, session sync, monitoring |
| Management               | Central manager (Panorama, FMC, FortiManager, Security Director) hierarchy |
| Routing                  | Static vs dynamic, VRFs/virtual routers, PBR             |
| Remote access            | Client, auth, posture — often a separate workstream      |

Deliverable: **design document** + **interface/zone mapping table**.

## Phase 4 — Convert

Tooling (verify current availability and supported versions with the vendor):

| Target       | Vendor tool                                   |
| ------------ | --------------------------------------------- |
| Cisco FTD    | Cisco Secure Firewall Migration Tool (ASA, Check Point, PAN-OS, FortiGate sources) |
| FortiGate    | FortiConverter                                |
| PAN-OS       | Expedition (retired by the vendor — check current guidance) or scripted conversion via XML API |
| Any          | Scripted conversion (Python + vendor API / text parsing), reviewed in Git |

Rules for conversion:

- Convert objects → groups → services → NAT → security rules, in that order
- Keep a **mapping file** (source rule ID → target rule name) — it drives validation
- Flag every rule the tool could not convert fully; never let a tool silently drop a rule
- Review **all** NAT rules and **all** rules containing `any` by hand
- Keep the converted config in version control; every manual fix is a commit

## Phase 5 — Validate (pre-cutover)

See `validation-checklist.md`. Minimum:

- Object, rule and NAT counts match the cleaned source inventory (± documented deltas)
- Policy-match simulation for every critical flow on the target
- Peer review by a second engineer

## Phase 6 — Cutover

See `cutover-and-rollback.md`. Choose a strategy:

| Strategy             | How                                               | Risk / effort                     |
| -------------------- | ------------------------------------------------- | --------------------------------- |
| Big bang             | All interfaces swung in one window                | One outage window; large blast radius |
| Phased by segment    | Move one zone/VLAN/site at a time (routing or VLAN moves) | More windows; smaller blast radius |
| Parallel / shadow    | Target in-line in tap/monitor or behind routing  | Highest confidence; needs extra ports/IPs |
| Per-VPN              | Re-point VPN peers one at a time                  | Requires peer coordination        |

## Phase 7 — Hypercare & Optimise

- Review deny logs every few hours for the first 24–72 h; fix legitimate denies via emergency change
- Keep the old device powered, configured and cabled-ready for the agreed rollback period
- After 30 days: remove temporary rules; after 90 days: hit-count cleanup on the new platform
- Then start optimisation: App-ID / application control, user-based rules, decryption, profile tuning
- Decommission old device (wipe, licence transfer, CMDB update)

---

## RACI (example)

| Activity               | Firewall team | App owners | Network team | Change board |
| ---------------------- | ------------- | ---------- | ------------ | ------------ |
| Discovery / export     | R/A           | C          | C            | I            |
| Cleanup sign-off       | R             | A          | I            | I            |
| Conversion & review    | R/A           | I          | C            | I            |
| Test plan per flow     | C             | R/A        | I            | I            |
| Cutover execution      | R/A           | C          | R            | A (approve)  |
| Go/no-go decision      | R             | C          | C            | A            |
