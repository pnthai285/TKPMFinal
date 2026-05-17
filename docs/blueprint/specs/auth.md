# Delta for Auth & RBAC

## Mô tả tính năng
Tính năng Auth & RBAC quản lý xác thực người dùng, quản lý phiên làm việc và phân quyền truy cập cho 4 nhóm vai trò: `STUDENT`, `ORGANIZER`, `CHECKIN_STAFF`, `ADMIN`. Hệ thống sử dụng cơ chế **JWT không trạng thái (stateless)** kết hợp **Refresh Token Rotation** và **Redis Blocklist** để đảm bảo khả năng mở rộng ngang, bảo mật cao và khả năng thu hồi token khi cần. Mọi kiểm tra quyền được thực thi **100% tại Backend**, giao diện frontend chỉ mang tính hỗ trợ UX và không được coi là ranh giới bảo mật.

---

## ADDED Requirements

### Requirement: Xác thực không trạng thái dựa trên JWT
Hệ thống PHẢI sử dụng JWT (JSON Web Token) cho xác thực không trạng thái, kết hợp access token thời gian sống ngắn và refresh token có thể xoay vòng.

#### Scenario: Người dùng đăng nhập với thông tin hợp lệ
- GIVEN một người dùng có `status = ACTIVE`
- WHEN người dùng gửi email và mật khẩu hợp lệ đến `POST /auth/login`
- THEN backend xác thực mật khẩu bằng bcrypt hash
- AND cấp access token (JWT, TTL 15 phút) chứa `user_id`, `roles[]`, `iat`, `exp`
- AND cấp refresh token (JWT, TTL 7 ngày) chứa `user_id`, `jti`, `exp`
- AND access token được lưu trong bộ nhớ client (memory)
- AND refresh token được lưu trong cookie `HTTP-only`, `Secure`, `SameSite=Strict`

#### Scenario: Người dùng đăng nhập sai thông tin
- GIVEN bất kỳ người dùng nào
- WHEN người dùng gửi email hoặc mật khẩu không chính xác
- THEN backend trả về `401 Unauthorized` với message chung để tránh tiết lộ thông tin tồn tại tài khoản

#### Scenario: Tài khoản bị khóa cố đăng nhập
- GIVEN một người dùng có `status = LOCKED`
- WHEN người dùng gửi thông tin đăng nhập hợp lệ
- THEN backend trả về `403 Forbidden` với message "Tài khoản đã bị khóa"

---

### Requirement: Làm mới và xoay vòng Token (Token Rotation)
Hệ thống PHẢI hỗ trợ tự động làm mới access token thông qua refresh token, bắt buộc xoay vòng (rotation) mỗi lần sử dụng.

#### Scenario: Access token hết hạn, refresh token còn hợp lệ
- GIVEN một người dùng đã xác thực có access token hết hạn
- WHEN client gửi refresh token đến `POST /auth/refresh`
- THEN backend kiểm tra refresh token còn hạn và `jti` không nằm trong Redis blocklist
- AND cấp cặp token mới: access token (15 phút) + refresh token (7 ngày)
- AND thu hồi refresh token cũ bằng cách lưu `jti` vào Redis với TTL = thời gian còn lại của token cũ

#### Scenario: Refresh token hết hạn hoặc đã bị thu hồi
- GIVEN một refresh token đã hết hạn hoặc `jti` của nó đã nằm trong Redis blocklist
- WHEN client gửi token này đến `POST /auth/refresh`
- THEN backend trả về `401 Unauthorized`
- AND client bắt buộc chuyển hướng người dùng đến trang đăng nhập lại

#### Scenario: Tấn công Replay bằng refresh token đã dùng
- GIVEN một refresh token đã được xoay vòng thành công
- WHEN token cũ bị đánh cắp và gửi lại đến `POST /auth/refresh`
- THEN backend phát hiện `jti` đã nằm trong Redis blocklist
- AND trả về `401 Unauthorized`, đồng thời có thể kích hoạt cảnh báo bảo mật (tuỳ chính sách)

---

### Requirement: Kiểm soát truy cập dựa trên vai trò (RBAC)
Hệ thống PHẢI thực thi RBAC tại Backend thông qua Guard middleware. Việc ẩn/hiện giao diện ở frontend KHÔNG được dùng để thay thế kiểm tra quyền.

#### Scenario: Sinh viên cố truy cập API quản trị workshop
- GIVEN một người dùng đã xác thực có role `STUDENT`
- WHEN người dùng gọi API chỉ dành cho `ORGANIZER` hoặc `ADMIN` (ví dụ: `POST /admin/workshops`)
- THEN `RolesGuard` từ chối request
- AND backend trả về `403 Forbidden`

#### Scenario: Nhân sự check-in cố truy cập API admin
- GIVEN một người dùng đã xác thực có role `CHECKIN_STAFF`
- WHEN người dùng gọi API admin workshop
- THEN backend trả về `403 Forbidden`

#### Scenario: Organizer cố xem dữ liệu riêng tư của sinh viên khác
- GIVEN một người dùng đã xác thực có role `ORGANIZER`
- WHEN người dùng cố gắng gọi endpoint QR cá nhân của sinh viên khác (`GET /me/registrations/:id/qr`)
- THEN backend kiểm tra quyền sở hữu (`student_id` hoặc `user_id` khớp với token)
- AND trả về `403 Forbidden` do vi phạm nguyên tắc ownership

#### Scenario: Admin gán vai trò cho người dùng
- GIVEN một người dùng đã xác thực có role `ADMIN`
- WHEN admin gán vai trò mới cho người dùng đích qua `POST /admin/users/:id/roles`
- THEN vai trò được lưu vào database
- AND hệ thống ghi nhận một bản ghi audit log chứa hành động, người thực hiện và timestamp

---

### Requirement: Bảo mật mật khẩu và Hashing
Hệ thống PHẢI mã hóa mật khẩu bằng bcrypt với cost factor ≥ 12. Tuyệt đối không lưu plaintext, MD5 hoặc SHA-1.

#### Scenario: Lưu trữ mật khẩu khi đăng ký hoặc đổi mật khẩu
- GIVEN một người dùng mới hoặc người dùng yêu cầu đổi mật khẩu
- WHEN người dùng gửi mật khẩu gốc
- THEN backend hash mật khẩu bằng bcrypt (cost factor ≥ 12) trước khi lưu vào `users.password_hash`

---

### Requirement: Xác thực Webhook độc lập
Hệ thống PHẢI xác thực webhook từ cổng thanh toán bằng chữ ký HMAC, không phụ thuộc vào hệ thống RBAC của người dùng.

#### Scenario: Cổng thanh toán gửi webhook xác nhận
- GIVEN cổng thanh toán được cấu hình shared HMAC secret
- WHEN cổng thanh toán gửi webhook có chữ ký đến `POST /payments/webhook`
- THEN `WebhookSignatureGuard` xác thực HMAC-SHA256
- AND chỉ xử lý payload nếu chữ ký hợp lệ, bỏ qua việc kiểm tra `Authorization` header

---

### Requirement: Ghi nhật ký kiểm toán (Audit Logging)
Hệ thống PHẢI ghi lại audit log cho tất cả các hành động nhạy cảm liên quan đến bảo mật và quản trị.

#### Scenario: Ghi log hành động bảo mật/quản trị
- GIVEN bất kỳ hành động nào sau đây xảy ra: đăng nhập, thay đổi vai trò, tạo/hủy workshop, thu hồi token
- THEN hệ thống ghi bản ghi audit log bao gồm: `actor_user_id`, `action_type`, `target_resource`, `timestamp`, và `ip_address`
- AND log được lưu vào bảng riêng hoặc hệ thống tập trung để truy vết và điều tra sự cố

---

## Technical Constraints

| Tham số | Giá trị | Lý do / Ghi chú triển khai |
| --- | --- | --- |
| **Access Token TTL** | 15 phút | Thời gian sống ngắn giảm thiểu rủi ro nếu token bị lộ qua XSS |
| **Refresh Token TTL** | 7 ngày | Cân bằng giữa bảo mật và trải nghiệm người dùng (ít phải đăng nhập lại) |
| **Lưu trữ Access Token** | Client memory | Không lưu trong `localStorage`/`sessionStorage` để tránh bị đánh cắp bởi script |
| **Lưu trữ Refresh Token** | `HTTP-only` + `Secure` cookie | Ngăn JavaScript truy cập, bảo vệ khỏi XSS; yêu cầu HTTPS |
| **Hashing mật khẩu** | bcrypt, cost factor ≥ 12 | Chuẩn công nghiệp, kháng brute-force, phù hợp CPU hiện đại |
| **Cơ chế xoay vòng** | Bắt buộc mỗi lần refresh | Chống replay attack; token cũ bị revoke ngay lập tức |
| **Thu hồi token** | Redis blocklist `jti:{jti}` | JWT vốn stateless, dùng Redis lưu `jti` với TTL = thời gian còn lại của token để revoke hiệu quả |
| **Điểm thực thi quyền** | Backend Guards (`JwtAuthGuard` → `RolesGuard`) | Frontend chỉ ẩn UI; mọi endpoint đều phải được bảo vệ ở server |
| **Xác thực Webhook** | HMAC-SHA256 signature | Tách biệt hoàn toàn khỏi luồng user auth; chống giả mạo webhook từ bên ngoài |
| **Phạm vi Audit Log** | Login, role change, workshop CRUD, token revoke | Đảm bảo khả năng truy vết (traceability) cho admin và kiểm toán hệ thống |
| **Kiểm tra Ownership** | Backend validate `user_id` / `student_id` trong token | Ngăn người dùng A thao tác trên dữ liệu của người dùng B dù cùng role |
