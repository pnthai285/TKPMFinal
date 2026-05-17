# Delta for Registration

## Mô tả tính năng
Tính năng Registration cho phép sinh viên đăng ký tham gia workshop miễn phí hoặc có phí. Hệ thống được thiết kế để duy trì tính công bằng và nhất quán dữ liệu dưới tải đột biến (12.000 sinh viên), chống oversell thông qua cơ chế khóa dòng (row lock) cấp database, đồng thời hỗ trợ giữ chỗ tạm thời (hold slot) cho luồng thanh toán. Thiết kế đảm bảo mọi quyết định phân bổ chỗ ngồi đều được thực thi trong transaction ngắn, không phụ thuộc vào số chỗ hiển thị realtime, và tách biệt hoàn toàn các tác vụ nặng (gửi email, gọi payment gateway) ra khỏi vùng giữ lock.

---

## ADDED Requirements

### Requirement: Đăng ký Workshop & Chống Oversell
Hệ thống PHẢI cho phép sinh viên đăng ký workshop trong khi ngăn chặn tuyệt đối tình trạng oversell thông qua database-level row locking và kiểm tra giới hạn chỗ ngồi trong transaction.

#### Scenario: Sinh viên đăng ký workshop miễn phí
- GIVEN một sinh viên đã xác thực có role `STUDENT`, token queue hợp lệ và `Idempotency-Key`
- WHEN sinh viên gửi request `POST /registrations` cho workshop miễn phí
- THEN backend kiểm tra authentication, rate limit, queue token và mở database transaction
- AND khóa dòng workshop bằng `SELECT ... FOR UPDATE`
- AND kiểm tra workshop `status = OPEN`, còn thời gian đăng ký và `confirmed_count + held_count < capacity`
- AND tạo registration với trạng thái `CONFIRMED`, tăng `confirmed_count`
- AND commit transaction
- AND phát event `RegistrationConfirmed` và event cập nhật realtime seat count

#### Scenario: Sinh viên đăng ký workshop có phí
- GIVEN điều kiện tương tự scenario trên nhưng workshop có phí (`fee_type = PAID`)
- WHEN sinh viên gửi request đăng ký
- THEN luồng validation và row lock được thực thi tương tự
- AND tạo registration với trạng thái `PENDING_PAYMENT` cùng `hold_expires_at` (hiện tại + 10 phút)
- AND tăng `held_count`
- AND commit transaction, trả về payment URL cho frontend để chuyển hướng thanh toán

#### Scenario: Workshop hết chỗ
- GIVEN một workshop có `confirmed_count + held_count >= capacity`
- WHEN sinh viên cố gắng đăng ký
- THEN backend trả về lỗi `WORKSHOP_FULL`
- AND KHÔNG tạo registration, giữ nguyên seat count

#### Scenario: Workshop bị hủy
- GIVEN một workshop có `status = CANCELLED`
- WHEN sinh viên cố gắng đăng ký
- THEN backend trả về lỗi `WORKSHOP_CANCELLED`

---

### Requirement: Xác thực Idempotency Key
Hệ thống PHẢI hỗ trợ idempotent registration thông qua header `Idempotency-Key`, lưu trữ key trong 24 giờ để xử lý client retry an toàn.

#### Scenario: Sinh viên retry request với cùng Idempotency-Key
- GIVEN một registration đã được tạo thành công kèm `Idempotency-Key`
- WHEN sinh viên gửi request mới với cùng key trong vòng 24 giờ
- THEN backend dò tìm key trong DB
- AND trả về kết quả registration cũ mà KHÔNG tạo bản ghi mới hoặc thay đổi seat count
- AND KHÔNG gọi lại payment gateway hoặc trigger notification trùng lặp

---

### Requirement: Xác thực Virtual Queue Token
Hệ thống PHẢI validate queue token trước khi xử lý registration, đảm bảo token chỉ dùng một lần (one-time-use) và gắn chặt với `user_id` + `workshop_id`.

#### Scenario: Queue token hợp lệ và được sử dụng
- GIVEN sinh viên có queue token hợp lệ trong Redis (`qt:{user_id}:{workshop_id}`)
- WHEN backend nhận request đăng ký
- THEN backend kiểm tra token tồn tại, đúng user/workshop và chưa hết hạn
- AND xử lý registration
- AND xóa token khỏi Redis ngay lập tức sau khi dùng thành công

#### Scenario: Queue token hết hạn
- GIVEN queue token đã quá TTL 120 giây
- WHEN sinh viên gửi request đăng ký kèm token cũ
- THEN backend trả về `QUEUE_TOKEN_EXPIRED`
- AND yêu cầu sinh viên xin token mới (quay lại hàng đợi ảo)

#### Scenario: Token bị lạm dụng (sai user/workshop)
- GIVEN token được cấp cho `user_id=A` và `workshop_id=X`
- WHEN request gửi đến với `user_id=B` hoặc `workshop_id=Y`
- THEN backend từ chối xử lý, coi như token không hợp lệ

---

### Requirement: Quản lý Hold Slot & Hết hạn tự động
Hệ thống PHẢI tự động giải phóng chỗ ngồi tạm thời khi sinh viên không hoàn tất thanh toán trong cửa sổ giữ chỗ.

#### Scenario: Hold slot hết hạn mà chưa thanh toán
- GIVEN một registration `PENDING_PAYMENT` đã vượt quá `hold_expires_at`
- WHEN BullMQ `hold-expire-worker` kích hoạt
- THEN worker kiểm tra trạng thái registration (idempotent check)
- AND chuyển registration sang `EXPIRED`, giảm `held_count`
- AND phát event `RegistrationExpired` để trả chỗ về pool

#### Scenario: Một sinh viên chỉ có một registration active cho một workshop
- GIVEN sinh viên đã có registration `CONFIRMED` hoặc `PENDING_PAYMENT` cho workshop X
- WHEN sinh viên cố đăng ký lại workshop X
- THEN backend từ chối request nhờ ràng buộc unique index
- AND trả về lỗi `DUPLICATE_ACTIVE_REGISTRATION`

---

### Requirement: An toàn Transaction & Cô lập I/O
Hệ thống PHẢI đảm bảo transaction giữ row lock cực ngắn và KHÔNG thực hiện bất kỳ tác vụ nặng nào (email, payment, QR generation) trong vùng lock.

#### Scenario: Transaction giữ lock tối thiểu
- GIVEN registration transaction đang chạy
- WHEN row lock được giữ trên dòng `workshops`
- THEN transaction CHỈ thực hiện: kiểm tra capacity, insert registration, update counters
- AND commit ngay lập tức
- AND mọi tác vụ external (gọi payment, sinh QR, gửi notification) được thực thi SAU khi transaction hoàn tất

---

## Technical Constraints

| Tham số | Giá trị | Lý do / Ghi chú triển khai |
| --- | --- | --- |
| **Cơ chế khóa dòng** | `SELECT ... FOR UPDATE` | Ngăn tranh chấp ghi đồng thời, đảm bảo quyết định cuối cùng tại DB |
| **Bất biến chỗ ngồi** | `confirmed_count + held_count ≤ capacity` | Ràng buộc cứng kiểm tra trong transaction, không tin UI realtime |
| **Idempotency-Key TTL** | 24 giờ | Đủ để client retry trong phiên đăng ký; sau đó cho phép đăng ký lại |
| **Hold Slot TTL** | 10 phút (`hold_expires_at`) | Cân bằng thời gian sinh viên thanh toán & công bằng phân bổ chỗ |
| **Queue Token TTL** | 120 giây | Cửa sổ thời gian sinh viên hoàn tất thao tác đăng ký |
| **One-time-use Token** | Xóa khỏi Redis ngay sau khi dùng | Ngăn replay, đảm bảo mỗi token chỉ điều hướng 1 request vào DB |
| **Transaction Scope** | Không chứa I/O ngoài (email, gateway, QR) | Giữ lock < 50ms, tránh deadlock & giảm throughput bottleneck |
| **Active Registration Rule** | Partial Unique Index `(student_id, workshop_id)` WHERE status IN (`CONFIRMED`, `PENDING_PAYMENT`) | Ngăn sinh viên đăng ký trùng workshop đang hoạt động |
| **Realtime Seat Count** | KHÔNG phải source of truth | Chỉ dùng cho UX; registration API luôn kiểm tra DB trong transaction |
| **Hold Expiration Trigger** | BullMQ delayed job (10 phút) hoặc cron reconcile | Tự động reclaim chỗ, không phụ thuộc client |

---

## Integration Points & Dependencies

### Phụ thuộc hệ thống
| Thành phần | Vai trò | Giao thức / Interface |
| --- | --- | --- |
| **PostgreSQL** | Source of truth cho seat count, registration state, idempotency keys | Prisma ORM, `SELECT FOR UPDATE`, Partial Unique Index |
| **Redis** | Lưu virtual queue token, kiểm tra one-time-use | `ioredis`, key `qt:{user_id}:{workshop_id}`, TTL 120s |
| **BullMQ** | Lập lịch hold-expire job, emit domain events | Queue `hold-expire`, delayed job, retry policy |
| **Payment Module** | Nhận registration `PENDING_PAYMENT`, khởi tạo payment intent | Internal service call / Domain event |
| **Notification Module** | Nhận event `RegistrationConfirmed` / `RegistrationExpired` | Outbox pattern, async dispatch |
| **Realtime Module** | Nhận event thay đổi seat count để push SSE | Redis Pub/Sub `ws:{workshop_id}:seats` |

### Endpoint API liên quan
| Method | Endpoint | Role yêu cầu | Mô tả |
| --- | --- | --- | --- |
| `POST` | `/registrations` | `STUDENT` | Tạo đăng ký (miễn phí/có phí), yêu cầu queue token + idempotency key |
| `GET` | `/me/registrations` | `STUDENT` | Danh sách đăng ký của chính sinh viên |
| `GET` | `/me/registrations/:id/qr` | `STUDENT` | Trả về QR image (base64) nếu status `CONFIRMED` |

### Event Domain phát ra
| Event | Trigger bởi | Payload chính | Consumer |
| --- | --- | --- | --- |
| `RegistrationConfirmed` | Transaction commit (free) | `{ registration_id, student_id, workshop_id }` | Notification, QR Generator |
| `SeatCountUpdated` | confirmed_count/held_count thay đổi | `{ workshop_id, remaining_seats, held_count, confirmed_count }` | Realtime SSE Module |
| `RegistrationExpired` | Hold-expire-worker | `{ registration_id, workshop_id, expired_at }` | Notification, Realtime |

---

## Error Handling Matrix

| Lỗi | Hành động hệ thống | Phản hồi cho user | Hành động khắc phục |
| --- | --- | --- | --- |
| **Workshop hết chỗ** | Kiểm tra trong transaction, không insert | `WORKSHOP_FULL` → UI hiển thị "Đã hết chỗ" | Sinh viên có thể đăng ký workshop khác hoặc chờ slot trống |
| **Queue token hết hạn** | Redis TTL expire, token không tồn tại | `QUEUE_TOKEN_EXPIRED` → Yêu cầu xin token mới | Frontend tự động request lại queue token |
| **Client retry timeout** | Idempotency-Key trùng trong 24h | Trả về registration cũ (HTTP 200) | Không tạo bản ghi mới, seat count không đổi |
| **Hold slot quá hạn** | Worker chuyển `PENDING_PAYMENT` → `EXPIRED` | Sinh viên thấy "Đăng ký đã hết hạn" | Cần đăng ký lại từ đầu nếu còn chỗ |
| **Đăng ký trùng workshop** | Partial unique index chặn ở DB | `DUPLICATE_ACTIVE_REGISTRATION` | UI hiển thị "Bạn đã đăng ký workshop này" |
| **Worker emit event lỗi** | BullMQ retry event domain | Không ảnh hưởng registration đã commit | Notification/Realtime tự sync lại khi worker hồi phục |
| **DB connection drop giữa transaction** | Prisma rollback tự động | `500 Internal Server Error` | Client retry với cùng idempotency key → hệ thống xử lý lại an toàn |

---

## Tiêu chí chấp nhận (Acceptance Criteria)

- [ ] 100 request đồng thời tranh 1 chỗ cuối cùng → chỉ đúng 1 request được tạo `CONFIRMED` hoặc `PENDING_PAYMENT`.
- [ ] Retry request với cùng `Idempotency-Key` trong 24h → trả về kết quả cũ, không tạo registration mới, không thay đổi seat count.
- [ ] Hold slot quá hạn 10 phút chưa thanh toán → worker tự động chuyển `EXPIRED`, giảm `held_count`, giải phóng chỗ.
- [ ] Workshop miễn phí đăng ký thành công → status `CONFIRMED`, phát sinh QR, gửi notification.
- [ ] Workshop có phí đăng ký thành công → status `PENDING_PAYMENT`, chuyển đúng sang payment flow, không gửi email xác nhận ngay.
- [ ] Queue token hết hạn hoặc dùng sai user/workshop → request bị từ chối ngay tại middleware.
- [ ] Transaction giữ lock KHÔNG chứa bất kỳ call nào đến payment gateway, email provider hoặc QR generator.
- [ ] Một sinh viên không thể có 2 registration `ACTIVE` (`CONFIRMED`/`PENDING_PAYMENT`) cho cùng 1 workshop.
- [ ] Số chỗ hiển thị trên UI có thể chênh lệch nhẹ do network, nhưng DB transaction luôn là source of truth cuối cùng.
- [ ] Event `RegistrationConfirmed` và `SeatCountUpdated` được phát ra đúng 1 lần sau khi transaction commit thành công.

---

## Ghi chú triển khai (Implementation Notes)

### Cấu trúc Payload Request
```typescript
interface CreateRegistrationPayload {
  workshopId: string;
  queueToken: string;        // Từ POST /workshops/:id/queue-token
  idempotencyKey: string;    // UUID v4 từ client
}
```

### Luồng Transaction & Row Lock (Pseudo-code)
```typescript
async function createRegistration(studentId: string, payload: CreateRegistrationPayload) {
  // 1. Validate queue token (Redis) → throw nếu hết hạn/sai user
  await validateQueueToken(studentId, payload.workshopId, payload.queueToken);

  // 2. Check idempotency (DB) → return cached nếu tồn tại
  const existing = await prisma.registrations.findUnique({
    where: { idempotency_key: payload.idempotencyKey }
  });
  if (existing) return existing;

  // 3. DB Transaction với Row Lock
  const result = await prisma.$transaction(async (tx) => {
    const workshop = await tx.workshops.findUnique({
      where: { id: payload.workshopId },
      select: { status: true, capacity: true, confirmed_count: true, held_count: true, ends_at: true },
    });
    if (workshop.status !== 'OPEN') throw new AppError('WORKSHOP_CLOSED_OR_CANCELLED');
    if (workshop.confirmed_count + workshop.held_count >= workshop.capacity) {
      throw new AppError('WORKSHOP_FULL');
    }

    // Tạo registration
    const reg = await tx.registrations.create({
      data: {
        student_id: studentId,
        workshop_id: payload.workshopId,
        status: workshop.fee_type === 'FREE' ? 'CONFIRMED' : 'PENDING_PAYMENT',
        hold_expires_at: workshop.fee_type === 'PAID' ? addMinutes(new Date(), 10) : null,
        idempotency_key: payload.idempotencyKey,
      }
    });

    // Update counters
    const updateData = workshop.fee_type === 'FREE'
      ? { confirmed_count: { increment: 1 } }
      : { held_count: { increment: 1 } };
    await tx.workshops.update({ where: { id: payload.workshopId }, data: updateData });

    return reg;
  });

  // 4. Xóa queue token sau khi commit thành công
  await redis.del(`qt:${studentId}:${payload.workshopId}`);

  // 5. Emit events & trigger async jobs (NGOÀI transaction)
  if (result.status === 'CONFIRMED') {
    eventBus.publish('RegistrationConfirmed', { registrationId: result.id });
    eventBus.publish('SeatCountUpdated', { workshopId: payload.workshopId });
    queueService.enqueueDelayed('hold-expire', { registrationId: result.id }, { delay: 600_000 }); // 10m
  } else {
    // PENDING_PAYMENT → enqueue hold-expire ngay
    queueService.enqueueDelayed('hold-expire', { registrationId: result.id }, { delay: 600_000 });
    paymentService.initiatePayment(result.id);
  }

  return result;
}
```

### Ràng buộc Database (Prisma Schema)
```prisma
model Registration {
  id                String   @id @default(uuid())
  student_id        String
  workshop_id       String
  status            String   // CONFIRMED, PENDING_PAYMENT, EXPIRED, CANCELLED, FAILED
  hold_expires_at   DateTime?
  qr_token_hash     String?
  idempotency_key   String   @unique
  created_at        DateTime @default(now())

  workshop Workshop @relation(fields: [workshop_id], references: [id])

  // Partial unique index để chống đăng ký trùng active
  @@unique([student_id, workshop_id], name: "unique_active_registration", fields: [status])
  // Note: Prisma partial index, cần viết raw SQL migration hoặc dùng @@ignore + check constraint
  // SQL Migration: CREATE UNIQUE INDEX unique_active_reg ON registrations (student_id, workshop_id) 
  //                WHERE status IN ('CONFIRMED', 'PENDING_PAYMENT');
}
```

### Xử lý Idempotency & Queue Token
- **Idempotency:** Kiểm tra `registrations.idempotency_key` trước khi mở transaction. Nếu tồn tại → trả về kết quả cũ, bỏ qua toàn bộ logic.
- **Queue Token:** Redis key `qt:{user_id}:{workshop_id}`. TTL `120s`. Middleware `QueueTokenGuard` đọc key, validate, nếu hợp lệ → cho qua. Sau khi transaction commit → `DEL` key. Nếu request fail/timeout → key vẫn tồn tại đến TTL, sinh viên phải xin lại → đảm bảo fairness.

