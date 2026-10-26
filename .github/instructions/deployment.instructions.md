---
applyTo: "**/.github/workflows/**,**/deploy*,**/docker/**"
---

# Sygnal Deployment

How the Sygnal push gateway is built and deployed. For org-wide deployment patterns, see [deployment.instructions.md](../../../.github/.github/instructions/deployment.instructions.md).

## Architecture

Sygnal runs as a **Docker container on the Synapse EC2 instances**, managed by the [Ansible playbook](../../../ansible/). It is **not** deployed via ECS/Fargate like the choreographer and CMS.

### Staging

| Component | Detail |
|-----------|--------|
| **Docker image** | Host-local `localhost/pangea-sygnal:current`, tagged from `061565848348.dkr.ecr.us-east-1.amazonaws.com/pangea-staging-sygnal:<sha>` by each deploy. ECR tags are immutable; nothing in ECR moves. |
| **CI/CD** | `.github/workflows/deploy-staging.yml` — push to `main` → ECR build → SSM deploy |
| **IAM (push)** | `github-deploy-staging` OIDC role — ECR push + SSM RunCommand |
| **IAM (pull)** | `CloudWatchAgentServerRole` — `ecr-pull` inline policy (Terraform: `staging/iam/ec2-ecr-pull`) |
| **Ansible vars** | [`ansible/inventory/staging/host_vars/matrix.staging.pangea.chat/vars.yml`](../../../ansible/inventory/staging/host_vars/matrix.staging.pangea.chat/vars.yml) |
| **EC2 instance** | `i-028b5c19a329c8961` (`54.144.22.101`) |
| **Systemd service** | `matrix-sygnal` |

### Production

| Component | Detail |
|-----------|--------|
| **Docker image** | Host-local `localhost/pangea-sygnal:current`, tagged from `061565848348.dkr.ecr.us-east-1.amazonaws.com/pangea-prod-sygnal:<sha>` by each deploy. ECR tags are immutable; nothing in ECR moves. |
| **CI/CD** | `.github/workflows/deploy-production.yml` — GitHub release published → ECR build → SSM deploy → Matrix notification |
| **IAM (push)** | `github-deploy-prod` OIDC role — ECR push + SSM RunCommand |
| **IAM (pull)** | `CloudWatchAgentServerRole` — `ecr-pull-prod` inline policy (Terraform: `prod/iam/ec2-ecr-pull`) |
| **Ansible vars** | [`ansible/inventory/production/host_vars/matrix.pangea.chat/vars.yml`](../../../ansible/inventory/production/host_vars/matrix.pangea.chat/vars.yml) |
| **EC2 instance** | `i-074c64998e68a5bc6` (`52.203.29.202`) |
| **Systemd service** | `matrix-sygnal` |

## Staging Deploys (CI/CD)

Every push to `main` triggers the deploy workflow:

1. **Build & push** — skipped when `:${SHA}` is already in ECR; otherwise Docker Buildx builds the `linux/amd64` image and pushes it with only the `:${SHA}` tag
2. **Deploy** — OIDC auth → SSM RunCommand on staging EC2:
   - `docker pull` the `:${SHA}` image (ECR auth via `amazon-ecr-credential-helper`) and tag it `localhost/pangea-sygnal:current`
   - `systemctl restart matrix-sygnal`
   - Health check via `systemctl is-active`

## Production Deploys (CI/CD)

Publishing a GitHub release triggers the deploy workflow:

1. **Build & push** — skipped when `:${SHA}` is already in ECR; otherwise Docker Buildx builds the `linux/amd64` image and pushes it with only the `:${SHA}` tag
2. **Deploy** — OIDC auth → SSM RunCommand on production EC2:
   - `docker pull` the `:${SHA}` image (ECR auth via `amazon-ecr-credential-helper`) and tag it `localhost/pangea-sygnal:current`
   - `systemctl restart matrix-sygnal`
   - Health check via `systemctl is-active`
3. **Notify** — Posts release summary to the team Matrix room (inlined steps — cannot use reusable workflow from private `.github` repo since sygnal is public)

To deploy: create a GitHub release using the [release template](../../../.github/RELEASE_TEMPLATE.md). `workflow_dispatch` is also available for manual re-deploys.

## Rollback

To roll back, run the deploy workflow on a tag or branch that points at the older commit: for production, run Deploy to Production on an earlier release tag; for staging, run Deploy to Staging on a branch or tag at that commit. If that commit's image is still in ECR it is reused, not rebuilt; older commits are rebuilt from source. Releases and tags from before the switch to host-local tags cannot be redeployed this way.

### Bootstrap (First-Time Setup)

Before CI/CD can deploy, Ansible must bootstrap the Sygnal container, ECR credential helper, and Traefik route.

Ansible's `matrix_sygnal_docker_image_source` names the ECR image a fresh host starts from. The role tags it as `current` only when that tag is missing, so playbook runs never undo a CI deploy. Update the SHA there when you rebuild a host. If `current` disappears from a host that already ran Sygnal (for example after `docker system prune -a` while Sygnal was stopped), the next playbook run recreates it from that pinned SHA, which may be older than what was running, so deploy the latest commit again afterwards.

**Important:** Firebase credentials are deployed by the `auxiliary` role (`install-aux` tag), not the `sygnal` role. The Firebase Admin SDK JSON file (`pangea-chat-936ee-firebase-adminsdk-*.json`) must be present in Sygnal's data directory before the container starts. Include `install-aux` in the bootstrap:

```bash
# Push an initial image to ECR first (crane copy or run the workflow once)
ansible-playbook -i inventory/<env>/hosts setup.yml --tags=install-aux,setup-sygnal,start
```

### Config-Only Changes

If only Ansible config changes (e.g., FCM apps), re-run the playbook:

```bash
ansible-playbook -i inventory/<env>/hosts setup.yml --tags=setup-sygnal,start
```

## Infrastructure (Terraform)

Managed in `devops/terraform/{staging,prod}/`:

| Resource | Staging | Production |
|----------|---------|------------|
| ECR repository | `staging/ecr/sygnal/` | `prod/ecr/sygnal/` |
| DNS CNAME | `staging/dns/sygnal/` | `prod/dns/sygnal/` |
| OIDC role (ECR push + SSM) | `staging/iam/github-oidc/` | `prod/iam/github-oidc/` |
| EC2 ECR pull policy | `staging/iam/ec2-ecr-pull/` | `prod/iam/ec2-ecr-pull/` |

## Client Integration

The client sets the push gateway URL in [`client/lib/config/setting_keys.dart`](../../../client/lib/config/setting_keys.dart):

```dart
pushNotificationsGatewayUrl: 'https://sygnal.pangea.chat/_matrix/push/v1/notify'
```

The client registers a pusher with Synapse via `POST /_matrix/client/v3/pushers/set`. Synapse forwards push payloads to Sygnal → FCM/APNs.
