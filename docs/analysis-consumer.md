# Phân tích yêu cầu — vai Consumer (Nhóm B1)

*Lưu ý: Trong hệ thống Smart Campus, nhóm B1 (IoT Ingestion) đóng vai trò Consumer (Publisher đẩy dữ liệu) cho 2 dependency là Cặp 05 và Cặp 06 thông qua cơ chế Queue Async.*

---

## PHẦN 1: CẶP ĐÀM PHÁN 05 (IoT Ingestion ↔ Core Business)

- Cặp đàm phán: 05
- Product: B
- Consumer service: IoT Ingestion (Nhóm B1 - Đóng vai trò Publisher)
- Provider service: Core Business (Nhóm B6 - Đóng vai trò Subscriber)
- Người viết: Nguyễn Kim Đức Vượng
- Ngày: 20/05/2026

### 1. Resource (Event/Message) Consumer cần gửi

| Resource | Consumer dùng để làm gì? | Field bắt buộc với Consumer | Field có thể tùy chọn |
|---|---|---|---|
| `SensorTelemetry` | Đẩy dữ liệu đo lường định kỳ từ cảm biến (nhiệt độ, độ ẩm). | `event_id`, `device_id`, `sensor_type`, `value`, `timestamp` | `battery_level`, `signal_strength` |
| `SensorAlert` | Gửi cảnh báo khẩn cấp khi cảm biến vượt ngưỡng an toàn. | `event_id`, `device_id`, `alert_type`, `severity`, `timestamp` | `raw_value_context` |

### 2. API (Topic/Queue) Consumer cần gọi

| Method | Path (Topic) | Lúc nào gọi? | Kỳ vọng từ Broker |
|---|---|---|---|
| PUBLISH | `campus.iot.telemetry.v1` | Định kỳ (ví dụ 5s/lần) khi cảm biến gửi dữ liệu về. | Nhận được ACK từ Broker (lưu hàng đợi thành công). |
| PUBLISH | `campus.iot.alerts.v1` | Ngay lập tức khi phát hiện bất thường. | Nhận được ACK từ Broker (lưu hàng đợi thành công). |

### 3. Error case Consumer cần xử lý

| Status | Consumer hiểu là gì? | Consumer sẽ xử lý thế nào? |
|---|---|---|
| 400 | Payload sai schema | Ghi log lỗi cấu trúc, không gửi lại để tránh loop. |
| 401 | Thiếu credential Broker | Cảnh báo hệ thống, kiểm tra lại tài khoản kết nối. |
| 404 | Không tìm thấy Topic | Tự động tạo topic hoặc báo lỗi cấu hình. |
| 408 | Mất kết nối mạng | Lưu dữ liệu tạm vào buffer cục bộ, retry khi có mạng. |
| 409 | Gửi trùng Event (Duplicate) | Không cần xử lý, Core Business tự lọc trùng bằng `event_id`. |
| 503 | Core Business Down | Không ảnh hưởng, dữ liệu nằm chờ an toàn trong Queue. |

### 4. Giả định bổ sung

- Giả định 1: Dữ liệu cảm biến gửi đi dạng JSON, timestamp chuẩn ISO 8601 (VD: 2026-05-20T10:00:00Z).
- Giả định 2: Core Business (B6) phải áp dụng cơ chế Idempotency dựa trên trường `event_id` để tránh cảnh báo sai nếu IoT lỡ đẩy trùng message.
- Giả định 3: Với event `SensorAlert`, message queue phải cấu hình độ bền (durable) để không mất dữ liệu cảnh báo quan trọng.

### 5. Câu hỏi cho Provider

1. Giới hạn chịu tải của Core Business là bao nhiêu message/giây để bên IoT cấu hình Rate Limiting?
2. Core Business có cần nhận toàn bộ lịch sử dữ liệu nếu IoT offline một thời gian rồi đẩy bù lên, hay chỉ quan tâm dữ liệu realtime mới nhất?
3. Core Business có định dùng Dead-Letter Queue (DLQ) để lưu các alert xử lý lỗi không?

### 6. Rủi ro tích hợp

| Rủi ro | Tác động | Đề xuất xử lý |
|---|---|---|
| Hai bên lệch kiểu dữ liệu (IoT gửi string, Core mong float) | Core Business parse lỗi, crash app. | Chốt chính xác data type payload trong Lab 03. |
| Nghẽn cổ chai khi ngàn cảm biến cùng đẩy dữ liệu | Message nghẽn ở Queue, trễ cảnh báo. | Core Business nên thiết kế cơ chế xử lý batch hoặc scale worker. |

---
---

## PHẦN 2: CẶP ĐÀM PHÁN 06 (IoT Ingestion ↔ Analytics)

- Cặp đàm phán: 06
- Product: B
- Consumer service: IoT Ingestion (Nhóm B1 - Đóng vai trò Publisher)
- Provider service: Analytics (Nhóm B5 - Đóng vai trò Subscriber)
- Người viết: Nguyễn Kim Đức Vượng
- Ngày: 20/05/2026

### 1. Resource (Event/Message) Consumer cần gửi

| Resource | Consumer dùng để làm gì? | Field bắt buộc với Consumer | Field có thể tùy chọn |
|---|---|---|---|
| `TelemetryBatch` | Gửi gói dữ liệu đo lường theo cụm (nhiệt độ, độ ẩm) để Analytics phân tích và vẽ biểu đồ. | `batch_id`, `device_id`, `timestamp`, `metrics` (mảng các giá trị) | `firmware_version`, `location_id` |
| `DeviceStatusEvent` | Thông báo trạng thái online/offline của cảm biến để Analytics tính toán tỷ lệ uptime. | `event_id`, `device_id`, `status`, `timestamp` | `battery_level` |

### 2. API (Topic/Queue) Consumer cần gọi

| Method | Path (Topic) | Lúc nào gọi? | Kỳ vọng response |
|---|---|---|---|
| PUBLISH | `campus/iot/telemetry/aggregate` | Định kỳ 1-5 phút/lần (gom thành batch gửi 1 lần). | Broker trả về ACK (Lưu hàng đợi thành công). |
| PUBLISH | `campus/iot/device/status` | Ngay khi cảm biến bị ngắt kết nối hoặc kết nối lại. | Broker trả về ACK (Lưu hàng đợi thành công). |

### 3. Error case Consumer cần xử lý

| Status | Consumer hiểu là gì? | Consumer sẽ xử lý thế nào? |
|---:|---|---|
| 400 | Payload sai schema (Schema Registry reject) | Ghi log lỗi cấu trúc, không gửi lại để tránh vòng lặp lỗi. |
| 401 | Thiếu credentials kết nối Broker | Cảnh báo hệ thống, kiểm tra lại tài khoản/mật khẩu RabbitMQ/Kafka. |
| 403 | Không có quyền Publish vào topic này | Liên hệ admin hệ thống để cấp quyền Publish cho tài khoản IoT. |
| 404 | Không tìm thấy Topic/Queue đích | Cấu hình tự động tạo Topic (auto-create) hoặc báo lỗi cấu hình. |
| 408 | Mất kết nối mạng (Timeout) | Lưu dữ liệu tạm thời vào ổ cứng (buffer), tự động retry khi có mạng. |
| 422 | Vi phạm rule (vd: timestamp ở tương lai) | Drop message và ghi log cảnh báo cảm biến bị trôi giờ. |

### 4. Giả định bổ sung

- Giả định 1: IoT Ingestion sẽ ưu tiên gửi dữ liệu theo dạng **Batch** (gộp nhiều bản ghi) thay vì gửi lẻ tẻ từng dòng để giảm tải cho hệ thống Analytics.
- Giả định 2: Giờ giấc (Timestamp) trong tất cả payload đều dùng chuẩn UTC (ISO 8601) để Analytics dễ tính toán, không bị sai lệch múi giờ.
- Giả định 3: Các dữ liệu rác (như cảm biến lỗi trả về `NaN` hoặc giá trị ảo) sẽ được IoT tự động lọc bỏ (filter) trước khi đẩy sang Analytics.

### 5. Câu hỏi cho Provider

1. Nhóm Analytics (B5) muốn nhận batch dữ liệu bao nhiêu records một lần, hoặc bao nhiêu phút một lần là tối ưu cho việc xử lý?
2. Nếu mạng chập chờn, dữ liệu của buổi sáng đến chiều mới được IoT đẩy lên (Late Arrival Data), Analytics có cơ chế nào để cập nhật lại biểu đồ thống kê trong quá khứ không?
3. Analytics có cần IoT Ingestion gửi kèm ID khu vực (vị trí tòa nhà/phòng) không, hay hệ thống của B5 sẽ tự map dựa trên `device_id`?

### 6. Rủi ro tích hợp

| Rủi ro | Tác động | Đề xuất xử lý |
|---|---|---|
| Thay đổi cấu trúc mảng `metrics` | Analytics parse lỗi, biểu đồ không lên số. | Chốt cứng cấu trúc JSON Schema (Event Contract) trên Lab 03. |
| IoT đẩy dữ liệu ồ ạt sau khi có mạng lại (Spike load) | Lag hệ thống xử lý của Analytics, tràn RAM. | Thống nhất tốc độ tiêu thụ (Rate Limit) và Analytics nên kéo dữ liệu từ tốn. |
| Sai lệch định dạng thời gian | Thống kê lệch ngày, không hiển thị đúng xu hướng. | Bắt buộc validate chuẩn ISO 8601 ở bước đẩy vào Queue. |