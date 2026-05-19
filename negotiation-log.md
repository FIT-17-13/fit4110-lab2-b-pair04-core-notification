# Biên bản đàm phán hợp đồng Event

- Cặp đàm phán: Core Business ↔ Notification (Pair 04)
- Product: B
- Producer: Core Business (Nhóm 12 — B6)
- Consumer: Notification (B7)
- Phiên: v1.0
- Ngày: 2026-05-19

---

## Issue #1 — Severity có cần gửi kèm trong event không?

- Raised by: Consumer (Notification)
- Event: `alert.created`
- Concern: Notification cần biết mức độ nghiêm trọng để quyết định gửi qua kênh nào (Telegram, email, SMS). Nếu thiếu severity, Notification phải gọi lại API Core Business để lấy thông tin — mất tính async.
- Proposal: Core Business luôn đính kèm trường `severity` (enum: LOW, MEDIUM, HIGH, CRITICAL) trong mọi event alert.
- Resolution: Accepted
- Rationale: Notification cần quyết định kênh gửi ngay khi nhận event. Nếu phải callback lại Core sẽ phá vỡ nguyên tắc loose coupling của kiến trúc event-driven.
- Impact: Core Business bắt buộc gửi `severity` trong payload. Notification map: MEDIUM → Email, HIGH → Telegram + Email, CRITICAL → Telegram + Email + SMS.

---

## Issue #2 — Notification tự định tuyến kênh hay Core gửi danh sách channel?

- Raised by: Consumer (Notification)
- Event: `alert.created`, `alert.escalated`
- Concern: Ai chịu trách nhiệm quyết định gửi qua kênh nào? Nếu Core Business chỉ định channel, Notification chỉ là "loa phát thanh" — nhưng nếu Notification tự quyết thì cần biết severity và cấu hình nội bộ.
- Proposal: Notification tự định tuyến kênh dựa trên `severity` và bảng cấu hình nội bộ (ai nhận, qua kênh nào). Core Business không cần quan tâm kênh gửi.
- Resolution: Accepted
- Rationale: Tách biệt trách nhiệm (SoC): Core Business lo nghiệp vụ, Notification lo delivery. Khi cần thêm kênh mới (VD: Zalo), chỉ cần sửa Notification mà không động vào Core.
- Impact: Event payload không có field `channels`. Notification duy trì bảng mapping `severity → channels` riêng.

---

## Issue #3 — Xử lý khi downstream (Telegram/Email) lỗi

- Raised by: Consumer (Notification)
- Event: Tất cả events
- Concern: Nếu Telegram API hoặc Email server bị lỗi tạm thời, ai chịu trách nhiệm retry? Nếu retry ở tầng queue sẽ gây duplicate message trên Telegram.
- Proposal: Notification tự retry gửi ở tầng application (3 lần, exponential backoff). Queue chỉ retry khi Notification consumer crash (message không ACK).
- Resolution: Accepted
- Rationale: Tách retry thành 2 tầng: (1) Queue retry khi consumer crash, (2) Application retry khi downstream lỗi. Tránh gửi tin nhắn trùng.
- Impact: Notification cần lưu trạng thái gửi cho mỗi eventId + channel. Core Business không cần quan tâm retry logic.

---

## Issue #4 — Idempotency — tránh gửi thông báo trùng

- Raised by: Producer (Core Business)
- Event: Tất cả events
- Concern: At-least-once delivery có thể khiến Notification nhận cùng event 2 lần. Nếu gửi 2 tin Telegram cho cùng 1 alert sẽ gây khó chịu cho người nhận.
- Proposal: Mỗi event bắt buộc có `eventId` (UUID v4). Notification check `eventId` đã xử lý chưa trước khi gửi thông báo.
- Resolution: Accepted
- Rationale: Pattern idempotent consumer là bắt buộc cho hệ thống phân tán. Notification lưu `eventId` đã gửi thành công trong cache (TTL 6 giờ, bằng retention).
- Impact: Core Business sinh UUID cho mỗi event. Notification thêm bảng/cache `processed_events`.

---

## Issue #5 — Thứ tự event (Ordering) và xử lý out-of-order

- Raised by: Consumer (Notification)
- Event: `alert.created`, `alert.escalated`, `alert.resolved`
- Concern: Queue có thể giao `alert.escalated` trước `alert.created`. Notification cần gửi thông báo escalation nhưng chưa có context ban đầu (message, alertType). Hoặc nhận `alert.resolved` trước `alert.created`, dẫn đến gửi "đã xử lý" khi người nhận chưa biết có alert.
- Proposal: Notification xử lý mỗi event độc lập: (1) `alert.created` → gửi thông báo mới, (2) `alert.escalated` → gửi thông báo leo thang kèm dữ liệu có sẵn trong event, (3) `alert.resolved` → gửi thông báo đóng. Mỗi event phải tự chứa đủ thông tin để gửi thông báo mà không cần tra cứu event khác.
- Resolution: Accepted
- Rationale: Thiết kế "self-contained event" giúp Notification không cần buffer hay tra cứu state, xử lý từng event đơn giản và hiệu quả.
- Impact: Core Business đảm bảo mỗi event chứa đủ `alertId`, `message` (hoặc `reason`), `severity` để Notification compose thông báo hoàn chỉnh.

---

## Issue #6 — Retention và thời gian sống của message

- Raised by: Cả hai bên
- Event: Tất cả events
- Concern: Thông báo cảnh báo có tính thời sự cao. Nếu Notification bị down 24 giờ, khi quay lại nhận được hàng trăm event cũ thì có nên gửi thông báo không?
- Proposal: Message retention 6 giờ. Event cũ hơn 6 giờ tự động bị xóa khỏi queue vì mất tính kịp thời.
- Resolution: Accepted
- Rationale: Cảnh báo an ninh cần gửi trong vài phút. Nếu sau 6 giờ mới gửi thì đã quá trễ để hành động, chỉ gây spam cho người nhận.
- Impact: Cấu hình TTL = 6h trên broker. Cần monitoring consumer lag để phát hiện sớm khi Notification bị chậm.

---

## Issue #7 — Xử lý alert flood (nhiều alert cùng lúc)

- Raised by: Consumer (Notification)
- Event: `alert.created`
- Concern: Khi xảy ra sự cố lớn (VD: mất điện toàn campus), Core Business có thể phát hàng chục event `alert.created` trong vài giây. Nếu Notification gửi từng tin riêng lẻ sẽ spam inbox người nhận.
- Proposal: Notification implement cơ chế digest/batching: gom các alert cùng severity trong cửa sổ 30 giây và gửi 1 tin tổng hợp thay vì nhiều tin riêng.
- Resolution: Accepted
- Rationale: Digest giảm spam, người nhận dễ đọc hơn. Vẫn đảm bảo thông báo kịp thời (chậm tối đa 30 giây).
- Impact: Notification implement sliding window 30 giây. Core Business không cần thay đổi gì.

---

# Chốt hợp đồng sơ bộ v1.0

Producer sign-off: Nhóm 12 — Core Business (B6)
Consumer sign-off: Notification (B7)
Witness (GV/TA):   _________________
Date:              2026-05-19
