# Upstream PR Progress — SUBMITTED

PRs opened 2026-03-02 from `pangeachat/sygnal` (fork of `element-hq/sygnal`).

## PR A: Async `create()` factory fix

- **Branch**: `fix/gcm-async-create`
- **Upstream PR**: https://github.com/element-hq/sygnal/pull/431
- **Status**: Submitted. Tests pass (61/61), codestyle clean, DCO signed.
- **Change**: Moves `aiohttp.ClientSession` + `google_auth_request` creation from `__init__` to async `create()` override in `GcmPushkin`
- **Changelog**: `changelog.d/gcm-async-create.bugfix`

## PR B: Configurable notification content for FCM v1

- **Branch**: `feat/fcm-notification-content`
- **Upstream PR**: https://github.com/element-hq/sygnal/pull/432
- **Status**: Submitted. Tests pass (64/64), codestyle clean, DCO signed. Addresses upstream issue #366.
- **Change**: Adds opt-in `include_notification_content` config option. When enabled, injects `notification.title` (room_name > sender_display_name > sender > "New Message") and `notification.body` (content.body or "New Message") into FCM v1 requests.
- **Changelog**: `changelog.d/gcm-notification-content.feature`

## Repo history

- **pangeachat/sygnal** — current canonical fork (parent: element-hq/sygnal). All branches, upstream PRs, and deployment live here.
- **pangeachat/sygnal-legacy** — archived 2026-03-02. Was the original fork (parent: matrix-org/sygnal, which is archived). Kept for Git history only.
