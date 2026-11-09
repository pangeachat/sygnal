# sygnal — Pangea fork of Element's Matrix push gateway

Fork of [element-hq/sygnal](https://github.com/element-hq/sygnal). Adds visible FCM notification content for iOS (upstream sends data-only, which iOS throttles/drops).

- **Upstream**: element-hq/sygnal — fork is synced to upstream main
- **License**: AGPL-3.0 + Element Commercial (matches upstream)
- **Our changes**: Two upstream PRs pending — [#431](https://github.com/element-hq/sygnal/pull/431) (async create fix) and [#432](https://github.com/element-hq/sygnal/pull/432) (notification content)
- **Dead tokens are rejected** (fork-only): an FCM v1 `SENDER_ID_MISMATCH` (token from another Firebase project) or APNs `BadDeviceToken` rejects the pushkey, so Synapse drops the pusher instead of failing every push to it. Upstream treats both as a permanent failure and keeps the pusher. A wrong Firebase project in config would also show as `SENDER_ID_MISMATCH` and would then remove every pusher, so check the project first when these spike.
- **Staging**: CI/CD via ECR + OIDC + SSM (push to `main` auto-deploys). See [deployment.instructions.md](instructions/deployment.instructions.md)
- **Production**: CI on GitHub release; see [deployment.instructions.md](instructions/deployment.instructions.md)
- **Firebase**: Per environment — `pangea-chat-936ee` (prod), `pangea-chat-staging-analytics` (staging); apps `com.talktolearn.chat` + `com.talktolearn.chat.data_message`
- **Config**: Managed by ansible repo (`pangeachat/ansible`), not this repo
- **Infrastructure**: Managed by devops repo (`pangeachat/devops`) — ECR, DNS, IAM all in Terraform
- **Legacy repo**: `pangeachat/sygnal-legacy` (archived) — old fork of matrix-org/sygnal

| [deployment.instructions.md](instructions/deployment.instructions.md) | Build, deploy, CI/CD, Ansible config, Terraform infra |

## Open work
- Once upstream PRs are merged, switch production to upstream main + config-only changes
