# sygnal — Pangea fork of Element's Matrix push gateway

Fork of [element-hq/sygnal](https://github.com/element-hq/sygnal). Adds visible FCM notification content for iOS (upstream sends data-only, which iOS throttles/drops).

- **Upstream**: element-hq/sygnal v0.17.0 (Nov 2025)
- **Fork base**: v0.15.1 — **24 commits behind**, needs rebase
- **License concern**: Upstream switched to AGPL-3.0 + Element Commercial after our fork point. Our copy still has Apache 2.0.
- **Our change**: Single PR #1 — injects `notification.title` and `notification.body` into FCM v1 requests (see [fork-changes.md](docs/fork-changes.md))
- **Deployed**: Production only (`localhost/pangeachat/sygnal:main`, built on EC2). Staging has no Sygnal yet.
- **Firebase**: Project `pangea-chat-936ee`, apps `com.talktolearn.chat` + `com.talktolearn.chat.data_message`
- **Config**: Managed by ansible repo (`pangeachat/synapse`), not this repo

| [deployment.instructions.md](instructions/deployment.instructions.md) | Build, deploy, Ansible config, client integration, what's non-standard |

## Open work
- Rebase onto upstream v0.17.0 (resolve `send_badge_counts` conflict, adopt AGPL)
- Deploy to staging for parity
- Evaluate whether upstream #366 fix makes our patch obsolete
