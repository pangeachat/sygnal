---
applyTo: "**/.github/workflows/**,**/deploy*,**/docker/**"
---

# Sygnal Deployment

How the Sygnal push gateway is built and deployed. For org-wide deployment patterns, see [deployment.instructions.md](../../.github/.github/instructions/deployment.instructions.md).

## Architecture

Sygnal runs as a **Docker container on the Synapse EC2 instances**, managed by the [Ansible playbook](../../ansible/). It is **not** deployed via ECS/Fargate like the choreographer and CMS.

### Staging

| Component | Detail |
|-----------|--------|
| **Docker image** | `061565848348.dkr.ecr.us-east-1.amazonaws.com/pangea-staging-sygnal:main` |
| **CI/CD** | `.github/workflows/deploy-staging.yml` — push to `main` → ECR build → SSM deploy |
| **IAM (push)** | `github-deploy-staging` OIDC role — ECR push + SSM RunCommand |
| **IAM (pull)** | `CloudWatchAgentServerRole` — `ecr-pull` inline policy (Terraform: `staging/iam/ec2-ecr-pull`) |
| **Ansible vars** | [`ansible/inventory/staging/host_vars/matrix.staging.pangea.chat/vars.yml`](../../ansible/inventory/staging/host_vars/matrix.staging.pangea.chat/vars.yml) |
| **EC2 instance** | `i-028b5c19a329c8961` (`100.55.34.39`) |
| **Reverse proxy** | Traefik v3 — routes `sygnal.staging.pangea.chat` → container port 6000, Let's Encrypt TLS |
| **DNS** | CNAME `sygnal.staging.pangea.chat` → `matrix.staging.pangea.chat` (Terraform: `staging/dns/sygnal`) |
| **ECR repo** | Terraform: `staging/ecr/sygnal` |
| **Systemd service** | `matrix-sygnal` |

### Production

| Component | Detail |
|-----------|--------|
| **Docker image** | `061565848348.dkr.ecr.us-east-1.amazonaws.com/pangea-prod-sygnal:production` |
| **CI/CD** | `.github/workflows/deploy-production.yml` — GitHub release published → ECR build → SSM deploy → Matrix notification |
| **IAM (push)** | `github-deploy-prod` OIDC role — ECR push + SSM RunCommand |
| **IAM (pull)** | `CloudWatchAgentServerRole` — `ecr-pull-prod` inline policy (Terraform: `prod/iam/ec2-ecr-pull`) |
| **Ansible vars** | [`ansible/inventory/production/host_vars/matrix.pangea.chat/vars.yml`](../../ansible/inventory/production/host_vars/matrix.pangea.chat/vars.yml) |
| **EC2 instance** | `i-074c64998e68a5bc6` (`52.203.29.202`) |
| **Reverse proxy** | Traefik — routes `sygnal.pangea.chat` → container port 6000 |
| **DNS** | CNAME `sygnal.pangea.chat` → `matrix.pangea.chat` (Terraform: `prod/dns/sygnal`) |
| **ECR repo** | Terraform: `prod/ecr/sygnal` |
| **Systemd service** | `matrix-sygnal` |

## Staging Deploys (CI/CD)

Every push to `main` triggers the deploy workflow:

1. **Build & push** — Docker Buildx builds `linux/amd64` image, pushes to ECR with `:main` and `:${SHA}` tags
2. **Deploy** — OIDC auth → SSM RunCommand on staging EC2:
   - `docker pull` the new image (ECR auth via `amazon-ecr-credential-helper`)
   - `systemctl restart matrix-sygnal`
   - Health check via `systemctl is-active`

## Production Deploys (CI/CD)

Publishing a GitHub release triggers the deploy workflow:

1. **Build & push** — Docker Buildx builds `linux/amd64` image, pushes to ECR with `:production` and `:${SHA}` tags
2. **Deploy** — OIDC auth → SSM RunCommand on production EC2:
   - `docker pull` the new image (ECR auth via `amazon-ecr-credential-helper`)
   - `systemctl restart matrix-sygnal`
   - Health check via `systemctl is-active`
3. **Notify** — Posts release summary to the team Matrix room (via reusable workflow)

To deploy: create a GitHub release using the [release template](../../.github/RELEASE_TEMPLATE.md). `workflow_dispatch` is also available for manual re-deploys.

### Bootstrap (First-Time Setup)

Before CI/CD can deploy, Ansible must bootstrap the Sygnal container, ECR credential helper, and Traefik route.

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

**Note:** Both envs share a single `CloudWatchAgentServerRole` IAM role (created outside Terraform). The ECR pull policies are named `ecr-pull` (staging) and `ecr-pull-prod` (production) to avoid collision. The OIDC provider is created by staging; production references it via data source (`create_oidc_provider = false`).

## Client Integration

The client sets the push gateway URL in [`client/lib/config/setting_keys.dart`](../../client/lib/config/setting_keys.dart):

```dart
pushNotificationsGatewayUrl: 'https://sygnal.pangea.chat/_matrix/push/v1/notify'
```

The client registers a pusher with Synapse via `POST /_matrix/client/v3/pushers/set`. Synapse forwards push payloads to Sygnal → FCM/APNs.

## Future Work

None at this time. All infrastructure is managed by Terraform.
