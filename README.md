# pfSense Segmentation Lab — Infrastructure-as-Code network hardening

Declarative, **lockout-safe** hardening of a home/lab network: one source of truth in
Ansible renders a flat LAN into a segmented **pfSense CE** firewall (VLANs, default-deny
between tiers, a fail-closed VPN egress kill-switch, bastion-only administration) plus a
**FreeBSD host** running VNET jails. Every change is dry-run first, snapshotted, and
applied in phases that can't paint you into a corner.

> ℹ️ **Portfolio version.** This is a sanitized, public copy of a private project.
> All addresses, the VPN provider, hardware, and rule specifics are **generic
> illustrative examples** and do not reflect any live deployment. It shows the
> engineering approach — the methodology, structure, and safety model — not an
> operational configuration. Secrets and live values never enter this repo.

---

## The problem

A typical prosumer network is one flat L2 segment: every device can reach every other
device and the firewall's admin GUI, and all traffic egresses straight to the ISP. That
means no blast-radius control, no separation between trusted admin and untrusted compute,
and no protection if a single host is compromised.

The goal of this project was to turn that flat network into a **segmented, least-privilege**
design **as code** — repeatable, reviewable, and reversible — without ever risking a
permanent lockout of the firewall during the migration.

## The design (illustrative)

Three VLAN tiers behind pfSense, an isolated services tier hosted in FreeBSD VNET jails,
and policy-routed egress through a commercial WireGuard VPN with a fail-closed kill-switch.

```
                 Internet
                    │ WAN
              ┌─────┴─────┐
              │  pfSense   │  802.1Q trunk (tagged VLAN 10/20/30)
              └─────┬─────┘
              ┌─────┴─────┐
              │  managed   │  (802.1Q switch — configured by hand)
              │  switch    │
              └┬────┬────┬─┘
       tag10,30│    │U20 │U20
        ┌──────┘    │    │
   ┌────┴────┐   ┌──┴─┐ ┌┴───────────┐
   │ FreeBSD │   │cmp │ │ operator   │   VLAN20 OPERATOR — egress via VPN (kill-switch)
   │  host   │   │node│ │ workstation│
   │ (jails) │   └────┘ └────────────┘
   └─┬────┬──┘
  bridge10 bridge30
    │       │
 bastion  store          VLAN10 MGMT     — bastion jail = SOLE firewall admin source
 jail     jail           VLAN30 SERVICES — store jail (SMB3) serves operators only
```

See **[ARCHITECTURE.md](ARCHITECTURE.md)** for the data model, the phase pipeline, the
admin/lockout model, and the kill-switch design rationale.

## What makes it interesting (engineering highlights)

- **Single source of truth.** One `group_vars/all.yml` drives every playbook — VLANs,
  hosts, ports, gateways, and feature toggles. Change a subnet in one place; the whole
  ruleset re-renders.
- **Lockout-safe phase pipeline.** Build steps are numbered `00 → 70` and applied in an
  order that always keeps an admin path open. A temporary "scaffold" admin rule
  (phase 18) holds your workstation in during the build; the final lockdown (phase 60)
  removes it — and is the *only* step that tightens admin access.
- **Commit-confirm lockdown with auto-revert.** The point-of-no-return phase arms a
  5-minute timer that restores the previous config unless you explicitly confirm you
  still have access — the same "did this change lock me out?" safety used by good
  network automation.
- **Snapshot before every phase.** Each playbook fetches `config.xml` to the control
  node first, so every phase has a clean rollback target; a `rollback` tag restores the
  newest on-box backup.
- **Dry-run discipline.** `--check --diff` before every apply; `site.yml` deliberately
  stops *before* lockdown so the dangerous step is always a separate, deliberate action.
- **Fail-closed egress.** Internet-bound tiers are policy-routed through the VPN with no
  WAN fallback beneath them, so a dropped tunnel fails closed instead of leaking.
- **Defense in depth for admin.** Primary path = operator → bastion → firewall; an
  out-of-band WireGuard "break-glass" tunnel is a second path; a serial console is the
  always-works backstop.

## Tech stack

| Area | Tooling |
|---|---|
| Config management | Ansible, `pfsensible.core` (pfSense CE config over SSH), `community.general` |
| Firewall | pfSense CE — VLANs, rules, outbound NAT, gateways, Suricata IDS |
| Host virtualization | FreeBSD VNET jails, bridged VLAN legs, Samba (SMB3) |
| Secrets | `ansible-vault`, gitignored vault files, dedicated SSH keypair |
| Egress | Commercial WireGuard VPN with monitored-gateway kill-switch |

## Repo layout

```
pfsense-segmentation-lab/
├── README.md                     # this case study
├── ARCHITECTURE.md               # topology, data model, phase pipeline, design rationale
├── ansible.cfg                   # remote_user, key, roles_path, vault file
├── requirements.yml              # pfsensible.core + community.general
├── inventory/hosts.yml           # firewall + freebsd group + (optional) lab group
├── group_vars/all.example.yml    # ★ SINGLE SOURCE OF TRUTH (copy → all.yml)
├── host_vars/
│   └── pfsense.vault.example.yml # secret template (real *.vault.yml is gitignored)
├── site.yml                      # orchestrator: runs 00→70, STOPS before lockdown
├── playbooks/                    # 00 bootstrap … 70 freebsd, 99 validate, 60 lockdown
├── roles/freebsd_host/           # VLAN legs, VNET jails, Samba (tasks + templates)
└── docs/                         # threat_model.md, runbook.md
```

## Quick start (against your own gear)

```bash
# 1. Control node: Ansible + collections + a dedicated key
sudo apt install -y ansible git        # or: pipx install ansible
ansible-galaxy collection install -r requirements.yml
ssh-keygen -t ed25519 -f ~/.ssh/pfsense_ansible -N ""

# 2. Configure (copy the examples, fill in YOUR values)
cp group_vars/all.example.yml group_vars/all.yml
cp host_vars/pfsense.vault.example.yml host_vars/pfsense.vault.yml
$EDITOR group_vars/all.yml host_vars/pfsense.vault.yml
ansible-vault encrypt host_vars/pfsense.vault.yml

# 3. Baseline + dry-run the whole chain
ansible-playbook playbooks/00_bootstrap.yml
ansible-playbook site.yml --check --diff

# 4. Apply phase-by-phase (--check --diff before each), validate, THEN lock down
#    See ARCHITECTURE.md for the full ordered runbook.
```

## Skills demonstrated

Infrastructure-as-Code · idempotent configuration management · network segmentation &
least-privilege firewall design · threat modeling · safe-change methodology
(dry-run, snapshot, commit-confirm rollback) · FreeBSD jails/networking · secrets
hygiene · pfSense / WireGuard / Suricata.

## What I'd do next

- Close the IDS→IPS loop: promote Suricata to inline blocking after a false-positive
  tuning window, and wire alerts into an automated response.
- CI for the IaC: lint + a `--check` dry-run against a pfSense VM on every PR.
- Replace the GUI-owned WireGuard step with a fully-templated `config.xml` fragment so
  the entire build is hands-off.

---

*Sanitized for public release. No live addresses, credentials, or operational specifics
are included.*
