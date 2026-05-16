# Phân tích yêu cầu — vai Consumer

- Cặp đàm phán: pair-01
- Product: Camera Stream (A) / AI Vision (B)
- Consumer service: Camera Stream Service
- Provider service: AI Vision Service
- Người viết: Nhóm 14 – Consumer
- Ngày: 2026-05-17

---

## 1. Resource Consumer cần nhận/gửi

| Resource | Consumer dùng để làm gì? | Field bắt buộc với Consumer | Field có thể tùy chọn |
|---|---|---|---|
| `DetectRequest` | Gửi ảnh lên AI Vision khi phát hiện motion | cameraId, imageSource, correlationId, timestamp | — |
| `DetectResponse` | Nhận kết quả phân tích để chuyển tiếp cho Core Business | detectionId, objects, riskLevel, processedAt | note, boundingBox |
| `DetectedObject` | Hiển thị object bị phát hiện trong ảnh | label, confidence | boundingBox |
| `ModelInfo` | Kiểm tra model AI đang dùng có hỗ trợ loại object cần detect không | modelId, supportedLabels | version, lastUpdated |
| `HealthStatus` | Kiểm tra AI Vision còn sống trước khi gửi frame | status | service, time |

---

## 2. API Consumer cần gọi

| Method | Path | Lúc nào gọi? | Kỳ vọng response |
|---|---|---|---|
| GET | `/health` | Trước khi bắt đầu gửi frame hoặc định kỳ health check | 200 với status: ok |
| POST | `/vision/detect` | Ngay khi Camera phát hiện motion | 200 với detectionId, objects, riskLevel |
| GET | `/vision/detections/{detectionId}` | Khi cần tra cứu lại kết quả cũ để đối chiếu | 200 với đầy đủ thông tin detection |
| GET | `/vision/models/info` | Khi khởi động service để kiểm tra model tương thích | 200 với danh sách supportedLabels |

---

## 3. Error case Consumer cần xử lý

| Status | Consumer hiểu là gì? | Consumer sẽ xử lý thế nào? |
|---:|---|---|
| 400 | Request sai schema, thiếu field bắt buộc | Log lỗi chi tiết từ errors[], không retry |
| 401 | Thiếu hoặc hết hạn Bearer token | Tự động refresh token rồi retry 1 lần |
| 404 | detectionId không tồn tại | Hiển thị trạng thái không tìm thấy, không retry |
| 422 | cameraId sai pattern hoặc mimeType không hỗ trợ | Log lỗi nghiệp vụ, báo admin kiểm tra cấu hình camera |
| 500 | Lỗi nội bộ phía AI Vision | Retry sau 3 giây tối đa 3 lần, nếu vẫn lỗi thì alert Core Business |

---

## 4. Giả định bổ sung

- Giả định 1: AI Vision xử lý đồng bộ và trả kết quả ngay, Consumer không cần implement polling logic.
- Giả định 2: Consumer tự sinh correlationId dạng UUID trước khi gửi request để phục vụ trace log.
- Giả định 3: Khi riskLevel = HIGH hoặc CRITICAL, Consumer tự động forward detectionId sang Core Business mà không cần thêm bước xác nhận.
- Giả định 4: Consumer ưu tiên gửi ảnh dạng URL nếu AI Vision và Camera cùng mạng nội bộ, chỉ dùng base64 khi không share được storage.

---

## 5. Câu hỏi cho Provider

1. Thời gian xử lý tối đa cho một request POST /vision/detect là bao nhiêu giây? Consumer cần biết để set timeout hợp lý.
2. Khi objects trả về rỗng, riskLevel có luôn là LOW không hay có thể là giá trị khác?
3. Provider có hỗ trợ batch detect nhiều frame trong một request không, hay phải gọi từng request riêng lẻ?

---

## 6. Rủi ro tích hợp

| Rủi ro | Tác động | Đề xuất xử lý |
|---|---|---|
| Provider đổi kiểu dữ liệu của riskLevel | Consumer parse lỗi hoặc logic sai | Chốt enum rõ ràng trong openapi.yaml |
| Provider thiếu mã lỗi cụ thể | Consumer khó phân biệt lỗi để xử lý đúng | Chuẩn hóa Problem Details với errors[] |
| Độ trễ AI Vision cao khi tải lớn | Camera bị block chờ response | Thống nhất timeout và cơ chế retry |
| URL ảnh không accessible từ AI Vision | 422 hoặc 500 không rõ nguyên nhân | Quy định rõ URL phải trong campus network |
| Consumer gửi frame liên tục gây quá tải | AI Vision trả 500 hàng loạt | Provider cần có rate limiting, Consumer cần throttle |