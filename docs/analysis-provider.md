# Phân tích yêu cầu — vai Provider

- Cặp đàm phán: pair-01
- Product: Camera Stream (A) / AI Vision (B)
- Provider service: AI Vision Service
- Consumer service: Camera Stream Service
- Người viết: Nhóm 14 – Provider
- Ngày: 2026-05-17

---

## 1. Resource chính

| Resource | Mô tả | Thuộc tính bắt buộc | Thuộc tính tùy chọn |
|---|---|---|---|
| `DetectRequest` | Yêu cầu phân tích ảnh từ Camera | cameraId, imageSource, correlationId, timestamp | — |
| `DetectResponse` | Kết quả phân tích ảnh từ AI Vision | detectionId, cameraId, objects, riskLevel, processedAt | note |
| `DetectedObject` | Một object được phát hiện trong ảnh | label, confidence | boundingBox |
| `ModelInfo` | Thông tin model AI đang chạy | modelId, version, supportedLabels, lastUpdated | — |
| `HealthStatus` | Trạng thái service | status, service, time | — |

---

## 2. Action/API dự kiến

| Method | Path | Mục đích | Consumer gọi khi nào? |
|---|---|---|---|
| GET | `/health` | Kiểm tra service còn sống không | Trước khi gửi batch frame hoặc khi monitor |
| POST | `/vision/detect` | Gửi ảnh để AI phân tích | Ngay khi Camera phát hiện motion |
| GET | `/vision/detections/{detectionId}` | Tra cứu kết quả detection cũ | Khi cần lấy lại kết quả đã xử lý |
| GET | `/vision/models/info` | Lấy thông tin model AI | Khi Consumer muốn biết model đang dùng |

---

## 3. Error case

| Status | Tình huống | Response body dự kiến |
|---:|---|---|
| 400 | Payload sai định dạng JSON hoặc thiếu trường bắt buộc | `Problem` với errors[] chỉ rõ field lỗi |
| 401 | Thiếu hoặc sai Bearer token | `Problem` |
| 404 | detectionId không tồn tại trong hệ thống | `Problem` |
| 422 | cameraId không đúng pattern CAM-NNN hoặc mimeType không hỗ trợ | `Problem` |
| 500 | Lỗi nội bộ AI model hoặc downstream service | `Problem` |

---

## 4. Giả định bổ sung

- Giả định 1: AI Vision xử lý đồng bộ, trả kết quả ngay trong response của POST /vision/detect, không cần polling.
- Giả định 2: Ảnh base64 giới hạn tối đa 5MB. Nếu vượt quá, trả 422 với detail rõ ràng.
- Giả định 3: correlationId do Consumer tự sinh, Provider chỉ log lại để trace, không cần unique constraint phía server.
- Giả định 4: riskLevel = LOW khi objects trả về rỗng, không trả null để Consumer không cần null-check.

---

## 5. Câu hỏi cho Consumer

1. Camera có thể gửi nhiều frame liên tiếp trong thời gian ngắn không? Provider cần biết để thiết kế rate limiting.
2. Consumer có cần retry tự động khi nhận 500 không? Nếu có thì Provider cần đảm bảo idempotency cho POST /vision/detect.
3. Consumer xử lý riskLevel = CRITICAL như thế nào? Có cần Provider gửi thêm thông tin bổ sung không?

---

## 6. Rủi ro tích hợp

| Rủi ro | Tác động | Đề xuất xử lý |
|---|---|---|
| Tên field không thống nhất | Consumer parse lỗi hoặc bỏ sót dữ liệu | Chốt toàn bộ naming trong openapi.yaml, không dùng alias |
| Payload ảnh base64 quá lớn | Timeout hoặc mock server lỗi | Thống nhất giới hạn 5MB, ghi rõ trong description |
| Consumer không gửi correlationId đúng format | Khó trace log khi debug | Bắt buộc correlationId trong required, validate pattern |
| riskLevel trả null khi không detect được | Consumer bị null pointer exception | Quy định riskLevel luôn có giá trị, mặc định LOW |
| Retry gây xử lý trùng lặp | AI chạy detect nhiều lần cùng frame | Khuyến nghị Consumer dùng correlationId để detect duplicate |