# Simnet-cloudstack

CloudStack customizations, plugins and operational tooling for the Simnet platform.
Everything that changes how our CloudStack behaves and does not ship with upstream lives here.

## Layout

| Path | What |
|------|------|
| `storage/pure-flasharray-iscsi/` | Pure FlashArray as managed block storage over iSCSI (upstream plugin is FC-only) |
| `plugins/` | Java-side CloudStack plugin work, when we need it (fork notes, patches) |
| `docs/runbooks/` | Short operator instructions, one file per task |
| `docs/development-plan.md` | Roadmap and stage gates for the storage work |

Each feature folder is self-contained: patched files, pristine upstream copies for diffing,
configs, Ansible to deploy, tests to prove it. Read the folder's `README.md` first.

## Rules

- Keep diffs against upstream minimal and documented. Every patched file has its upstream
  twin next to it under `upstream/`, with the CloudStack tag it came from.
- Nothing is deployed to a host by hand. If it is not in Ansible here, it does not exist.
- Staging CloudStack first, always. Production only after the acceptance checklist passes.
- Anything stable for a few months in production is a candidate for an upstream PR.
