# Versioning Log

## v1.0.0 — 2026-05-17

### Thay đổi
- Khởi tạo contract API giữa Camera Stream (Consumer) và AI Vision (Provider).
- Thêm 4 endpoint: GET /health, POST /vision/detect, GET /vision/detections/{detectionId}, GET /vision/models/info.
- Schema DetectRequest dùng oneOf + discriminator cho ImageSource (BASE64 / URL).
- Schema DetectResponse có trường note với union type ["string", "null"].
- Tất cả lỗi 4xx/5xx dùng Problem Details (RFC 9457).
- Pass Spectral lint với campus-spectral.yaml, không có error.

### Quyết định thiết kế
- Chọn REST sync thay vì async vì user story yêu cầu response nhanh cho Camera.
- correlationId bắt buộc để phục vụ trace log trong kiến trúc microservice.
- riskLevel mặc định LOW khi không detect được object, không dùng null.

### Sign-off
- Provider: AI Vision Service – Nhóm 14 ✓
- Consumer: Camera Stream Service – Nhóm 14 ✓
- Date: 2026-05-17