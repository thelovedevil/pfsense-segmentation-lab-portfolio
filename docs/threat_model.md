# Threat model

> Illustrative — describes the classes of risk this design addresses and how, not a live
> assessment of any specific network.

## What this hardens

| Attack surface | Mitigation |
|---|---|
| Firewall GUI/SSH reachable from the whole LAN | Admin plane on the MGMT VLAN; a single bastion host is the sole permitted source; anti-lockout rule disabled |
| Flat L2 — any host reaches any host | VLAN segmentation with default-deny between tiers; no lateral movement to the lab supernet |
| Compromised endpoint pivoting | Services tier isolated in VNET jails; the store jail cannot initiate outbound to other tiers (replies pass by state only) |
| Spoofed RFC1918 / bogon ingress on WAN | Explicit block-private + block-bogon on WAN |
| ISP-visible egress / IP exposure | Internet tiers policy-routed through a VPN; **fail-closed** when the tunnel drops |
| Undetected intrusion | Suricata IDS (ET Open) with EVE JSON forwarded off-box |
| Config drift / no change audit | Everything Ansible-managed, git-tracked, with a `config.xml` snapshot before each phase |
| Locking yourself out while hardening | Scaffold admin rule during the build + commit-confirm auto-revert on lockdown + serial console backstop |

## Explicitly out of scope (for this version)

| Risk | Status |
|---|---|
| Inline IPS (active blocking) | Future — requires a false-positive tuning window first |
| High-fidelity commercial threat-intel rules | Cost-gated; ET Open used here |
| Endpoint posture / NAC | Out of scope — this is firewall + host segmentation only |
| Automated IDS-alert → block feedback loop | Future enhancement |

## Residual risks worth stating

- **Verify-on-target settings.** Some firewall behaviours (gateway-down handling, the
  anti-lockout toggle) have release-specific config keys, so the design treats them as
  *verify-on-target* and requires a live functional test (drop the tunnel; confirm egress
  stops) rather than trusting the written config.
- **GUI-owned WireGuard.** Until the tunnel is fully templated, the WireGuard object is a
  manual step outside Ansible's idempotency guarantees.
- **Single physical switch.** A mis-trunk can drop switch + segment connectivity at once;
  the runbook keeps the switch reset path handy.

## Rollback impact

| Phase rolled back | What breaks |
|---|---|
| `10_vlans` | VLAN interfaces removed — hosts on new VLANs lose connectivity |
| `20_firewall` | Segmentation rules removed — tiers can reach each other again |
| `25_nat_killswitch` | Kill-switch removed — egress may fall back to WAN (leak risk) |
| `50_wireguard_admin` | Break-glass path removed — only bastion + console remain |
| `60_lockdown` | Anti-lockout re-enabled, scaffold restored — admin plane re-opens |
