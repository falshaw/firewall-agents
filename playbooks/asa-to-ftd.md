# Playbook: Cisco ASA → Cisco Secure Firewall Threat Defense (FTD)

Read `00-methodology.md` first. Agent: `cisco-security-expert`.

## Tooling

Use the **Cisco Secure Firewall Migration Tool (FMT)**: it ingests an ASA config (file or live), maps interfaces to FTD interfaces and security zones/interface groups, and pushes objects, ACLs as access control rules, NAT, routes and site-to-site VPN to FMC. Check the FMT release notes for supported source versions and features.

What FMT does not finish for you: anything without an FMC equivalent (handled through FlexConfig or redesign), remote access VPN details, and design improvements.

## Object & Policy Mapping

| ASA                            | FTD / FMC                                         |
| ------------------------------ | ------------------------------------------------- |
| `nameif` + `security-level`    | Logical name + **security zone** or interface group (no security levels on FTD) |
| `object network`               | Network object                                    |
| `object-group network`         | Network group                                     |
| `object service` / group       | Port object / group                               |
| Interface ACL                  | Access control policy (ACP) rule with source/destination zones |
| Global ACL                     | ACP rules with `any` zones                        |
| ACE with high hit count, L3/L4 only, no inspection needed | Prefilter rule (Fastpath) or ACP **Trust** |
| ACE that should be inspected   | ACP **Allow** + intrusion policy + file policy    |
| Implicit deny                  | ACP default action (Block all traffic) — enable logging |
| `time-range`                   | Time range object                                 |
| `remark`                       | Rule comment                                      |

### Rule actions on FTD

| Action        | Effect                                                         |
| ------------- | -------------------------------------------------------------- |
| Prefilter Fastpath | Bypasses Snort entirely                                   |
| Trust         | Passes without further inspection (Snort sees the first packets) |
| Allow         | Permits, subject to intrusion/file/malware inspection          |
| Block / Block with reset | Drop / reset                                        |
| Monitor       | Log and continue evaluation                                    |

**Design choice:** ASA ACLs are stateful L3/L4 only. Converting everything to Allow + IPS adds inspection load; converting everything to Trust gives ASA-like behaviour without the NGFW value. Typical approach: Allow + balanced IPS policy for user/internet flows; Trust or Fastpath for high-volume, low-risk east-west flows (backups, replication).

## NAT

FTD NAT uses the same model as ASA 8.3+:

| ASA                    | FTD (FMC NAT policy)                    |
| ---------------------- | --------------------------------------- |
| Object (auto) NAT      | Auto NAT rule                           |
| Twice NAT              | Manual NAT rule (before / after auto)   |
| NAT exemption          | Manual static identity NAT with `no-proxy-arp` and `route-lookup` |
| `interface` PAT        | Dynamic PAT to destination interface IP |

ACP rules match the **real (untranslated) IP**, same as ASA 8.3+ ACLs — no address rewrite needed.

## VPN

| ASA                          | FTD / FMC                                       |
| ---------------------------- | ----------------------------------------------- |
| Crypto map + ACL             | Policy-based site-to-site topology (protected networks) |
| VTI                          | Route-based site-to-site (VTI)                  |
| `tunnel-group` (L2L)         | Endpoint in the topology                        |
| `tunnel-group` + `group-policy` (RA) | Remote access VPN: connection profile + group policy |
| AnyConnect images / profiles | Secure Client packages and profiles uploaded to FMC |
| Dynamic Access Policies      | Supported in FMC 7.x (verify version)           |

## Features Needing Attention

| ASA feature                        | FTD handling                                      |
| ---------------------------------- | ------------------------------------------------- |
| Multiple context mode              | Multi-instance on supported hardware              |
| Some `inspect` engines and options | Default inspections exist; non-default via FlexConfig |
| EIGRP, PBR, some routing knobs     | Supported natively on recent FMC; older via FlexConfig |
| `management-access`, banners, specific `sysopt` | FlexConfig or platform settings  |
| Threat detection / shunning        | Replaced by Snort-based intrusion policies       |
| Local user database for RA         | Local realm (FMC 7.x) or external AAA            |

## Verification

```
> show conn address 10.20.0.10
> show nat detail
> show xlate
> packet-tracer input outside tcp 198.51.100.20 40000 203.0.113.10 443 detailed
> system support firewall-engine-debug        # see which ACP rule matched in Snort
> system support trace                        # packet-tracer-style live trace incl. Snort
> show crypto ipsec sa
```

In FMC: Unified Events / Connection Events filtered by the flow; Health Monitor for Snort CPU.

## Gotchas Checklist

- [ ] Every ASA interface mapped to a zone or interface group
- [ ] Security-level implicit permits enumerated as rules
- [ ] Trust vs Allow decided per rule category
- [ ] Prefilter rules reviewed (they bypass all NGFW inspection)
- [ ] ACP default action logs
- [ ] FlexConfig objects reviewed and documented
- [ ] Remote access VPN tested with each client OS before cutover
- [ ] Snort CPU baseline measured under real load
