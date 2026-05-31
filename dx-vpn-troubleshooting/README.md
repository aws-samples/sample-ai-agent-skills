# Direct Connect & Site-to-Site VPN Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Direct Connect and Site-to-Site VPN problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Direct Connect and Site-to-Site VPN when the console alone isn't enough — DX connection failures, VIF provisioning, BGP session debugging, VPN tunnel establishment, IKE/IPsec negotiation, routing propagation, failover validation, Transit Gateway integration, MACsec encryption, LAG configuration, and performance issues.

### Activate When

- DX connection state is down or degraded
- VIF creation or provisioning failures
- BGP session not establishing or flapping
- LAG member connection issues
- VPN tunnel down or unstable
- IKE Phase 1 or Phase 2 negotiation failures
- Routes not propagating over BGP
- Route table misconfiguration
- Asymmetric routing between DX and VPN
- DX-to-VPN failover not working
- Transit Gateway attachment or routing issues
- MACsec or VPN encryption problems
- Bandwidth or latency issues
- DX provisioning or VPN migration questions

---

## Skill Structure

```
dx-vpn-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-connection-down.md
    ├── A2-vif-creation-failures.md
    ├── A3-bgp-session-issues.md
    ├── A4-lag-problems.md
    ├── B1-tunnel-down.md
    ├── B2-ike-negotiation-failures.md
    ├── B3-vpn-routing-issues.md
    ├── B4-vpn-throughput-limits.md
    ├── C1-bgp-route-propagation.md
    ├── C2-route-table-issues.md
    ├── C3-asymmetric-routing.md
    ├── D1-dx-to-vpn-failover.md
    ├── D2-active-active-passive.md
    ├── E1-tgw-attachment-issues.md
    ├── E2-tgw-route-table.md
    ├── F1-macsec-encryption.md
    ├── F2-vpn-encryption.md
    ├── G1-bandwidth-issues.md
    ├── G2-latency.md
    ├── H1-dx-setup-provisioning.md
    ├── H2-vpn-migration.md
    ├── Z1-general-troubleshooting.md
    ├── dx-vpn-guardrails.md
    └── dx-vpn-hallucination-patterns.yaml
```

---

## Runbook Library (28 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Direct Connect** | A1–A4 | Connection down, VIF creation failures, BGP session issues, LAG problems |
| **B — VPN** | B1–B4 | Tunnel down, IKE negotiation failures, routing issues, throughput limits |
| **C — Routing** | C1–C3 | BGP route propagation, route table issues, asymmetric routing |
| **D — Failover** | D1–D2 | DX-to-VPN failover, active-active vs active-passive |
| **E — Transit Gateway** | E1–E2 | TGW attachment issues, TGW route table |
| **F — Security** | F1–F2 | MACsec encryption, VPN encryption |
| **G — Performance** | G1–G2 | Bandwidth issues, latency |
| **H — Migration** | H1–H2 | DX setup/provisioning, VPN migration |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## License

MIT-0
