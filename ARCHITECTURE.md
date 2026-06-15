# Architecture & design rationale

> Illustrative values throughout — a generic three-VLAN example, not a live deployment.

## 1. Topology

```
                 Internet
                    │  WAN (igb0)
              ┌─────┴─────┐
              │  pfSense   │  LAN trunk igb1 (802.1Q, tagged 10,20,30)
              └─────┬─────┘
              ┌─────┴─────┐
              │  managed   │  (802.1Q switch — configured by hand)
              │  switch    │
              └┬───┬───┬──┬┘
   tag10,30    │   │U20│U20          U = untagged / PVID
        ┌──────┘   │   │
   ┌────┴────┐  ┌──┴┐ ┌┴──────────┐
   │ FreeBSD │  │cmp│ │ operator   │   VLAN20 OPERATOR  10.0.20.0/24
   │  host   │  └───┘ └────────────┘
   │ (jails) │
   └─┬─────┬─┘
 bridge10 bridge30
   │        │
 bastion  store
 10.0.10.10 10.0.30.10
 (bastion) (file store)

VLAN10 MGMT     gw 10.0.10.1   firewall admin (bastion = SOLE source)
VLAN20 OPERATOR gw 10.0.20.1   egress → VPN (kill-switch)
VLAN30 SERVICES gw 10.0.30.1   egress → VPN ; store SMB → operators
MGMT egress → WAN direct (updates only)
```

## 2. Data model (`group_vars/all.yml`)

One declarative file feeds every playbook. Key blocks:

| Block | Purpose |
|---|---|
| `hardware` | WAN/LAN/VPN interface names |
| `vlans` | id / subnet / gateway / prefix per tier |
| `hosts` | bastion, store, firewall, switch, operator hosts |
| `ports` | GUI / SSH / SMB / DNS |
| `gateways` | `WAN_GW`, `VPN_GW` |
| `vpn` | listen port, monitor IP, full-tunnel flag (keys from vault) |
| `admin` | `sole_source`, `disable_anti_lockout`, `scaffold_admin_ip`, `legacy_lan_subnet` |
| `freebsd` | trunk, bridges, jails |
| `samba` | store-jail share (SMB3 sealed, hosts_allow operator net) |
| `decisions` | feature toggles that change which rules are emitted |

The `decisions` block is the design's flexibility knob: e.g. `bastion_ssh`,
`store_internet`, `mgmt_internet`, `operator_isolation` each gate specific rules, so the
same code renders different postures from data alone.

## 3. Phase pipeline

| # | Playbook | What it does |
|---|---|---|
| 00 | `00_bootstrap` | SSH check + baseline `config.xml` snapshot |
| 10 | `10_vlans` | MGMT / OPERATOR / SERVICES interfaces (additive) |
| 12 | `12_dhcp` | DHCP for operator devices (MGMT/SERVICES static) |
| 15 | `15_vpn_egress` | VPN egress interface + monitored gateway |
| 18 | `18_scaffold_admin` | **anti-lockout scaffold** — temp admin rule (removed in 60) |
| 20 | `20_firewall` | segmentation rule tables (toggle-driven) |
| 25 | `25_nat_killswitch` | outbound NAT scoping + fail-closed behaviour |
| 30/40 | `suricata` / `log_forward` | optional IDS + off-box logging |
| 50 | `50_wireguard_admin` | break-glass admin WireGuard (2nd admin path) |
| 70 | `70_freebsd_host` | FreeBSD host: VLAN legs, VNET jails, Samba |
| 99 | `99_validate` | config asserts + reachability matrix |
| 60 | `60_lockdown` | **LAST** — remove scaffold, enforce bastion-only, auto-revert |

`site.yml` runs `00 → 70` and **stops before lockdown** on purpose. Lockdown and
validation are deliberate, separate commands.

## 4. Admin & lockout model

- **Primary:** operator → SSH bastion → firewall GUI/SSH. The bastion is the *only*
  source permitted to the admin plane (a single rule at the top of MGMT).
- **Break-glass:** a WireGuard tunnel terminating on the firewall (phase 50) — an
  out-of-band path if the bastion chain is down.
- **Backstop:** a serial / physical console — segment-independent, always works.
- **Build rails:** phase 18 keeps your workstation connected during the build; phase 60
  removes that rule and disables the anti-lockout rule, armed with a **5-minute
  auto-revert** so a mistake self-heals.

Why this matters: tightening a firewall *while administering it over that same firewall*
is the classic way to brick remote access. The scaffold + auto-revert + console make the
dangerous step recoverable by construction.

## 5. Segmentation & the egress kill-switch (rationale)

Each interface is **default-deny**; tiers are blocked from lateral movement to the lab
supernet, and only specific flows are permitted (operator → bastion SSH; operator → store
SMB; everyone → DNS on the firewall). Internet-bound tiers egress **only** through the
VPN gateway:

- The internet rule routes via `VPN_GW`; a `block → LAB_NET` rule sits above it, so only
  non-lab (internet) traffic ever reaches the VPN gateway — "permit internet, deny
  lateral" expressed with positive matches.
- Outbound NAT exists **only** for operator/services → VPN and mgmt/self → WAN. The
  deliberate *absence* of a WAN NAT for the internet tiers means a stray packet has no
  translation path out the WAN.
- When the monitored VPN gateway is marked down, the VPN pass rules stop matching and
  traffic falls through to default-deny — **fail closed**, no fallback.

The result is asserted in phase 99: no operator/services rule routes `any` via WAN.

> The exact firewall setting names for gateway-down behaviour vary by pfSense release, so
> the relevant phase flags them as **verify-on-target** and the runbook insists on a live
> drop-the-tunnel test rather than trusting the config alone.

## 6. FreeBSD VNET jails

The services tier lives on a separate FreeBSD host that trunks VLAN 10 + 30 to the
switch. Each jail gets its own network stack (`vnet`) via an `epair` bridged onto its
VLAN, with its own default route to the firewall gateway for that VLAN. The host is **not**
a router (`gateway_enable=NO`); jails route through the firewall like any other host, so
the same segmentation rules apply to them. The store jail runs Samba with SMB3 sealing,
bound to its VLAN address and allowed only from the operator subnet.

## 7. The manual edges (honestly noted)

Two things are deliberately *not* fully automated, because the tooling can't do them
cleanly and pretending otherwise would be a footgun:

- **WireGuard tunnels** — pfSense CE fully supports WireGuard, but `pfsensible.core` has
  no WireGuard module, so the tunnel/peer objects are created once in the GUI (or via a
  templated `config.xml` fragment) and Ansible owns everything around them.
- **The managed switch** — 802.1Q tagging/PVIDs are configured by hand in the switch UI
  (trunk the VLANs to the firewall, tag 10/30 to the FreeBSD host, untag PVID-20 to the
  operator ports, move switch management onto the MGMT VLAN before removing VLAN 1).

Both are called out in the playbooks with `[VERIFY]` markers and in `docs/runbook.md`.
