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

### 5. DNS leaks silently even with a working VPN tunnel
A working WireGuard tunnel does NOT mean DNS is safe. Three independent leak vectors:
- **ISP DHCP injection:** most ISPs push their DNS servers via DHCP. pfSense accepts them by
  default (`dnsallowoverride` present = enabled). Your resolv.conf fills with ISP entries, and
  every lookup leaks your queries to the ISP in plaintext.
  **Fix:** remove `<dnsallowoverride>` from config.xml. Add explicit DNS servers (e.g. Quad9)
  with their gateway set to the VPN gateway.
- **Unbound root-resolver mode:** out of the box, Unbound does iterative root resolution — DNS
  queries go directly to root/TLD servers via whatever route the OS picks (usually WAN). Even
  with DNS servers configured, Unbound ignores them unless **forwarding mode** is enabled.
  **Fix:** set `<forwarding/>` in the Unbound config, and set the outgoing interface to the
  VPN tunnel interface.
- **IPv6 DNS:** pfSense's default ruleset allows all IPv6 out from the firewall host. This lets
  Unbound send IPv6 DNS out WAN even when IPv4 DNS is tunneled.
  **Fix:** add floating block rules for inet6 port 53 (DNS) and 853 (DoT) out WAN, and set
  `do-ip6: no` in Unbound custom options.
- Phase 15 automates all three fixes via `pfsense_phpshell`.

### 6. VPN provider NAT can break ICMP gateway monitoring (dpinger)
Some VPN providers' server-side NAT rewrites ICMP echo-reply destinations: you send from the tunnel
IP, the reply comes back, but the provider delivers it with a different destination address. dpinger
never sees the response → gateway shows 100% loss / "down" permanently.
- **Fix:** set `monitor_disable: true` on the VPN gateway. The kill-switch still works because
  `skip_rules_gw_down` + `gw_down_kill_states` operate on the WireGuard interface state (link
  down / handshake timeout), not on dpinger status alone.
- **Also:** do NOT use the same monitor IP for both WAN and VPN gateways.
  `setup_gateways_monitor()` creates conflicting host routes — whichever route wins, the other
  gateway's dpinger gets the wrong path.
