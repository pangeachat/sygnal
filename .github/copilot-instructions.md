# sygnal — Pangea fork of Element's Matrix push gateway

Fork of [element-hq/sygnal](https://github.com/element-hq/sygnal). Adds visible FCM notification content for iOS (upstream sends data-only, which iOS throttles/drops).

- **Upstream**: element-hq/sygnal — fork is synced to upstream main
- **License**: AGPL-3.0 + Element Commercial (matches upstream)
- **Our changes**: Two upstream PRs pending — [#431](https://github.com/element-hq/sygnal/pull/431) (async create fix) and [#432](https://github.com/element-hq/sygnal/pull/432) (notification content)
- **Production branch**: `notification-request-updates` — currently deployed (see [deployment.instructions.md](instructions/deployment.instructions.md))
- **Deployed**: Production only (`localhost/pangeachat/sygnal:main`, built on EC2). Staging has no Sygnal yet.
- **Firebase**: Project `pangea-chat-936ee`, apps `com.talktolearn.chat` + `com.talktolearn.chat.data_message`
- **Config**: Managed by ansible repo (`pangeachat/ansible`), not this repo
- **Legacy repo**: `pangeachat/sygnal-legacy` (archived) — old fork of matrix-org/sygnal

| [deployment.instructions.md](instructions/deployment.instructions.md) | Build, deploy, Ansible config, client integration, what's non-standard |

## Open work
- Deploy to staging for parity
- Standardize deployment (CI/CD pipeline, IaC for DNS)
- Once upstream PRs are merged, switch production to upstream main + config-only changes
