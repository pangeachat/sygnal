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
| **Docker image** | `localhost/pangeachat/sygnal:main` — built directly on the EC2 instance |
| **CI/CD** | **None.** Manual SSH + docker build + systemd restart. |
| **Ansible vars** | [`ansible/inventory/production/host_vars/matrix.pangea.chat/vars.yml`](../../ansible/inventory/production/host_vars/matrix.pangea.chat/vars.yml) |
| **Reverse proxy** | Traefik — routes `sygnal.pangea.chat` → container port 6000 |
| **DNS** | CNAME `sygnal.pangea.chat` → `matrix.pangea.chat` (Route 53, not yet in Terraform) |
| **Systemd service** | `matrix-sygnal` |

## Staging Deploys (CI/CD)

Every push to `main` triggers the deploy workflow:

1. **Build & push** — Docker Buildx builds `linux/amd64` image, pushes to ECR with `:main` and `:${SHA}` tags
2. **Deploy** — OIDC auth → SSM RunCommand on staging EC2:
   - `aws ecr get-login-password | docker login` (ECR auth on-instance)
   - `docker pull` the new image
   - `systemctl restart matrix-sygnal`
   - Health check via `systemctl is-active`

### Bootstrap (First-Time Setup)

Before CI/CD can deploy, Ansible must bootstrap the Sygnal container and Traefik route:

```bash
# Push an initial image to ECR first (or run the workflow once to populate it)
ansible-playbook -i inventory/staging/hosts setup.yml --tags=setup-sygnal,start
```

## Production Deploys (Manual)

1. **SSH into production**: `ssh ubuntu@52.203.29.202`
2. **Pull latest code**: `cd /path/to/sygnal && git pull origin main`
3. **Rebuild**: `docker build -f docker/Dockerfile -t localhost/pangeachat/sygnal:main .`
4. **Restart**: `sudo systemctl restart matrix-sygnal`
5. **Verify**: `sudo journalctl -fu matrix-sygnal`

### Config-Only Changes

If only Ansible config changes (e.g., FCM apps), re-run the playbook:

```bash
ansible-playbook -i inventory/<env>/hosts setup.yml --tags=setup-sygnal,start
```

## Infrastructure (Terraform)

All staging infrastructure is managed in `devops/terraform/staging/`:

| Resource | Terragrunt path |
|----------|----------------|
| ECR repository | `ecr/sygnal/` |
| DNS CNAME | `dns/sygnal/` |
| OIDC role (ECR push + SSM) | `iam/github-oidc/` |
| EC2 ECR pull policy | `iam/ec2-ecr-pull/` |

## Client Integration

The client sets the push gateway URL in [`client/lib/config/setting_keys.dart`](../../client/lib/config/setting_keys.dart):

```dart
pushNotificationsGatewayUrl: 'https://sygnal.pangea.chat/_matrix/push/v1/notify'
```

The client registers a pusher with Synapse via `POST /_matrix/client/v3/pushers/set`. Synapse forwards push payloads to Sygnal → FCM/APNs.

## Future Work

- Add CI/CD for production (mirror the staging pattern with a `pangea-prod-sygnal` ECR repo)
- Move production DNS CNAME into Terraform
