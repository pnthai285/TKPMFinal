# Delta for Notification Module

## Mô tả tính năng
Notification Module chịu trách nhiệm gửi thông báo xác nhận và cảnh báo thay đổi workshop đến sinh viên qua các kênh như email và in-app. Hệ thống sử dụng kiến trúc **Event-Driven** kết hợp **Outbox Pattern** để đảm bảo không mất sự kiện, cùng **Channel Adapter Pattern** để tách biệt hoàn toàn logic nghiệp vụ khỏi logic truyền tải. Thiết kế cho phép mở rộng kênh mới (ví dụ: Telegram) trong tương lai mà không cần sửa đổi mã nguồn của Registration, Payment hay Workshop module. Mọi thất bại của kênh thông báo đều được xử lý độc lập, không được phép rollback giao dịch nghiệp vụ gốc.

---

## ADDED Requirements

### Requirement: Event-Driven Notification với Channel Adapter
Hệ thống PHẢI xử lý thông báo thông qua mô hình adapter, tách biệt module nghiệp vụ khỏi cơ chế gửi kênh cụ thể.

#### Scenario: Mapping sự kiện đến template và kênh gửi
- GIVEN hệ thống đã cấu hình bảng ánh xạ (event → template → channels)
- WHEN một domain event được phát ra (ví dụ: `RegistrationConfirmed`, `PaymentSucceeded`, `WorkshopCancelled`, `WorkshopUpdated`, `RegistrationExpired`, `PaymentFailed`)
- THEN notification worker tra bảng mapping để chọn template và danh sách kênh mục tiêu
- AND tạo bản ghi `notification_deliveries` cho từng kênh (mỗi kênh 1 record độc lập)
- AND gọi `ChannelAdapter` tương ứng để thực thi việc gửi

#### Scenario: Thêm kênh mới không sửa logic nghiệp vụ
- GIVEN hệ thống đang chạy ổn định với email và in-app
- WHEN team phát triển thêm kênh Telegram
- THEN chỉ cần implement `TelegramAdapter` thỏa mãn interface `NotificationChannelAdapter`
- AND cập nhật cấu hình mapping thêm kênh `telegram`
- AND KHÔNG cần sửa code trong Registration, Payment hay Workshop module

---

### Requirement: Outbox Pattern và Xử lý Bất đồng bộ
Hệ thống PHẢI ghi nhận sự kiện vào database trước khi xử lý bất đồng bộ, đảm bảo tính nhất quán cuối cùng (eventual consistency).

#### Scenario: Module nghiệp vụ phát sự kiện
- GIVEN Registration hoặc Payment module hoàn tất transaction thành công
- WHEN module ghi một bản ghi vào bảng `notification_events` (outbox table)
- THEN transaction nghiệp vụ commit thành công, sự kiện được lưu an toàn
- AND Notification worker độc lập đọc hàng đợi outbox theo polling
- AND worker xử lý sự kiện, tạo delivery record, và gọi channel adapter

#### Scenario: Worker xử lý lại sự kiện đã xử lý
- GIVEN một domain event đã được worker xử lý thành công trước đó
- WHEN worker đọc lại cùng event do cơ chế retry hoặc restart
- THEN worker kiểm tra constraint `UNIQUE(event_id, channel)`
- AND bỏ qua việc tạo duplicate delivery, xử lý lỗi gracefully

---

### Requirement: Chính sách Retry Độc lập Theo Kênh
Hệ thống PHẢI retry các lần gửi thất bại theo lịch trình exponential backoff, mỗi kênh retry độc lập với nhau.

#### Scenario: Email provider tạm lỗi và retry theo backoff
- GIVEN một `notification_delivery` cho kênh `email` thất bại lần đầu
- WHEN worker áp dụng retry policy
- THEN lịch retry được thực thi chính xác:
  - Attempt 1: Ngay lập tức
  - Attempt 2: Chờ 1 phút
  - Attempt 3: Chờ 5 phút
  - Attempt 4: Chờ 30 phút
  - Attempt 5 (cuối cùng): Chờ 2 giờ
- AND nếu vẫn thất bại sau attempt 5 → chuyển status thành `FAILED_PERMANENT`
- AND delivery của kênh `in-app` cho cùng sự kiện KHÔNG bị ảnh hưởng, vẫn gửi bình thường

#### Scenario: Template thiếu biến bắt buộc
- GIVEN một notification template tham chiếu biến không tồn tại trong payload sự kiện
- WHEN worker cố gắng render template
- THEN delivery được đánh dấu `FAILED` kèm `error_reason` cụ thể
- AND KHÔNG retry lại để tránh lỗi lặp vô hạn

---

### Requirement: Cách ly Thất bại Thông báo (Notification Isolation)
Hệ thống PHẢI đảm bảo việc gửi thông báo thất bại không bao giờ làm rollback hoặc hủy transaction nghiệp vụ gốc.

#### Scenario: Email thất bại nhưng đăng ký vẫn thành công
- GIVEN một sinh viên đăng ký workshop miễn phí thành công
- WHEN Notification Module cố gắng gửi email xác nhận nhưng gặp lỗi kết nối
- THEN registration vẫn giữ nguyên trạng thái `CONFIRMED`
- AND hệ thống ghi nhận lỗi delivery, tiếp tục retry độc lập
- AND sinh viên vẫn nhận được QR và in-app notification (nếu cấu hình)

#### Scenario: In-app lỗi, email vẫn gửi được
- GIVEN cùng một sự kiện `PaymentSucceeded`
- WHEN kênh `in-app` gặp lỗi database trong lúc gửi
- THEN kênh `email` vẫn hoạt động bình thường, gửi xác nhận thành công
- AND `in-app` được đưa vào hàng retry độc lập

---

## Technical Constraints

| Tham số | Giá trị | Lý do / Ghi chú triển khai |
| --- | --- | --- |
| **Outbox table** | `notification_events` | Đảm bảo sự kiện không mất nếu worker crash ngay sau commit nghiệp vụ |
| **Delivery table** | `notification_deliveries` | Theo dõi trạng thái gửi chi tiết cho từng kênh, hỗ trợ audit & retry |
| **Max retry attempts** | 5 lần | Cân bằng giữa độ tin cậy và tài nguyên; sau đó → `FAILED_PERMANENT` |
| **Backoff schedule** | 0s → 1m → 5m → 30m → 2h | Exponential backoff giảm tải cho provider lỗi, tránh spam request |
| **Retry independence** | Mỗi kênh retry riêng biệt | Email lỗi không block in-app, và ngược lại |
| **Unique constraint** | `UNIQUE(event_id, channel)` | Ngăn chặn gửi trùng lặp khi worker đọc lại outbox hoặc retry |
| **Channel interface** | `NotificationChannelAdapter` | Contract chuẩn: `send(recipient, payload) → Promise<void>` |
| **Business isolation** | Notification failure ≠ Business rollback | Registration/Payment commit trước, outbox ghi sau; không dùng distributed transaction |
| **Status tracking** | `PENDING` → `SENT` / `FAILED` / `FAILED_PERMANENT` | Cho phép admin/debug truy vết lịch sử gửi thông báo |
| **Template engine** | Handlebars/Mustache hoặc tương đương | Hỗ trợ biến động, validate biến trước khi render để tránh lỗi runtime |

---

## Integration Points & Dependencies

### Phụ thuộc hệ thống
| Thành phần | Vai trò | Giao thức / Interface |
| --- | --- | --- |
| **Business Modules** | Phát domain event, ghi outbox | DB transaction (`notification_events` table) |
| **BullMQ + Redis** | Queue job gửi thông báo theo channel | `ioredis`, queue name `notification` |
| **Email Provider** | Gửi email xác nhận/cảnh báo | HTTP REST SDK (ví dụ: Resend, SendGrid) |
| **In-App Service** | Lưu & hiển thị thông báo trong Student Web | DB insert vào `user_notifications` |
| **Database (PostgreSQL)** | Lưu outbox, delivery status, retry count | Prisma ORM, unique constraint `(event_id, channel)` |

### Bảng ánh xạ Sự kiện → Template → Kênh
| Domain Event | Template | Kênh gửi | Trigger bởi |
| --- | --- | --- | --- |
| `RegistrationConfirmed` (miễn phí) | `registration_confirmed` | email, in-app | Registration Module |
| `PaymentSucceeded` | `registration_confirmed` | email, in-app | Payment Module |
| `RegistrationExpired` | `registration_expired` | in-app | Hold Expire Worker |
| `WorkshopCancelled` | `workshop_cancelled` | email, in-app | Organizer Admin |
| `WorkshopUpdated` (đổi phòng/giờ) | `workshop_updated` | email, in-app | Organizer Admin |
| `PaymentFailed` | `payment_failed` | in-app | Payment Module |

---

## Error Handling Matrix

| Lỗi | Hành động hệ thống | Phản hồi cho user | Hành động khắc phục |
| --- | --- | --- | --- |
| **Email Provider timeout/5xx** | Delivery → `RETRYING`, tăng `attempt_count` | Sinh viên vẫn thấy QR & in-app | Retry theo backoff; nếu >5 lần → `FAILED_PERMANENT`, alert admin |
| **In-App DB lỗi khi insert** | Delivery → `RETRYING` | Sinh viên không nhận bell notification ngay | Retry độc lập; email vẫn gửi thành công |
| **Template thiếu biến** | Delivery → `FAILED`, ghi `error_reason` | Không gửi kênh này | Fix template/payload; không retry để tránh loop |
| **Worker crash giữa chừng** | Job re-queue bởi BullMQ | Không ảnh hưởng user | Worker restart, đọc lại outbox, tiếp tục xử lý idempotent |
| **Sự kiện bị xử lý trùng** | Vi phạm `UNIQUE(event_id, channel)` → catch error | Không có duplicate | Worker log warning, bỏ qua record trùng |
| **Kênh mới chưa config** | Mapping không tìm thấy channel | Bỏ qua kênh đó | Admin/config cập nhật mapping table |

---

## Tiêu chí chấp nhận (Acceptance Criteria)

- [ ] Đăng ký workshop miễn phí thành công → tạo `notification_event`, sinh delivery cho email + in-app.
- [ ] Thanh toán thành công qua webhook → gửi email & in-app dùng template `registration_confirmed`.
- [ ] Email provider lỗi → delivery chuyển `RETRYING`, retry đúng 5 lần theo lịch backoff; registration KHÔNG bị rollback.
- [ ] Cùng một event KHÔNG tạo quá 1 delivery cho mỗi kênh nhờ `UNIQUE(event_id, channel)`.
- [ ] Workshop bị hủy hoặc đổi giờ → tất cả sinh viên đã đăng ký nhận được email + in-app tương ứng.
- [ ] Thêm kênh Telegram chỉ cần implement `TelegramAdapter` và cập nhật config mapping → KHÔNG sửa code business module.
- [ ] Worker xử lý lại event đã thành công → bỏ qua gracefully, không gửi trùng.
- [ ] Sau 5 lần retry thất bại liên tiếp → delivery chuyển `FAILED_PERMANENT`, ghi log rõ ràng.
- [ ] In-app và email retry độc lập: lỗi kênh này không ảnh hưởng trạng thái kênh kia.
- [ ] Admin có thể truy vấn `notification_deliveries` để audit lịch sử gửi, số lần retry và lý do lỗi.

---

## Ghi chú triển khai (Implementation Notes)

### Interface Channel Adapter chuẩn
```typescript
interface NotificationChannelAdapter {
  channel: 'email' | 'in-app' | 'telegram';
  send(recipient: string, template: string, payload: Record<string, any>): Promise<void>;
}
```

### Luồng Worker xử lý Outbox (pseudo-code)
```typescript
async function processNotificationJob(eventId: string, channel: string) {
  // 1. Lấy event từ outbox
  const event = await prisma.notification_events.findUnique({ where: { id: eventId } });
  
  // 2. Tạo delivery record (idempotent nhờ unique constraint)
  const delivery = await prisma.notification_deliveries.upsert({
    where: { event_id_channel: { event_id: eventId, channel } },
    create: { event_id: eventId, channel, status: 'PENDING', attempt_count: 0 },
    update: { attempt_count: { increment: 1 } },
  });

  // 3. Resolve adapter & send
  const adapter = adapters[delivery.channel];
  await adapter.send(event.recipient_user_id, event.event_type, event.payload);

  // 4. Update status
  await prisma.notification_deliveries.update({
    where: { id: delivery.id },
    data: { status: 'SENT', sent_at: new Date() },
  });
}
```

### Cấu hình Retry trong BullMQ
```typescript
queue.add('notification.send', payload, {
  attempts: 5,
  backoff: {
    type: 'exponential',
    delay: 60_000, // 1 phút, hệ thống sẽ nhân theo công thức 1→5→30→120 phút
  },
  removeOnComplete: true,
  removeOnFail: false, // Giữ lại job failed để audit
});
```

### Bảng Mapping Config (JSON/DB)
```json
{
  "RegistrationConfirmed": {
    "template": "registration_confirmed",
    "channels": ["email", "in-app"]
  },
  "WorkshopUpdated": {
    "template": "workshop_updated",
    "channels": ["email", "in-app", "telegram"]
  }
}
```
*Lưu ý: Khi thêm kênh, chỉ cần mở rộng mảng `channels` trong config và đăng ký adapter mới. Logic query event & dispatch hoàn toàn không thay đổi.*
