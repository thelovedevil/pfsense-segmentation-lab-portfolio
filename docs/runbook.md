# Runbook

> Illustrative operational notes. Always keep a serial console available while applying
> any phase that touches the admin plane or segmentation.

## Apply a phase

```bash
# Always dry-run first
ansible-playbook playbooks/<phase>.yml --check --diff

# Apply
ansible-playbook playbooks/<phase>.yml --diff

# Validate, then commit your IaC
ansible-playbook playbooks/99_validate.yml
git add -A && git commit -m "phase-N: <summary>"
```

## Ordered build (full path)

```
00 bootstrap → 10 vlans → 12 dhcp → 15 vpn_egress → 18 scaffold_admin
  → 20 firewall → 25 nat_killswitch → (30 suricata, 40 log_forward)
  → 50 wireguard_admin → 70 freebsd_host → 99 validate
  → confirm bastion + break-glass access → 60 lockdown (cancel the auto-revert)
```

`site.yml` runs `00 → 70` and stops before lockdown. Lockdown is always a separate,
deliberate command.

## Rollback a phase

Every playbook carries a `rollback` tag that restores the newest pre-phase snapshot:

```bash
ansible-playbook playbooks/<phase>.yml --tags rollback
```

Snapshots are also fetched to `snapshots/` on the control node before each phase.

## Emergency console recovery

If you lock yourself out:

1. Open the serial/console shell on the firewall.
2. `ls -lt /cf/conf/backup/`
3. `cp /cf/conf/backup/config-<ts>.xml /conf/config.xml && /etc/rc.reload_all`
4. Do **not** factory-reset — it wipes the config.

## Lockdown safety (commit-confirm)

`60_lockdown` arms a 5-minute auto-revert. After it runs, confirm you still have admin via
the bastion **and** the break-glass WireGuard tunnel, then cancel the revert:

```bash
ssh pfsense 'rm -f /tmp/pf_autorevert.flag'      # or: --tags cancel_revert
```

Do nothing and the previous config is restored automatically.

## Managed switch (manual, one time)

802.1Q config is done by hand in the switch UI, in this order (verify access at each step):

1. Create VLANs 10 / 20 / 30.
2. Trunk port → firewall: tag 10, 20, 30. Host port → FreeBSD: tag 10, 30.
3. Operator ports: untagged, PVID 20.
4. Move switch management onto the MGMT VLAN **before** removing ports from VLAN 1.
5. Remove every port from VLAN 1; save.

Keep the switch reset path handy — a mis-trunk can drop switch + segment connectivity at
once.

## Vault operations

```bash
ansible-vault encrypt host_vars/pfsense.vault.yml
ansible-vault edit    host_vars/pfsense.vault.yml
```

The vault password lives in `.vault_pass` (gitignored). The real `*.vault.yml` never
enters git.

## Suricata IDS → IPS promotion (future)

1. Run IDS (alert-only) for 1–2 weeks; suppress confirmed false positives.
2. Switch to inline/blocking mode.
3. Watch logs 24h for broken legitimate traffic before trusting it.
