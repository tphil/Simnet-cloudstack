# Development plan: Pure FlashArray block storage tier

Goal: a second primary storage tier on the existing Pure FlashArray, one array volume per
CloudStack volume, over iSCSI on 25GbE. Sold as the high-IOPS tier in Simnet.

Each stage gates the next.

## Stage 0: setup

- [ ] Record CloudStack version, Purity version, host OS, storage NICs per host
- [ ] Staging CloudStack with two KVM hosts
- [ ] Pod on the array for staging, API token
- [ ] This repo pushed, infra team has access

## Stage 1: make it work (1 to 2 weeks)

- [ ] Data path by hand, no CloudStack: iSCSI login, array volume visible under `/dev/mapper/3624a9370...` (checklist A1 to A3)
- [ ] Patched `connectVolume.sh` and `disconnectVolume.sh` run by hand (A4, A5)
- [ ] Host reboot keeps sessions, no stale maps (A6)
- [ ] Ansible deploys everything to both lab hosts from scratch
- [ ] Pool added to staging CloudStack, one disk offering and one compute offering tagged `pure-block`

Gate: a VM boots from a Pure root disk on staging.

## Stage 2: make it trustworthy (2 to 3 weeks)

- [ ] Functional cases B1 to B38 pass
- [ ] Failure cases C1 to C8 pass
- [ ] Performance D1 to D5 recorded, NFS vs Pure, with and without noisy neighbour
- [ ] `cloudstack-agent` upgrade followed by Ansible run restores the script (C8)
- [ ] Runbook in `docs/runbooks/` reviewed by someone who did not write it

Gate: checklist complete, numbers shared with management.

## Stage 3: production

- [ ] Pool added to production as tagged tier
- [ ] One internal tenant for 2 to 4 weeks, monitoring on multipath path state, iSCSI session count, stale maps
- [ ] Open to customers, mapped to Simnet offering tiers

## Later, only if needed

- Java work in a CloudStack fork: auto-create Pure host objects from host IQN, auto-login to
  portals, `protocol=iscsi` pool option. Worth it only if manual host onboarding hurts.
- Upstream PR to Apache CloudStack once stable in production (issue #10610 asks for this).
- NVMe/TCP: different device model (native NVMe multipath), separate project.
