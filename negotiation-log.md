# Biên bản đàm phán hợp đồng API

- Cặp đàm phán: pair-01
- Product: Camera Stream (A) / AI Vision (B)
- Provider: AI Vision Service – Nhóm 14
- Consumer: Camera Stream Service – Nhóm 14
- Phiên: v1.0
- Ngày: 2026-05-17

---

## Issue #1

- Raised by: Consumer
- Endpoint: POST /vision/detect
- Concern: Consumer chưa rõ ảnh nên gửi dạng base64 hay URL. Hai cách có trade-off khác nhau về băng thông và độ trễ.
- Proposal: Provider hỗ trợ cả hai dạng, dùng oneOf + discriminator theo trường sourceType (BASE64 hoặc URL).
- Resolution: Accepted
- Rationale: oneOf + discriminator cho phép mở rộng thêm kiểu ảnh trong tương lai mà không breaking change. Consumer linh hoạt chọn cách phù hợp với hạ tầng.
- Impact: Schema DetectRequest dùng ImageSource với oneOf gồm ImageSourceBase64 và ImageSourceURL.

---

## Issue #2

- Raised by: Consumer
- Endpoint: POST /vision/detect
- Concern: Không có giới hạn kích thước ảnh rõ ràng, dễ gây timeout hoặc lỗi 500 khi gửi frame quá lớn.
- Proposal: Provider quy định rõ trong description: base64 tối đa 5MB, URL phải accessible từ nội bộ campus network.
- Resolution: Accepted
- Rationale: Giới hạn rõ ràng giúp Consumer tự validate trước khi gửi, tránh lãng phí tài nguyên server.
- Impact: Thêm mô tả giới hạn vào description của trường data trong ImageSourceBase64.

---

## Issue #3

- Raised by: Provider
- Endpoint: POST /vision/detect
- Concern: Consumer không gửi correlationId nhất quán, gây khó trace log khi có lỗi.
- Proposal: Bắt buộc trường correlationId trong request body, Provider sẽ echo lại trong response và log.
- Resolution: Accepted
- Rationale: correlationId là yêu cầu observability cơ bản trong kiến trúc microservice. Bắt buộc từ đầu tránh phải breaking change sau.
- Impact: correlationId thêm vào required list của DetectRequest.

---

## Issue #4

- Raised by: Consumer
- Endpoint: GET /vision/detections/{detectionId}
- Concern: Consumer muốn biết detection có thể polling được không, hay chỉ dùng để tra cứu sau khi đã có detectionId từ POST.
- Proposal: Provider xác nhận endpoint này chỉ dùng để tra cứu. Lab 02 là REST sync, POST /vision/detect trả kết quả ngay, không cần polling.
- Resolution: Accepted
- Rationale: Đơn giản hóa flow cho Lab 02. Nếu sau này cần async thì mới thêm polling hoặc webhook.
- Impact: Ghi rõ trong description của getDetectionById rằng đây là tra cứu, không phải polling.

---

## Issue #5

- Raised by: Consumer
- Endpoint: POST /vision/detect
- Concern: Khi AI Vision không detect được object nào, riskLevel trả về là gì? Consumer cần biết để xử lý logic downstream.
- Proposal: Provider quy định riskLevel = LOW khi objects rỗng, không trả null để tránh Consumer phải null-check.
- Resolution: Accepted
- Rationale: Tránh null propagation sang Core Business. LOW là giá trị an toàn mặc định khi không có bất thường.
- Impact: Ghi rõ trong description của riskLevel trong DetectResponse.

---

## Issue #6

- Raised by: Provider
- Endpoint: Tất cả endpoint có lỗi
- Concern: Response lỗi cần chuẩn hóa để Consumer xử lý nhất quán, tránh parse nhiều format khác nhau.
- Proposal: Tất cả lỗi 4xx/5xx dùng Problem Details (RFC 9457) với Content-Type application/problem+json, gồm type, title, status, detail, instance, errors.
- Resolution: Accepted
- Rationale: Problem Details là chuẩn công nghiệp, giúp Consumer viết một error handler duy nhất cho mọi lỗi.
- Impact: Tất cả responses lỗi trong components/responses đều dùng schema Problem với application/problem+json.

---

# Chốt hợp đồng v1.0

Provider sign-off: AI Vision Service – Nhóm 14 ✓  
Consumer sign-off: Camera Stream Service – Nhóm 14 ✓  
Witness (GV/TA): FIT4110 Teaching Team  
Date: 2026-05-17

---

## Ghi chú warning nếu Spectral còn cảnh báo

| Warning | Lý do chấp nhận tạm thời | Kế hoạch sửa |
|---|---|---|
| operation-description thiếu ở /health, /vision/detections/{detectionId}, /vision/models/info | Đã bổ sung description ở phiên v1.0 | Đã sửa, không còn warning |