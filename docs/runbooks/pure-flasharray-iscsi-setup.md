# CloudStack + Pure FlashArray over iSCSI: staging setup

Adds a FlashArray pool as a second primary storage tier next to NFS. One array volume per
CloudStack volume, attached to KVM hosts over iSCSI on the 25GbE storage network.

CloudStack's built-in `Flash Array` plugin is Fibre Channel only on the host side. We replace
one script on each KVM host so it also discovers LUNs over iSCSI. The management server runs
stock CloudStack.

## Requirements

- Staging CloudStack 4.19.1 or newer, two KVM hosts
- KVM hosts on the same network as the array's iSCSI interfaces
- Purity REST API reachable from the management server
- `git` and `ansible` on the machine you run this from, SSH root access to the KVM hosts

## 1. FlashArray

1. Create a Pod named `cloudstack-staging`.
2. Note the iSCSI portal IPs (Settings > Network, interfaces with the `iscsi` service).
3. Create an API token for a storage-admin user.

## 2. KVM hosts

1. On each host: `cat /etc/iscsi/initiatorname.iscsi` (install `open-iscsi` first if missing).
2. On the array: create a Host object per KVM host, named exactly like the hostname (`hostname -s`), with that IQN.
3. On your machine:

```bash
git clone https://github.com/tphil/Simnet-cloudstack.git
cd Simnet-cloudstack/storage/pure-flasharray-iscsi/ansible
cp inventory.example.ini inventory.ini
# edit inventory.ini: host IPs and pure_iscsi_portals
ansible-playbook -i inventory.ini playbook.yml
```

Check on each host:

```bash
iscsiadm -m session            # one line per portal
systemctl is-active multipathd # active
```

Then on the array: create a 10 GB test volume, connect it to one host. On that host:

```bash
iscsiadm -m session --rescan
multipath -ll                  # shows 3624a9370... with all paths active
```

Disconnect and delete the test volume afterwards.

## 3. Add the pool

```bash
cmk create storagepool \
  zoneid=<zone-id> scope=zone hypervisor=KVM \
  name=pure-block provider="Flash Array" \
  url="https://<array-mgmt-ip>:443?pod=cloudstack-staging" \
  "details[0].api_token=<token>" \
  tags=pure-block capacitybytes=<bytes>
```

Check: `cmk list storagepools name=pure-block` shows `state: Up`.

## 4. Offerings

Create a disk offering and a compute offering with storage tag `pure-block` and provisioning
type `thin`. Tagged offerings land on the FlashArray; untagged ones stay on NFS.

## 5. Verify

1. Create a VM from the `pure-block` compute offering. It boots.
2. Attach a `pure-block` data disk. It appears in the guest.
3. Snapshot the data disk, create a volume from the snapshot.
4. Live migrate the VM to the other host.
5. Delete the VM and disk. Volumes disappear from the Pod on the array.

If step 1 hangs, on the host run `journalctl -t CS_SCSI_VOL_FIND` and send the output.

Full test list: `storage/pure-flasharray-iscsi/tests/acceptance-checklist.md`.

## After a cloudstack-agent upgrade

The package overwrites our script. Re-run the playbook from step 2.
