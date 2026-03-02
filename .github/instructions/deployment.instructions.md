---
applyTo: "**/.github/workflows/**,**/deploy*,**/docker/**"
---

# Sygnal Deployment

How the Sygnal push gateway is built and deployed. For org-wide deployment patterns, see [deployment.instructions.md](../../.github/.github/instructions/deployment.instructions.md).

## Architecture

Sygnal runs as a **Docker container on the production Synapse EC2 instance** (`matrix.pangea.chat` / `52.203.29.202`), managed by the same [Ansible playbook](../../ansible/) that deploys Synapse. It is **not** deployed via ECS/Fargate like the choreographer and CMS.

| Component | Detail |
|-----------|--------|
| **Docker image** | `localhost/pangeachat/sygnal:main` — built directly on the EC2 instance |
| **Ansible role** | `roles/custom/matrix-sygnal` in the ansible repo |
| **Config** | [`ansible/inventory/production/host_vars/matrix.pangea.chat/vars.yml`](../../ansible/inventory/production/host_vars/matrix.pangea.chat/vars.yml) |
| **Reverse proxy** | Traefik (managed by the Ansible playbook) — routes `sygnal.pangea.chat` → container port 6000 |
| **DNS** | CNAME `sygnal.pangea.chat` → `matrix.pangea.chat` (Route 53, created manually — not in Terraform) |
| **Systemd service** | `matrix-sygnal` (template: `matrix-sygnal.service.j2`) |
| **Staging** | **Not deployed.** No sygnal config in staging inventory. |

## Deploying a Code Change

There is no CI/CD pipeline for Sygnal deploys — it's a manual process on the production EC2 instance.

### Steps

1. **SSH into production**: `ssh ubuntu@52.203.29.202` (or `ssh ubuntu@matrix.pangea.chat`)
2. **Navigate to the sygnal clone** (wherever it was previously cloned on the instance)
3. **Pull the latest code**: `git pull origin main`
4. **Rebuild the Docker image**:
   ```bash
   docker build -f docker/Dockerfile -t localhost/pangeachat/sygnal:main .
   ```
5. **Restart the service**: `sudo systemctl restart matrix-sygnal`
6. **Verify**: `sudo journalctl -fu matrix-sygnal` and test a push notification from the app

### Config-Only Changes (Ansible Vars)

If only the Ansible config changes (e.g., adding an FCM app, updating `matrix_sygnal_apps`), re-run the playbook instead:

```bash
ansible-playbook -i inventory/production/hosts setup.yml --tags=setup-sygnal,start
```

This regenerates the Sygnal YAML config from the Jinja template and restarts the container.

## Key Ansible Config

Defined in the production vars file:

- `matrix_sygnal_enabled: true`
- `matrix_sygnal_docker_image: "localhost/pangeachat/sygnal:main"` — overrides the default `matrixdotorg/sygnal` image
- `matrix_sygnal_docker_image_force_pull: false` — critical, since the image is local (not from a registry)
- `matrix_sygnal_apps` — defines two FCM v1 apps: `com.talktolearn.chat` and `com.talktolearn.chat.data_message`
- Firebase service account JSON is deployed via `aux_file_definitions` into the Sygnal data path

## Client Integration

The client sets the push gateway URL in [`client/lib/config/setting_keys.dart`](../../client/lib/config/setting_keys.dart):

```dart
pushNotificationsGatewayUrl: 'https://sygnal.pangea.chat/_matrix/push/v1/notify'
```

The client registers a pusher with Synapse via `POST /_matrix/client/v3/pushers/set`, passing this URL. Synapse then sends push payloads to Sygnal, which forwards them to FCM/APNs. This is the standard Matrix push flow — Sygnal is **not** configured in Synapse's server config.

## What's Non-Standard

1. **Image built on-instance** — Most Ansible-managed services pull pre-built images from a registry. We build `localhost/pangeachat/sygnal:main` directly on the EC2 instance because the Ansible role doesn't support `self_build` and we need our fork's changes.
2. **No staging deployment** — Push notifications only work in production. Adding staging parity is tracked as open work.
3. **No CI/CD deploy pipeline** — Deploys are fully manual SSH + docker build + systemd restart.
4. **DNS not in Terraform** — The `sygnal.pangea.chat` CNAME was created manually in Route 53.

## Future Work

- Deploy to staging for parity
- Set up a CI/CD pipeline (GitHub Actions → ECR or on-instance build trigger)
- Move DNS into Terraform
