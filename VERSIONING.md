# Versioning — Event Contract: Core Business ↔ Notification

## Phiên bản hiện tại

| Phiên bản | Ngày | Trạng thái | Ghi chú |
|---|---|---|---|
| v1.0.0 | 2026-05-19 | Signed-off | Hợp đồng sự kiện sơ bộ giữa Core Business (B6) và Notification (B7) |

## Quy tắc versioning

Dùng **Semantic Versioning** (SemVer):

- **MAJOR** (x.0.0): Breaking change (VD: xóa field bắt buộc, đổi tên event)
- **MINOR** (1.x.0): Feature mới tương thích ngược (VD: thêm event mới, thêm field tùy chọn)
- **PATCH** (1.0.x): Sửa lỗi, cập nhật description

## Lịch sử thay đổi

### v1.0.0 — 2026-05-19

- Khởi tạo Event Contract sơ bộ cho pair-04 (Core Business → Notification)
- Events: `alert.created`, `alert.escalated`, `alert.resolved`
- Payload: JSON chuẩn CloudEvents
- Đàm phán 7 issues (severity, channel routing, downstream retry, idempotency, ordering, retention, alert flood)
- Đã sign-off Producer (B6) và Consumer (B7)
