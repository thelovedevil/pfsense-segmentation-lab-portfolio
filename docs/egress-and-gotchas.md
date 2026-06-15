# Egress matrix & operational gotchas

> Illustrative notes for the generic three-tier topology — the kind of operational detail that
> separates "playbooks" from "a system someone can actually run." Read before touching gateways,
> NAT, or moving a device between VLANs.

## What egresses where (the split-tunnel design)

The VPN is **only** for the untrusted/services tiers' *internet* traffic. The management/router
plane and all internal east-west traffic are deliberately **not** tunneled.

| Source → destination | Path | Why |
|---|---|---|
| OPERATOR / SERVICES → internet | **VPN** (fail-closed) | privacy + leak-proof egress for the untrusted tiers |
| MGMT (incl. FreeBSD host + bastion jail) → internet | **WAN direct** | the admin plane must stay reachable even if the VPN drops; infra updates want a reliable path; the bastion→firewall path is LAN-local anyway |
| the firewall itself → internet | **WAN direct** | the WireGuard underlay *rides on* WAN — it cannot tunnel itself |
| legacy flat LAN → internet | **WAN direct** | transitional; retired after migration |
| OPERATOR → store SMB (`10.0.30.10:445`) | **local inter-VLAN routing — NOT tunneled** | internal host; SMB3-sealed on the wire; tunnelling an internal copy out and back is pointless |
| OPERATOR → bastion SSH, → DNS (firewall), any allowed east-west | **local inter-VLAN routing** | internal; governed by the segmentation rules, not the VPN |

Rule of thumb: **internet-bound traffic from the untrusted tiers → VPN; everything internal or
management → local/WAN.** Only the OPERATOR/SERVICES *internet* rules carry `route-to <tunnel>`;
the SMB/SSH/DNS rules are plain LAN passes that match first (first-match-wins), so internal
traffic never reaches the VPN rule.

## Gotchas (the non-obvious operational landmines)

### 1. Pin the default gateway to WAN — never leave it "Automatic"
On Automatic, pfSense routes the system default through the "best online" gateway. The WAN
monitor often falsely reads **down** (ISPs drop ICMP to the monitored next-hop), and once the VPN
gateway is **online**, pfSense moves the **default route onto the tunnel** → everything (including
management) tries to egress via the VPN → full outage.
- **Fix:** System → Routing → Gateways → Default gateway (IPv4) = the WAN gateway, *explicitly*.
- **Also:** set the WAN gateway's **monitor IP** to something that answers ICMP (e.g. `9.9.9.9`).
  `1.1.1.1` rate-limits ICMP from VPN exits — don't use it for monitoring.

### 2. Operators can't manage the firewall — plan the control node
The OPERATOR rules block operator → firewall admin. Moving a control-node device onto the operator
VLAN to VPN its traffic **costs it firewall management** (SSH/GUI/Ansible) — the firewall drops the
admin connection regardless of source NAT or subnet (it's the segmentation rule, not the network).
- To have a device that's both VPN'd **and** able to admin: add a top-of-OPERATOR allow rule for
  its IP → `ADMIN_PORTS` (pin the IP via a static/DHCP reservation), or manage via the **bastion** /
  **WireGuard break-glass** instead.
- DHCP is **per-VLAN**: an operator-port device gets a `10.0.20.x` lease, not the flat-LAN scope.

### 3. WireGuard specifics
- pfsensible has no WireGuard module, and `pfsense_rewrite_config` **only reformats** config.xml —
  it cannot inject a tunnel. **Create the tunnel + peer in the GUI.**
- The assigned WG interface needs a **static IPv4 = the tunnel address** for a gateway to attach;
  the gateway IP is the **far-side tunnel address** with **`nonlocalgateway`** (outside the /32).
- Set the WG interface **MTU 1420** (1500 drops large packets).
- Kill-switch config keys on recent pfSense: **`skip_rules_gw_down` + `gw_down_kill_states`**
  (not `skiprules`).

### 4. Outbound NAT → Manual is GUI-only
Flip Outbound NAT to **Manual in the GUI** (it *copies* the existing auto rules → no connectivity
loss). Then delete the **operator/services → WAN** copies so those tiers translate **only** onto the
VPN interface — that absence of a WAN translation path is the leak-proofing.
