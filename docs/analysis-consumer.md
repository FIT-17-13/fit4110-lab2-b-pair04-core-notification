# Phân tích yêu cầu — vai Consumer (Notification)

- Cặp đàm phán: Pair 04 — Core Business → Notification
- Product: B
- Consumer service: Notification (B7)
- Producer service: Core Business (Nhóm 12 — B6)
- Người viết: Nhóm B7 & Nhóm 12
- Ngày: 2026-05-19

---

## 1. Event Consumer cần nhận

| Event | Consumer dùng để làm gì? | Field bắt buộc với Consumer |
|---|---|---|
| `alert.created` | Gửi thông báo cảnh báo mới qua Telegram/Email/SMS tùy severity | alertId, alertType, severity, message, createdAt |
| `alert.escalated` | Gửi thông báo leo thang cho cấp quản lý cao hơn | alertId, previousSeverity, escalatedSeverity, reason |
| `alert.resolved` | Gửi thông báo "đã xử lý xong" cho người đã nhận alert trước đó | alertId, resolvedBy, durationSeconds |

---

## 2. Luồng xử lý khi nhận event

| Event | Consumer sẽ làm gì? | Thời gian xử lý dự kiến |
|---|---|---|
| `alert.created` | Tra bảng routing (severity → channels → recipients), compose message, gửi đa kênh | < 5 giây |
| `alert.escalated` | Gửi thông báo cho nhóm quản lý cấp cao hơn (danh sách khác với alert.created) | < 5 giây |
| `alert.resolved` | Gửi thông báo "Alert #X đã được xử lý bởi Y, thời gian xử lý Z phút" | < 5 giây |

---

## 3. Error case Consumer cần xử lý

| Tình huống | Consumer xử lý thế nào? |
|---|---|
| Event trùng lặp (cùng eventId) | Check cache `processed_events`, bỏ qua nếu đã gửi |
| Event thiếu field `severity` | Reject, log `INVALID_SCHEMA`, không gửi thông báo |
| Telegram API trả lỗi 429 (rate limit) | Backoff 10 giây rồi retry, tối đa 3 lần |
| Email server timeout | Retry 3 lần, sau đó log `EMAIL_DELIVERY_FAILED`, tiếp tục gửi kênh khác |
| Alert flood (> 10 event trong 30 giây) | Gom lại thành 1 digest message tổng hợp |

---

## 4. Giả định bổ sung

- Giả định 1: Notification tự quản lý bảng routing (ai nhận, qua kênh nào) dựa trên severity. Core Business không cần biết chi tiết này.
- Giả định 2: Mỗi event phải self-contained — Notification không cần gọi lại API Core Business để lấy thêm thông tin.
- Giả định 3: Notification không gửi response/ACK nghiệp vụ về Core Business (fire-and-forget từ góc Core).

---

## 5. Câu hỏi cho Producer

1. Nội dung `message` trong event có support markdown (cho Telegram) hay chỉ plain text?
2. Core Business có giới hạn số ký tự cho `message` không? Notification cần biết để thiết kế template.
3. Khi alert được escalate, `severity` có thay đổi trong DB không hay chỉ thay đổi trong event?

---

## 6. Rủi ro tích hợp

| Rủi ro | Tác động | Đề xuất xử lý |
|---|---|---|
| Notification down > 6 giờ | Mất event do hết retention | Monitoring + auto-restart, alert khi consumer lag tăng |
| Core gửi severity không hợp lệ | Notification không biết gửi kênh nào | Validate enum, default gửi Email nếu severity unknown |
| Người nhận chặn bot Telegram | Thông báo không đến | Log delivery failure, fallback sang email |
