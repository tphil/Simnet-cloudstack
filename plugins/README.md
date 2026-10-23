# CloudStack plugin (Java) work

Empty on purpose. Nothing here is needed for the iSCSI tier; that is host scripts only.

If Java changes become necessary (see `docs/development-plan.md`, "Later, only if needed"):

1. Fork `apache/cloudstack`, branch from the exact release tag we run in production.
2. Keep changes inside `plugins/storage/volume/flasharray`, `plugins/storage/volume/adaptive`
   and `scripts/storage/multipath`. Nothing else.
3. Store the changes here as a patch series (`git format-patch`), one patch per concern,
   so rebasing onto the next release is mechanical.
4. CloudStack ships the management server as one shaded jar, so a Java change means building
   our own `cloudstack-management` package from the fork. Document the build here when it happens.
