# Acceptance checklist: Pure FlashArray over iSCSI

Run on staging CloudStack with two KVM hosts. Mark each case pass/fail with date and tester.

## A. Data path (no CloudStack)

| # | Case | Result |
|---|------|--------|
| A1 | `iscsiadm -m session` shows one session per Pure portal on each host | |
| A2 | Volume created on array and connected to host appears as `/dev/mapper/3624a9370...` after `iscsiadm -m session --rescan` | |
| A3 | `multipath -ll` shows all paths active, `prio` alua, policy `queue-length 0` | |
| A4 | `connectVolume.sh <lun> <wwid>` run by hand finds the volume and exits 0 | |
| A5 | `disconnectVolume.sh <wwid>` removes the map and all slave devices | |
| A6 | Host reboot: sessions re-established automatically, no stale maps | |

## B. CloudStack functional (from upstream plugin README)

| # | Case | Result |
|---|------|--------|
| B1 | Create zone-scope Pure pool | |
| B2 | Create cluster-scope Pure pool | |
| B3 | Update pool settings (zone and cluster) | |
| B4 | VM with root disk on Pure | |
| B5 | VM with root and data disk on Pure | |
| B6 | VM root on NFS, data on Pure | |
| B7 | VM root on Pure, data on NFS | |
| B8 | Snapshot root disk (Pure) | |
| B9 | Snapshot data disk (Pure) | |
| B10 | VM snapshot (no memory), root and data on Pure | |
| B11 | VM snapshot (no memory), root Pure, data NFS | |
| B12 | VM snapshot (no memory), root NFS, data Pure | |
| B13 | Template from Pure root snapshot | |
| B14 | Volume from Pure root snapshot | |
| B15 | Volume from Pure data snapshot | |
| B16 | New VM from B13 template, root on Pure | |
| B17 | New VM from B13 template, root on NFS | |
| B18 | Delete Pure snapshot | |
| B19 | Delete attached volume on running VM (must fail) | |
| B20 | Delete attached volume on stopped VM (must fail) | |
| B21 | Detach volume from stopped VM | |
| B22 | Attach volume to running VM | |
| B23 | Attach volume to stopped VM | |
| B24 | Thin disk offering tagged for Pure, attach | |
| B25 | Sparse disk offering tagged for Pure, attach | |
| B26 | Fat disk offering tagged for Pure (must fail) | |
| B27 | Migrate root volume Pure to NFS, stopped VM | |
| B28 | Migrate root volume NFS to Pure, stopped VM | |
| B29 | Migrate data volume Pure to NFS, stopped VM | |
| B30 | Migrate data volume NFS to Pure, stopped VM | |
| B31 | Migrate VM with volumes, all Pure to all NFS | |
| B32 | Migrate VM with volumes, all NFS to all Pure | |
| B33 | Live migrate VM with Pure root | |
| B34 | Live migrate VM with Pure data, NFS root | |
| B35 | Live migrate VM with Pure root, NFS data | |
| B36 | Migrate volume between two Pure pools, same array | |
| B37 | Resize Pure data volume on running VM, guest sees new size | |
| B38 | Delete VM and volumes, array Pod shows volumes removed | |

## C. Failure handling

| # | Case | Expected | Result |
|---|------|----------|--------|
| C1 | Pull one storage NIC during `fio` in guest | I/O continues, one path failed in `multipath -ll`, no guest error | |
| C2 | Restore NIC | Path returns active within 30s | |
| C3 | Pull both storage NICs for 10s | Guest I/O stalls then resumes, no filesystem errors | |
| C4 | Reboot KVM host with running VMs | HA restarts VMs elsewhere, no stale maps on rebooted host | |
| C5 | Restart `cloudstack-agent` during volume attach | Attach completes or fails cleanly and is retryable | |
| C6 | Restart management server during volume create | Volume ends Ready or Expunged, no orphan on array | |
| C7 | Pure controller failover | Guest I/O pauses under 30s, all paths recover | |
| C8 | Upgrade `cloudstack-agent` package | Patched script overwritten, Ansible run restores it | |

## D. Performance (see `fio/`)

| # | Case | NFS | Pure iSCSI |
|---|------|-----|------------|
| D1 | 4K random read IOPS, single VM | | |
| D2 | 8K random write with fsync, single VM | | |
| D3 | 8K random write with fsync, with noisy neighbour on same pool | | |
| D4 | 1M sequential read MB/s | | |
| D5 | Latency p99 during D3 | | |
