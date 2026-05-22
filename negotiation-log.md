# Biên bản đàm phán hợp đồng API

- Cặp đàm phán: 05 (IoT ↔ Core Business) & 06 (IoT ↔ Analytics)
- Product: B
- Provider: Nhóm B6 (Core Business) & Nhóm B5 (Analytics)
- Consumer: Nhóm B1 (IoT Ingestion - Đại diện: Nguyễn Kim Đức Vượng)
- Phiên: v1.0
- Ngày: 20/05/2026

---

## Issue #1

- Raised by: Consumer (B1)
- Endpoint: `campus.iot.alerts.v1` (Topic)
- Concern: Tránh lệch múi giờ giữa hệ thống phần cứng IoT và Core Business khi ghi nhận thời điểm xảy ra sự cố.
- Proposal: Thống nhất dùng chuẩn ISO 8601 (UTC) cho tất cả trường timestamp.
- Resolution: Accepted
- Rationale: Giúp hệ thống Core Business dễ dàng parse và đồng bộ mà không cần tự cộng trừ độ lệch múi giờ cục bộ.
- Impact: Consumer áp dụng chuẩn format `date-time` vào toàn bộ payload đẩy lên Queue.

---

## Issue #2

- Raised by: Consumer (B1)
- Endpoint: `campus.iot.alerts.v1` (Topic)
- Concern: Mạng chập chờn có thể khiến IoT vô tình đẩy 1 event cảnh báo (ví dụ: nhiệt độ quá cao) 2 lần, gây spam cảnh báo.
- Proposal: B1 sẽ gắn `event_id` (UUID) cho mỗi event. Nhóm B6 phải dùng ID này làm Idempotency key.
- Resolution: Accepted
- Rationale: Đảm bảo tính luỹ đẳng (Idempotency) cho hệ thống hướng sự kiện, tránh cảnh báo giả cho người dùng cuối.
- Impact: B1 thêm trường bắt buộc `event_id`. B6 thêm logic check ID trùng lặp trong cache (Redis/DB) trước khi xử lý.

---

## Issue #3

- Raised by: Consumer (B1)
- Endpoint: Message Broker Level
- Concern: Message payload vô tình bị lỗi schema khi IoT đẩy vào, dẫn đến kẹt hàng đợi hoặc bị drop mất.
- Proposal: Cấu hình Dead-letter Queue (DLQ) ở phía Broker để hứng các message lỗi, không block các message sau.
- Resolution: Modified
- Rationale: Tạm thời Lab 02 chưa thiết lập Broker thực tế, do đó hai bên thống nhất sẽ thiết kế chi tiết cấu hình DLQ trong đặc tả AsyncAPI ở Lab 03.
- Impact: Không thay đổi payload hiện tại. Đưa thành action item cho Lab 03.

---

## Issue #4

- Raised by: Consumer (B1)
- Endpoint: `campus/iot/telemetry/aggregate` (Topic)
- Concern: Nếu IoT gửi từng bản ghi nhiệt độ đơn lẻ liên tục sẽ làm nghẽn I/O và hệ thống tính toán của Analytics (B5).
- Proposal: Đóng gói dữ liệu thành mảng (Batch) và đẩy định kỳ 1 phút/lần thay vì realtime.
- Resolution: Accepted
- Rationale: Giảm tải áp lực cho hệ thống Analytics, giúp B5 dễ dàng tổng hợp dữ liệu (aggregate) để vẽ biểu đồ hơn.
- Impact: Cấu trúc message đổi từ object đơn thành object chứa mảng `metrics`.

---

## Issue #5

- Raised by: Consumer (B1)
- Endpoint: `campus/iot/telemetry/aggregate` (Topic)
- Concern: Các loại cảm biến khác nhau (nhiệt độ, độ ẩm, khói) có định dạng và đơn vị khác nhau, B5 khó thiết kế database.
- Proposal: Thống nhất cấu trúc mảng `metrics` luôn chứa các key chuẩn: `metric_name` (string), `value` (float), `unit` (string).
- Resolution: Accepted
- Rationale: Thiết kế schema linh hoạt (dynamic structure), hỗ trợ mở rộng thêm các loại cảm biến mới trong tương lai mà không cần sửa hợp đồng sự kiện.
- Impact: B1 code chuẩn hóa data trước khi gom batch. B5 thiết kế DB dựa theo cấu trúc này.

---

## Issue #6

- Raised by: Provider (Analytics - B5)
- Endpoint: `campus/iot/telemetry/aggregate` (Topic)
- Concern: B5 cần tổng hợp nhiệt độ theo từng khu vực/toà nhà, nhưng payload telemetry của IoT hiện tại chỉ cung cấp `device_id`.
- Proposal: B1 (IoT) truy vấn thêm thông tin và đẩy kèm `location_id` vào trong payload trước khi gửi.
- Resolution: Rejected
- Rationale: IoT Ingestion là hệ thống raw (cấp thấp), không lưu trữ meta-data nghiệp vụ như vị trí tòa nhà. Nhóm B5 phải tự fetch dữ liệu từ service Device Management hoặc Core Business để map từ `device_id` ra `location_id`.
- Impact: B1 không cần sửa code. B5 phải chủ động tích hợp thêm một dependency phụ để lấy dữ liệu vị trí.

---

# Chốt hợp đồng v1.0

Provider sign-off: Nhóm B5 & B6  
Consumer sign-off: Nguyễn Kim Đức Vượng (Đại diện B1)  
Witness (GV/TA):    
Date: 20/05/2026               

---

## Ghi chú warning nếu Spectral còn cảnh báo

| Warning | Lý do chấp nhận tạm thời | Kế hoạch sửa |
|---|---|---|
| Thiếu `tags`, `operationId`, `description` ở `/health` và `/core/...` | Nhóm B1 tập trung vào Queue Async (Event-driven), dùng chung file OpenAPI REST mẫu cho mục đích lấy minh chứng Mock Server. Các path này không thuộc phạm vi nghiệp vụ IoT. | Bỏ qua phần REST. Sẽ tập trung chuẩn hóa Event Contract bằng AsyncAPI ở Lab 03. |