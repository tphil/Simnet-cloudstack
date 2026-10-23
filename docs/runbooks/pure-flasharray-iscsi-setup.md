# CloudStack + Pure FlashArray over iSCSI: setup

Adds a FlashArray pool as a second primary storage tier next to NFS. One array volume per
CloudStack volume, attached to KVM hosts over iSCSI on the 25GbE storage network.

## Requirements

- CloudStack 4.19.1 or newer (provider `Flash Array` is built in)
- KVM hosts with 25GbE on the same network as the array's iSCSI interfaces
- Purity REST API reachable from the management server
- Patched `connectVolume.sh` from this repo (upstream is FC-only)

## 1. FlashArray

1. Create a Pod, e.g. `cloudstack`. All CloudStack volumes land in it.
2. Note the iSCSI portal IPs (Settings > Network, interfaces with the iscsi service).
3. Create an API token for a storage-admin user. Keep it for step 3.

## 2. KVM hosts (every host)

1. Read the host IQN: `cat /etc/iscsi/initiatorname.iscsi`
2. On the array, create a Host object named exactly like the KVM hostname (short name is fine) and add that IQN.
3. Put the portal IPs in the Ansible inventory and run:

```bash
ansible-playbook -i inventory.ini storage/pure-flasharray-iscsi/ansible/playbook.yml --limit <host>
```

The role installs packages, deploys `multipath.conf`, `iscsid.conf` and udev rules, logs in to
the portals, and installs the patched script.

Check: `iscsiadm -m session` lists one session per portal. Present a test volume from the array,
run `iscsiadm -m session --rescan`, and `multipath -ll` must show it as `3624a9370...` with all
paths active. Remove the test volume after.

## 3. Add the pool in CloudStack

```bash
cmk create storagepool \
  zoneid=<zone-id> scope=zone hypervisor=KVM \
  name=pure-block provider="Flash Array" \
  url="https://<array-mgmt-ip>:443?pod=cloudstack" \
  "details[0].api_token=<token>" \
  tags=pure-block capacitybytes=<bytes>
```

Use `scope=cluster podid=<pod-id> clusterid=<cluster-id>` instead of `scope=zone` to limit it to one cluster.

Check: `cmk list storagepools name=pure-block` shows `state: Up`.

## 4. Offerings

Create a disk offering and a compute offering with storage tag `pure-block` and provisioning
type `thin`. Everything tagged `pure-block` is placed on the FlashArray; untagged offerings
stay on NFS.

## 5. Verify

1. Create a VM from the `pure-block` compute offering. Boots.
2. Attach a `pure-block` data disk. Appears in the guest.
3. Snapshot the data disk, create a volume from the snapshot. Works.
4. Live migrate the VM to another host. Works.
5. Delete the VM and disk. Volumes disappear from the Pod on the array.

Full test list: `storage/pure-flasharray-iscsi/tests/acceptance-checklist.md`.

## After a cloudstack-agent upgrade

The package overwrites the patched script. Re-run the playbook from step 2.
