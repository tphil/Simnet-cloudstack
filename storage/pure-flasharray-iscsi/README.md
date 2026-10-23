# Pure FlashArray over iSCSI for CloudStack KVM

CloudStack 4.19+ ships a managed-storage plugin for Pure FlashArray (`provider="Flash Array"`,
source in `plugins/storage/volume/flasharray` and `plugins/storage/volume/adaptive`). It creates
one array volume per CloudStack volume and attaches it to KVM hosts as a raw multipath device.

Upstream only supports Fibre Channel: the host-side discovery script scans `/sys/class/fc_host`.
Our hosts reach the array over 25GbE, so we add iSCSI discovery. The management server runs
stock CloudStack; only one host-side script changes.

## What is in here

| Path | What |
|------|------|
| `scripts/connectVolume.sh` | Patched. Scans FC and iSCSI SCSI hosts, rescans iSCSI sessions. Drop-in for `/usr/share/cloudstack-common/scripts/storage/multipath/` |
| `upstream/` | Pristine scripts from CloudStack tag in `upstream/CLOUDSTACK_TAG`, for diffing |
| `config/` | `multipath.conf`, `iscsid.conf`, udev rules (Pure recommended settings) |
| `ansible/` | Role `pure_iscsi_host`: packages, configs, iSCSI login, patched script |
| `tests/acceptance-checklist.md` | Data path, 38 functional cases, failure cases, performance |
| `tests/fio/` | Benchmark jobs for NFS vs Pure comparison |

## Diff against upstream

```bash
diff -u upstream/connectVolume.sh scripts/connectVolume.sh
```

Two changes: a `scsi_hosts()` helper that lists `/sys/class/fc_host` and `/sys/class/iscsi_host`
host numbers, used in both the stale-device cleanup loop and the scan loop; and an
`iscsi_rescan()` call (`iscsiadm -m session --rescan`) after each scan. FC-only hosts behave
exactly as before.

## Deploy to a host

1. Put the host's IQN (`/etc/iscsi/initiatorname.iscsi`) on a Pure host object named after the
   KVM hostname.
2. Set `pure_iscsi_portals` in the inventory.
3. `ansible-playbook -i inventory.ini ansible/playbook.yml --limit <host>`
4. Verify: `iscsiadm -m session` shows one session per portal, `multipath -ll` is clean.

Re-run the playbook after every `cloudstack-agent` upgrade. It restores the patched script.

## Status

Stage 1 (make it work). Not yet run against a real array. See `docs/development-plan.md`.
