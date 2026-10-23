# Pure FlashArray over iSCSI on CloudStack KVM

Staging installation guide

## Overview

This adds a Pure FlashArray pool as a second primary storage tier next to NFS. CloudStack creates
one array volume per CloudStack volume and attaches it to the KVM host as a raw multipath block
device. Offerings tagged `pure-block` land on the array; everything else stays on NFS.

CloudStack's built-in `Flash Array` provider only discovers LUNs over Fibre Channel. Our hosts use
iSCSI over 25GbE, so one host-side script is replaced. The management server runs stock CloudStack.

```
VM -> virtio -> /dev/mapper/3624a9370... -> dm-multipath -> iSCSI (4 paths) -> FlashArray
                       ^ connectVolume.sh (ours) discovers the LUN here
```

## Prerequisites

| Item | Requirement |
|------|-------------|
| CloudStack | 4.19.1 or newer, staging environment |
| KVM hosts | Two, Ubuntu 22.04/24.04 or RHEL 9 family, storage NICs on the iSCSI VLAN |
| FlashArray | Purity 6.x, iSCSI interfaces up, REST API reachable from the management server |
| Workstation | `git`, `ansible` 2.14+, `cmk` configured against staging, SSH as root to the KVM hosts |

## Values to collect

Fill this in before starting. Examples are illustrative.

| Value | Example | Where |
|-------|---------|-------|
| Array management IP | `10.10.10.5` | Pure GUI, Settings > Network |
| iSCSI portal IPs | `10.10.20.11` `.12` `.13` `.14` | Pure GUI, Settings > Network, service `iscsi` |
| Pure Pod name | `cloudstack-staging` | You create it in step 1 |
| API token | `a1b2c3d4-...` | You create it in step 1 |
| KVM hostnames | `kvm-stg-01`, `kvm-stg-02` | `hostname -s` on each host |
| KVM host IQNs | `iqn.2004-10.com.ubuntu:01:9f3e...` | `/etc/iscsi/initiatorname.iscsi` |
| CloudStack zone ID | `a3f2...` | `cmk list zones filter=id,name` |
| Pool capacity (bytes) | `10995116277760` (10 TiB) | Your decision |

Note: a Pure "Pod" is an array-side namespace. It is unrelated to a CloudStack pod.

## Step 1: FlashArray

1. Pods > Create. Name: `cloudstack-staging`.
2. Settings > Access > Users. Create or pick a user with the `storage_admin` role, then
   Create API Token. Copy the token; it is shown once.
3. Hosts > Create, one per KVM host. Name must equal `hostname -s` on the host. Leave ports
   empty for now; you add the IQN in step 2.

## Step 2: KVM hosts

1. On each host, read the IQN. Install `open-iscsi` (Ubuntu) or `iscsi-initiator-utils` (RHEL) first if the file is missing.

   ```bash
   cat /etc/iscsi/initiatorname.iscsi
   # InitiatorName=iqn.2004-10.com.ubuntu:01:9f3e5b2a7c1d
   ```

2. On the array, open each Host object > Host Ports > Configure IQNs, paste the host's IQN.

3. On your workstation, deploy the host configuration:

   ```bash
   git clone https://github.com/tphil/Simnet-cloudstack.git
   cd Simnet-cloudstack/storage/pure-flasharray-iscsi/ansible
   cp inventory.example.ini inventory.ini
   ```

   Edit `inventory.ini`:

   ```ini
   [kvm_hosts]
   kvm-stg-01 ansible_host=10.10.0.11
   kvm-stg-02 ansible_host=10.10.0.12

   [kvm_hosts:vars]
   ansible_user=root
   pure_iscsi_portals=["10.10.20.11", "10.10.20.12", "10.10.20.13", "10.10.20.14"]
   ```

   Run:

   ```bash
   ansible-playbook -i inventory.ini playbook.yml
   ```

   The playbook installs `multipath-tools`, `open-iscsi`, `lsscsi`, `sg3-utils`; deploys
   `multipath.conf`, `iscsid.conf` and udev rules; logs in to every portal; and installs the
   patched `connectVolume.sh` into `/usr/share/cloudstack-common/scripts/storage/multipath/`.
   It is idempotent; run it again any time.

4. Verify on each host:

   ```bash
   iscsiadm -m session
   # tcp: [1] 10.10.20.11:3260,1 iqn.2010-06.com.purestorage:flasharray.5a1b2c3d (non-flash)
   # tcp: [2] 10.10.20.12:3260,1 iqn.2010-06.com.purestorage:flasharray.5a1b2c3d (non-flash)
   # tcp: [3] 10.10.20.13:3260,1 iqn.2010-06.com.purestorage:flasharray.5a1b2c3d (non-flash)
   # tcp: [4] 10.10.20.14:3260,1 iqn.2010-06.com.purestorage:flasharray.5a1b2c3d (non-flash)

   systemctl is-active multipathd iscsid
   # active
   # active
   ```

5. Data path test. On the array create a 10 GB volume `test-kvm-stg-01` inside the Pod and
   connect it to host `kvm-stg-01`. On that host:

   ```bash
   iscsiadm -m session --rescan
   multipath -ll
   # 3624a93701234567890abcdef00011234 dm-3 PURE,FlashArray
   # size=10G features='0' hwhandler='1 alua' wp=rw
   # `-+- policy='queue-length 0' prio=50 status=active
   #   |- 12:0:0:1 sdb 8:16 active ready running
   #   |- 13:0:0:1 sdc 8:32 active ready running
   #   |- 14:0:0:1 sdd 8:48 active ready running
   #   `- 15:0:0:1 sde 8:64 active ready running
   ```

   Four paths, all `active ready running`. Then disconnect the volume on the array and clean
   up on the host:

   ```bash
   cd /usr/share/cloudstack-common/scripts/storage/multipath
   ./disconnectVolume.sh 624a93701234567890abcdef00011234
   ```

   Delete the test volume on the array.

## Step 3: Add the pool

```bash
cmk create storagepool \
  zoneid=a3f2b1c0-0000-0000-0000-000000000001 \
  scope=zone hypervisor=KVM \
  name=pure-block \
  provider="Flash Array" \
  url="https://10.10.10.5:443?pod=cloudstack-staging" \
  "details[0].api_token=a1b2c3d4-e5f6-7890-abcd-ef1234567890" \
  tags=pure-block \
  capacitybytes=10995116277760
```

For a cluster-scoped pool use `scope=cluster podid=<cloudstack-pod-id> clusterid=<cluster-id>`
instead of `scope=zone`.

If the array uses a self-signed certificate, add `"details[1].api_skiptlsvalidation=true"`.

Verify:

```bash
cmk list storagepools name=pure-block filter=name,state,type,tags
# name: pure-block   state: Up   type: FiberChannel   tags: pure-block
```

`type: FiberChannel` is expected. It is the plugin's internal pool type and is unrelated to the transport.

## Step 4: Offerings

```bash
cmk create diskoffering \
  name=pure-block-thin displaytext="Block storage (Pure, thin)" \
  customized=true storagetype=shared provisioningtype=thin tags=pure-block

cmk create serviceoffering \
  name=pure-2c4g displaytext="2 vCPU, 4 GB, Pure root disk" \
  cpunumber=2 cpuspeed=2000 memory=4096 \
  storagetype=shared provisioningtype=thin tags=pure-block
```

Provisioning type must be `thin` or `sparse`. `fat` is rejected by the plugin.

## Step 5: Verify end to end

| # | Action | Expected |
|---|--------|----------|
| 1 | Deploy a VM with offering `pure-2c4g` | VM reaches `Running`; `multipath -ll` on its host shows the root volume |
| 2 | Attach a `pure-block-thin` 20 GB disk | Disk appears in the guest (`lsblk`) |
| 3 | Snapshot the data disk, create a volume from it | Both complete; the array Pod shows the new volume |
| 4 | Live migrate the VM to the other host | VM stays up; volume now mapped on the destination host only |
| 5 | Destroy and expunge the VM and disk | Volumes leave the Pod (they sit in the array's Destroyed list for 24h, then vanish) |

Full acceptance list: `storage/pure-flasharray-iscsi/tests/acceptance-checklist.md`.

## Troubleshooting

| Symptom | Check | Fix |
|---------|-------|-----|
| `create storagepool` fails: `Pod [x] not found` | Pod name in `url` | Must match the Pure Pod exactly |
| `create storagepool` fails: 401 or auth error | Token, user role | Recreate token for a `storage_admin` user |
| `create storagepool` fails: TLS/certificate | Array cert | Add `api_skiptlsvalidation=true` detail, or install a trusted cert on the array |
| VM deploy hangs at volume attach | On host: `journalctl -t CS_SCSI_VOL_FIND -n 50` | If `awaiting disk path` repeats, verify the Pure Host object name equals `hostname -s` and `iscsiadm -m session` shows all portals |
| `multipath -ll` shows `failed faulty` paths | NIC link, VLAN, portal reachability (`ping <portal>`) | Fix the network; paths recover automatically |
| Stale `/dev/mapper/3624a...` entries after deletes | `multipath -ll` | The agent runs `cleanStaleMaps.sh` every 60s; otherwise `multipath -f <wwid>` |
| Script missing after `cloudstack-agent` upgrade | `grep -c iscsi_host /usr/share/cloudstack-common/scripts/storage/multipath/connectVolume.sh` returns 0 | Re-run the playbook |

Logs: management server `/var/log/cloudstack/management/management-server.log` (filter `FlashArray`),
KVM host `/var/log/cloudstack/agent/agent.log` and `journalctl -t CS_SCSI_VOL_FIND`.

## Rollback

```bash
cmk update storagepool id=<pool-id> enabled=false     # stop new placements
# migrate or destroy any volumes still on the pool, then:
cmk delete storagepool id=<pool-id>
```

Hosts keep their iSCSI sessions and the patched script; both are harmless without the pool.

## Maintenance

- After every `cloudstack-agent` package upgrade, re-run the playbook. It restores the patched script.
- After a CloudStack upgrade, diff `upstream/connectVolume.sh` against the new release before re-deploying.
- Adding a KVM host: Pure Host object with its IQN, add to inventory, run the playbook.
