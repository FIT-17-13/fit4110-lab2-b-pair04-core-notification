# Checklist Lab 02 — Queue Async (Rubric B) — Pair 04

- [x] Đã xác định đúng cặp đàm phán (Core Business ↔ Notification).
- [x] Đã đọc đúng user story `user-stories/pair-04-core-notification-async.md`.
- [x] Đã hoàn thành `docs/event-contract-template.md` với đầy đủ Producer, Consumer, Event Name, Topic, Trigger, Điều kiện phát.
- [x] Đã viết payload schema sơ bộ kèm 3 ví dụ JSON (alert.created, alert.escalated, alert.resolved).
- [x] Nêu rõ `eventId` (UUID v4) và `correlationId` (alertId).
- [x] Đã xét 6 edge cases (duplicate, out-of-order, missing field, invalid schema, downstream failure, alert flood).
- [x] `negotiation-log.md` có đủ 7 issues (severity, channel routing, downstream retry, idempotency, ordering, retention, alert flood).
- [x] Mỗi issue có đầy đủ: concern, proposal, resolution, rationale, impact.
- [x] Có sign-off Producer (B6), Consumer (B7), Witness.
- [x] Đã chốt hướng chuyển tiếp sang Lab 03 (AsyncAPI, Message Broker, Dead-letter queue, digest).
- [x] Đã cập nhật `VERSIONING.md` cho phiên bản event contract.
- [x] Provider analysis (`docs/analysis-provider.md`) đã điền đầy đủ.
- [x] Consumer analysis (`docs/analysis-consumer.md`) đã điền đầy đủ.
