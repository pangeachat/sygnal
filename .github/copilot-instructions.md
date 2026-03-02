# sygnal — Pangea fork of Element's Matrix push gateway

Fork of [element-hq/sygnal](https://github.com/element-hq/sygnal). Adds visible FCM notification content for iOS (upstream sends data-only, which iOS throttles/drops).

- **Upstream**: element-hq/sygnal — fork is synced to upstream main
- **License**: AGPL-3.0 + Element Commercial (matches upstream)
- **Our changes**: Two upstream PRs pending — [#431](https://github.com/element-hq/sygnal/pull/431) (async create fix) and [#432](https://github.com/element-hq/sygnal/pull/432) (notification content)
- **Staging**: CI/CD via ECR + OIDC + SSM (push to `main` auto-deploys). See [deployment.instructions.md](instructions/deployment.instructions.md)
- **Production**: Manual deploy — image built on EC2 (`localhost/pangeachat/sygnal:main`)
- **Firebase**: Project `pangea-chat-936ee`, apps `com.talktolearn.chat` + `com.talktolearn.chat.data_message`
- **Config**: Managed by ansible repo (`pangeachat/ansible`), not this repo
- **Infrastructure**: Managed by devops repo (`pangeachat/devops`) — ECR, DNS, IAM all in Terraform
- **Legacy repo**: `pangeachat/sygnal-legacy` (archived) — old fork of matrix-org/sygnal

| [deployment.instructions.md](instructions/deployment.instructions.md) | Build, deploy, CI/CD, Ansible config, Terraform infra |

## Open work
- Add CI/CD for production (mirror staging ECR + SSM pattern)
- Move production DNS CNAME into Terraform
- Once upstream PRs are merged, switch production to upstream main + config-only changes
