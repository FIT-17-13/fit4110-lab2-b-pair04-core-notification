# Phân tích yêu cầu — vai Producer (Core Business)

- Cặp đàm phán: Pair 04 — Core Business → Notification
- Product: B
- Producer service: Core Business (Nhóm 12 — B6)
- Consumer service: Notification (B7)
- Người viết: Nhóm 12
- Ngày: 2026-05-19

---

## 1. Event chính Producer sẽ phát

| Event | Mô tả | Trigger | Tần suất |
|---|---|---|---|
| `alert.created` | Cảnh báo mới được tạo | Sensor vượt ngưỡng, truy cập trái phép, lỗi hệ thống | Trung bình (vài lần/giờ, burst khi sự cố) |
| `alert.escalated` | Cảnh báo chưa xử lý bị leo thang | Alert severity >= HIGH chưa acknowledge sau 15 phút | Thấp |
| `alert.resolved` | Cảnh báo đã được đóng | Admin hoặc hệ thống tự động resolve | Tương đương alert.created |

---

## 2. Payload field dự kiến

| Field | Kiểu | Bắt buộc | Mô tả |
|---|---|---|---|
| `alertId` | string | ✅ | Mã cảnh báo, dùng làm correlationId |
| `alertType` | string (enum) | ✅ (created) | SENSOR_THRESHOLD_EXCEEDED, UNAUTHORIZED_ACCESS, UNKNOWN_PERSON, SYSTEM_ERROR |
| `severity` | string (enum) | ✅ | LOW, MEDIUM, HIGH, CRITICAL |
| `message` | string | ✅ (created) | Nội dung cảnh báo để hiển thị cho người nhận |
| `locationId` | string | Tùy chọn | Vị trí xảy ra sự kiện |
| `resolvedBy` | string | ✅ (resolved) | Người/hệ thống đóng alert |
| `durationSeconds` | integer | ✅ (resolved) | Thời gian xử lý |
| `resolutionNote` | string | Tùy chọn | Ghi chú khi đóng |
| `previousSeverity` | string | ✅ (escalated) | Severity trước khi leo thang |
| `escalatedSeverity` | string | ✅ (escalated) | Severity sau leo thang |

---

## 3. Giả định bổ sung

- Giả định 1: Core Business chỉ phát event cho alert có severity >= MEDIUM. Alert LOW chỉ ghi log nội bộ, không cần thông báo.
- Giả định 2: Core Business không quan tâm Notification gửi qua kênh nào. Trách nhiệm chọn kênh thuộc về Notification service.
- Giả định 3: Mỗi event chứa đủ thông tin để Notification compose thông báo hoàn chỉnh (self-contained), không cần callback lại Core.

---

## 4. Câu hỏi cho Consumer

1. Notification có cần Core Business gửi danh sách người nhận, hay tự quản lý bảng routing?
2. Khi gửi thông báo thất bại (VD: Telegram API lỗi), Notification có cần báo lại cho Core không?
3. Notification có giới hạn rate limit cho mỗi kênh không? (VD: tối đa 30 tin Telegram/phút)

---

## 5. Rủi ro tích hợp

| Rủi ro | Tác động | Đề xuất xử lý |
|---|---|---|
| Alert flood (mất điện campus) | Spam inbox người nhận | Notification implement digest/batch 30 giây |
| Notification down kéo dài | Thông báo bị mất (hết retention) | Monitoring consumer lag, alert khi lag > 1 giờ |
| Message quá lớn (> 64KB) | Queue reject | Core Business giới hạn `message` tối đa 500 ký tự |
