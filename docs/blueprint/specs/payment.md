# Delta for Payment

## Mô tả tính năng
Tính năng Payment xử lý luồng thanh toán cho workshop có phí theo mô hình bất đồng bộ: tạo payment intent, chuyển hướng sinh viên đến cổng thanh toán, và chỉ xác nhận đăng ký khi nhận được signed webhook từ gateway. Hệ thống áp dụng cơ chế Idempotency để chống trừ tiền hai lần, Circuit Breaker để cô lập lỗi gateway, cùng Hold Slot để quản lý chỗ ngồi tạm thời. Thiết kế đảm bảo khi cổng thanh toán gặp sự cố kéo dài, các tính năng khác (xem lịch, đăng ký miễn phí) vẫn hoạt động bình thường, và trạng thái thanh toán cuối cùng luôn chính xác bất kể lỗi mạng hay client disconnect.

---

## ADDED Requirements

### Requirement: Thanh toán bất đồng bộ & Xác thực qua Webhook
Hệ thống PHẢI xử lý thanh toán workshop có phí thông qua payment intent bất đồng bộ. Việc xác nhận đăng ký (CONFIRMED) CHỈ được thực hiện khi nhận và xác thực chữ ký webhook từ gateway, tuyệt đối không tin tưởng vào client redirect.

#### Scenario: Sinh viên khởi tạo thanh toán cho workshop có phí
- GIVEN một sinh viên có registration trạng thái `PENDING_PAYMENT` và đã được giữ chỗ (hold slot)
- WHEN Payment Module tạo bản ghi payment và gọi gateway qua Payment Adapter
- THEN payment record được tạo với trạng thái `INITIATED`
- AND Adapter gọi gateway để tạo payment intent kèm `idempotency_key`
- AND gateway trả về `payment_url` và `payment_intent_id`
- AND sinh viên được chuyển hướng đến `payment_url` để hoàn tất thanh toán

#### Scenario: Thanh toán thành công qua webhook
- GIVEN sinh viên đã hoàn tất thanh toán tại cổng gateway
- WHEN gateway gửi signed webhook đến `POST /payments/webhook`
- THEN Backend xác thực chữ ký HMAC-SHA256 của webhook
- AND Payment Module cập nhật payment status thành `SUCCEEDED`
- AND Registration Module chuyển registration sang `CONFIRMED`
- AND giảm `held_count`, tăng `confirmed_count`, sinh mã QR
- AND Notification Module gửi xác nhận qua in-app và email

#### Scenario: Webhook bị gửi trùng lặp
- GIVEN một webhook cho payment đã được xử lý thành công trước đó
- WHEN gateway gửi lại cùng webhook do cơ chế retry hoặc mạng không ổn định
- THEN Backend trả về `200 OK` ngay lập tức
- AND KHÔNG cập nhật seat count hoặc tạo bản ghi trùng lặp

---

### Requirement: Idempotency trong thanh toán
Hệ thống PHẢI đảm bảo mỗi lần attempt đăng ký workshop có phí chỉ tạo ra tối đa một payment transaction, bất chấp client retry hoặc click nhiều lần.

#### Scenario: Sinh viên nhấn nút thanh toán nhiều lần
- GIVEN một registration đang ở trạng thái `PENDING_PAYMENT`
- WHEN sinh viên gửi nhiều request thanh toán cho cùng registration này
- THEN chỉ một payment intent được tạo (định danh bởi `idempotency_key`)
- AND các request tiếp theo trả về `payment_intent_id` đã tồn tại mà không gọi lại gateway

#### Scenario: Client timeout sau khi thanh toán
- GIVEN sinh viên đã thanh toán thành công nhưng browser timeout trước khi nhận phản hồi
- WHEN gateway vẫn gửi webhook độc lập đến backend
- THEN Backend xử lý webhook bình thường, cập nhật registration `CONFIRMED`
- AND sinh viên KHÔNG cần thanh toán lại, chỉ cần tải lại trang để xem QR

---

### Requirement: Circuit Breaker cô lập lỗi Gateway
Hệ thống PHẢI triển khai Circuit Breaker để ngăn lỗi từ cổng thanh toán làm sập toàn bộ backend, đảm bảo graceful degradation.

#### Scenario: Gateway lỗi liên tục kích hoạt Circuit Breaker
- GIVEN cổng thanh toán thất bại 5 lần liên tiếp trong vòng 30 giây
- WHEN Circuit Breaker chuyển sang trạng thái `Open`
- THEN mọi request tạo payment intent mới bị từ chối ngay lập tức
- AND hệ thống trả về `503 Service Unavailable` kèm thông báo "Thanh toán tạm gián đoạn"

#### Scenario: Circuit Breaker tự phục hồi
- GIVEN Circuit Breaker đang ở trạng thái `Half-Open` (sau 30 giây chờ)
- WHEN 3 request thăm dò (probe) liên tiếp thành công
- THEN Circuit Breaker chuyển về trạng thái `Closed`
- AND luồng thanh toán hoạt động bình thường trở lại

#### Scenario: Gateway down nhưng tính năng khác vẫn chạy
- GIVEN Circuit Breaker đang `Open`
- WHEN sinh viên truy cập xem lịch workshop hoặc đăng ký workshop miễn phí
- THEN các tính năng này hoạt động bình thường, không bị ảnh hưởng

---

### Requirement: Quản lý Hold Slot & Tự động Hết hạn
Hệ thống PHẢI quản lý chỗ ngồi tạm thời (hold slot) cho registration có phí và tự động giải phóng khi quá hạn thanh toán.

#### Scenario: Hold slot hết hạn mà chưa thanh toán
- GIVEN một registration `PENDING_PAYMENT` đã vượt quá `hold_expires_at` (10 phút)
- WHEN Hold-Expire Worker (BullMQ delayed job) chạy
- THEN registration chuyển sang trạng thái `EXPIRED`
- AND giảm `workshops.held_count`, giải phóng chỗ cho người khác

#### Scenario: Webhook thành công đến SAU khi hold đã hết hạn
- GIVEN một webhook `SUCCEEDED` được gửi đến khi registration đã chuyển `EXPIRED`
- WHEN Backend xử lý webhook này
- THEN hệ thống kích hoạt auto-refund thông qua `PaymentAdapter.refund()` với idempotency key riêng
- AND registration chuyển sang `NEEDS_REVIEW` để admin đối soát thủ công

---

## Technical Constraints

| Tham số | Giá trị | Lý do / Ghi chú triển khai |
| --- | --- | --- |
| **Xác thực webhook** | HMAC-SHA256 signature | Chống giả mạo webhook, không dùng JWT/session |
| **Idempotency key** | `payments.idempotency_key` UNIQUE | Ngăn duplicate payment intent cho cùng registration |
| **Payment Intent ID** | `payments.payment_intent_id` UNIQUE | Ràng buộc 1 intent / 1 payment record |
| **Circuit Breaker: Open** | ≥ 5 lỗi liên tiếp trong 30s HOẶC >50% lỗi/30s (min 10 req) | Giới hạn ngưỡng kích hoạt để cân bằng giữa độ nhạy và ổn định |
| **Circuit Breaker: Timeout** | 5 giây/gateway call | Vượt quá → tính là failure, tăng counter CB |
| **Circuit Breaker: Open Duration** | 30 giây | Thời gian chờ trước khi chuyển sang Half-Open |
| **Circuit Breaker: Half-Open Probes** | 3 request thành công liên tiếp | Xác nhận gateway đã ổn định trước khi mở lại hoàn toàn |
| **Hold Slot TTL** | 10 phút (`hold_expires_at`) | Cân bằng thời gian sinh viên thanh toán và công bằng chỗ ngồi |
| **Trạng thái Payment** | `INITIATED`, `SUCCEEDED`, `FAILED` | Theo dõi vòng đời giao dịch rõ ràng |
| **Trạng thái Registration (fee)** | `PENDING_PAYMENT` → `CONFIRMED` / `EXPIRED` / `FAILED` / `NEEDS_REVIEW` | Phản ánh chính xác trạng thái giữ chỗ và thanh toán |
| **Source of Truth** | Webhook signed | Client redirect KHÔNG được tin tưởng để confirm registration |

---

## Integration Points & Dependencies

### Phụ thuộc hệ thống
| Thành phần | Vai trò | Giao thức / Interface |
| --- | --- | --- |
| **Registration Module** | Tạo `PENDING_PAYMENT` + hold slot, trigger payment intent | DB transaction, domain events |
| **Payment Gateway (Mock)** | Xử lý thanh toán, trả webhook signed | HTTP REST, HMAC-SHA256 signature |
| **BullMQ + Redis** | Hold-expire delayed jobs, payment-reconcile cron jobs, CB state storage | `ioredis`, queue names `hold-expire`, `payment-reconcile` |
| **PostgreSQL** | Lưu payment record, idempotency keys, enforce uniqueness | Prisma ORM, `UNIQUE` constraints |
| **Notification Module** | Gửi xác nhận khi `SUCCEEDED`, cảnh báo khi `FAILED` | Outbox pattern, async events |

### Endpoint API liên quan
| Method | Endpoint | Role / Auth | Mô tả |
| --- | --- | --- | --- |
| `POST` | `/payments/:registrationId/intent` | `STUDENT` | Tạo payment intent, redirect URL |
| `POST` | `/payments/webhook` | Gateway HMAC Signature | Nhận & xử lý kết quả thanh toán từ gateway |
| `GET` | `/payments/:paymentId/status` | `STUDENT` | Kiểm tra trạng thái payment hiện tại |

### Event Domain phát ra
| Event | Trigger bởi | Payload chính | Consumer |
| --- | --- | --- | --- |
| `PaymentIntentCreated` | Intent tạo thành công | `{ registration_id, intent_id, amount }` | Audit Log, UI State |
| `PaymentSucceeded` | Webhook `SUCCEEDED` xử lý xong | `{ registration_id, payment_id, gateway_payload }` | Registration, Notification |
| `PaymentFailed` | Webhook `FAILED` hoặc timeout | `{ registration_id, error_code }` | Registration, Notification |
| `HoldExpired` | Hold-Expire Worker | `{ registration_id, expired_at }` | Registration, Notification |

---

## Error Handling Matrix

| Lỗi | Hành động hệ thống | Phản hồi cho user | Hành động khắc phục |
| --- | --- | --- | --- |
| **Gateway timeout (>5s)** | CB tăng counter, giữ registration `PENDING_PAYMENT` | "Đang xử lý thanh toán, vui lòng đợi" | Reconcile worker kiểm tra sau 15 phút |
| **Circuit Breaker Open** | Từ chối tạo intent mới, trả `503` | "Thanh toán tạm gián đoạn. Vui lòng thử lại sau." | Chờ 30s → Half-Open → probe 3 lần |
| **Webhook chữ ký sai** | Reject ngay tại `WebhookSignatureGuard`, log warning | Không hiển thị (xử lý nội bộ) | Kiểm tra `HMAC_WEBHOOK_SECRET` config |
| **Webhook trùng lặp** | Kiểm tra `payment_intent_id` UNIQUE, trả `200 OK` | Không ảnh hưởng user | Idempotent handler bỏ qua |
| **Hold expire trước khi pay** | Worker chuyển `EXPIRED`, giảm `held_count` | Sinh viên thấy "Đăng ký đã hết hạn" | Cần đăng ký lại từ đầu |
| **Pay success sau khi expire** | Auto-refund, chuyển `NEEDS_REVIEW` | Admin nhận cảnh báo đối soát | Admin hoàn tiền thủ công hoặc cấp lại chỗ |
| **Client mạng mất sau pay** | Webhook vẫn xử lý độc lập | Sinh viên refresh trang thấy QR đã sinh | Không cần thao tác thêm |

---

## Tiêu chí chấp nhận (Acceptance Criteria)

- [ ] Sinh viên tạo payment intent thành công → được redirect đến mock gateway, payment trạng thái `INITIATED`.
- [ ] Webhook `SUCCEEDED` đến → backend xác thực HMAC, cập nhật payment `SUCCEEDED`, registration `CONFIRMED`, sinh QR, gửi notification.
- [ ] Webhook `FAILED` đến → payment `FAILED`, registration `FAILED` hoặc retry, `held_count` được giảm.
- [ ] Client bấm thanh toán nhiều lần → chỉ 1 payment intent được tạo, các lần sau trả kết quả cũ (idempotent).
- [ ] Webhook bị gửi trùng nhiều lần → handler trả `200 OK`, seat count không thay đổi, không tạo bản ghi trùng.
- [ ] Gateway timeout 5 lần liên tiếp trong 30s → Circuit Breaker chuyển `Open`, từ chối request mới với `503`.
- [ ] Circuit Breaker `Open` → xem lịch và đăng ký workshop miễn phí vẫn hoạt động bình thường.
- [ ] Hold slot quá hạn 10 phút chưa pay → Hold-Expire Worker chuyển registration `EXPIRED`, trả chỗ về pool.
- [ ] Webhook `SUCCEEDED` đến sau khi hold đã `EXPIRED` → hệ thống gọi `refund()`, chuyển registration `NEEDS_REVIEW`.
- [ ] Client timeout sau khi pay thành công → webhook vẫn xử lý đúng, sinh viên không phải pay lại.
- [ ] `payment_intent_id` và `idempotency_key` đều có `UNIQUE constraint` trong DB.
- [ ] Không bao giờ confirm registration chỉ dựa vào client redirect hoặc URL params.

---

## Ghi chú triển khai (Implementation Notes)

### Cấu trúc Payment Payload & Webhook
```typescript
interface PaymentIntentPayload {
  registrationId: string;
  amount: number;
  currency: 'VND';
  idempotencyKey: string; // UUID v4 từ client
}

interface GatewayWebhookPayload {
  payment_intent_id: string;
  status: 'SUCCEEDED' | 'FAILED';
  timestamp: string;
  signature: string; // HMAC-SHA256
}
```

### Luồng Xử lý Webhook Idempotent (Pseudo-code)
```typescript
async function handlePaymentWebhook(payload: GatewayWebhookPayload) {
  // 1. Xác thực chữ ký
  if (!verifyHMAC(payload.signature, payload)) {
    throw new UnauthorizedException('Invalid webhook signature');
  }

  // 2. Idempotent check theo payment_intent_id
  const existingPayment = await prisma.payments.findUnique({
    where: { payment_intent_id: payload.payment_intent_id }
  });
  if (existingPayment) {
    // Đã xử lý rồi → trả 200, không làm gì thêm
    return { status: 'ALREADY_PROCESSED' };
  }

  // 3. Xử lý transaction
  await prisma.$transaction(async (tx) => {
    // Cập nhật payment
    await tx.payments.create({
      data: { registration_id: getRegIdFromIntent(payload), ... }
    });

    // Cập nhật registration & seat count
    if (payload.status === 'SUCCEEDED') {
      await handleSuccessfulPayment(tx, payload);
    } else {
      await handleFailedPayment(tx, payload);
    }
  });
}
```

### Logic Circuit Breaker (Redis State)
```typescript
class CircuitBreakerService {
  async execute<T>(fn: () => Promise<T>): Promise<T> {
    const state = await redis.get('cb:payment_gateway');
    if (state === 'OPEN') throw new ServiceUnavailableException('Payment gateway unavailable');

    try {
      const result = await Promise.race([
        fn(),
        new Promise((_, reject) => setTimeout(() => reject(new Error('TIMEOUT')), 5000))
      ]);
      await this.recordSuccess();
      return result;
    } catch (error) {
      await this.recordFailure();
      if (await this.shouldOpen()) {
        await redis.set('cb:payment_gateway', 'OPEN', 'EX', 30);
      }
      throw error;
    }
  }
  // recordSuccess/recordFailure xử lý counters, probe count cho Half-Open
}
```

### Auto-Refund & Late Webhook Handling
```typescript
// Trong handleSuccessfulPayment, kiểm tra hold_expires_at
const registration = await tx.registrations.findUnique({ where: { id: regId } });
if (registration.status === 'EXPIRED' || registration.hold_expires_at < new Date()) {
  // Quá hạn → cần hoàn tiền
  await paymentAdapter.refund(payload.payment_intent_id, generateIdempotencyKey());
  await tx.registrations.update({
    where: { id: regId },
    data: { status: 'NEEDS_REVIEW', note: 'Late webhook after hold expiry' }
  });
  return;
}
// Nếu còn hạn → proceed normal CONFIRM flow
```