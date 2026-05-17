# Delta for Workshop Catalog

## Mô tả tính năng
Tính năng Workshop Catalog quản lý toàn bộ vòng đời của workshop từ lúc khởi tạo (DRAFT), mở đăng ký (OPEN), đến khi hủy (CANCELLED). Hệ thống cung cấp giao diện công khai để sinh viên xem danh sách và chi tiết workshop, tích hợp cập nhật số chỗ còn lại theo thời gian thực qua SSE, đồng thời cung cấp dashboard thống kê cho Ban tổ chức. Thiết kế đảm bảo phân tách rõ ràng giữa dữ liệu quản trị và dữ liệu hiển thị, phát ra domain event khi có thay đổi quan trọng để các module khác (Notification, Realtime) phản hồi đúng, và duy trì tính nhất quán cao giữa counters (`confirmed_count`, `held_count`) và `capacity`.

---

## ADDED Requirements

### Requirement: Quản lý vòng đời Workshop (CRUD & State Transition)
Hệ thống PHẢI cho phép ORGANIZER và ADMIN tạo, cập nhật, mở đăng ký và hủy workshop theo luồng trạng thái nghiêm ngặt, đồng thời phát domain event khi thay đổi ảnh hưởng đến người dùng.

#### Scenario: Tạo workshop mới
- GIVEN một người dùng đã xác thực có role `ORGANIZER` hoặc `ADMIN`
- WHEN người dùng gửi request `POST /admin/workshops` với đầy đủ trường bắt buộc (`title, speaker_name, room_name, capacity, fee_type, price, starts_at, ends_at`)
- THEN backend tạo workshop với trạng thái mặc định `DRAFT`
- AND trả về đối tượng workshop hoàn chỉnh cho client

#### Scenario: Cập nhật thông tin workshop
- GIVEN một workshop ở trạng thái `DRAFT` hoặc `OPEN`
- WHEN organizer gửi `PATCH /admin/workshops/:id` để thay đổi thông tin (ví dụ: `room_name`, `starts_at`)
- THEN backend cập nhật các trường tương ứng
- AND nếu thay đổi liên quan đến `room_name` hoặc `starts_at/ends_at`, hệ thống phát domain event `WorkshopUpdated`

#### Scenario: Mở đăng ký workshop
- GIVEN một workshop đang ở trạng thái `DRAFT`
- WHEN organizer gửi `POST /admin/workshops/:id/open`
- THEN backend kiểm tra tính hợp lệ của dữ liệu (capacity > 0, thời gian hợp lệ)
- AND chuyển trạng thái workshop sang `OPEN`
- AND cho phép sinh viên bắt đầu gửi request `POST /registrations`

#### Scenario: Hủy workshop
- GIVEN một workshop đang ở trạng thái `OPEN` hoặc `DRAFT`
- WHEN organizer gửi `POST /admin/workshops/:id/cancel`
- THEN backend chuyển trạng thái workshop sang `CANCELLED`
- AND từ chối mọi request đăng ký mới cho workshop này
- AND phát domain event `WorkshopCancelled` để Notification module thông báo cho sinh viên đã đăng ký

---

### Requirement: Danh sách & Chi tiết Workshop công khai
Hệ thống PHẢI cung cấp endpoint công khai để sinh viên xem danh sách workshop đang mở và chi tiết từng workshop, đảm bảo hiệu suất và bảo mật dữ liệu.

#### Scenario: Xem danh sách workshop đang mở
- GIVEN bất kỳ client nào (đã xác thực hoặc ẩn danh)
- WHEN client gọi `GET /workshops` với phân trang
- THEN backend trả về danh sách các workshop có `status = OPEN`
- AND mỗi mục chỉ bao gồm các trường cơ bản: `id, title, speaker_name, room_name, starts_at, ends_at, capacity, confirmed_count, held_count, fee_type, price, summary_status`
- AND trường `ai_summary` được loại trừ khỏi danh sách để tối ưu payload

#### Scenario: Xem chi tiết workshop
- GIVEN bất kỳ client nào
- WHEN client gọi `GET /workshops/:id`
- THEN backend trả về đầy đủ thông tin workshop
- AND bao gồm `room_map_url` và `ai_summary` (nếu `summary_status = AI_GENERATED` hoặc `ADMIN_EDITED`)
- AND trả về `404 Not Found` nếu workshop không tồn tại hoặc đã bị xóa

---

### Requirement: Cập nhật số chỗ thời gian thực qua SSE
Hệ thống PHẢI đẩy thông tin số chỗ còn lại (`remaining_seats`, `held_count`, `confirmed_count`) đến client theo thời gian thực thông qua Server-Sent Events (SSE), phân phối đồng bộ qua Redis Pub/Sub.

#### Scenario: Client nhận cập nhật số chỗ realtime
- GIVEN một hoặc nhiều sinh viên đã kết nối SSE đến `GET /workshops/:id/seats`
- WHEN có thay đổi `confirmed_count` hoặc `held_count` trong database (do đăng ký, thanh toán, hoặc expire hold)
- THEN Backend API phát event nội bộ và publish payload `{ remaining_seats, held_count, confirmed_count }` tới Redis channel `ws:{workshop_id}:seats`
- AND tất cả API instance đang subscribe đều nhận event và push xuống client SSE
- AND client cập nhật UI mà không cần reload trang

#### Scenario: Kết nối SSE bị gián đoạn
- GIVEN một client đang lắng nghe SSE stream
- WHEN mạng bị mất hoặc trình duyệt đóng tab
- THEN `EventSource` tự động cố gắng reconnect
- AND sau khi kết nối lại, client gọi `GET /workshops/:id` để lấy snapshot số chỗ hiện tại
- AND tiếp tục lắng nghe event realtime mới

#### Scenario: Redis Pub/Sub tạm ngưng
- GIVEN Redis Pub/Sub bị mất kết nối
- WHEN seat count thay đổi trong database
- THEN luồng đăng ký vẫn hoạt động bình thường qua PostgreSQL transaction
- AND cập nhật realtime bị degrade (client không nhận push mới)
- AND hệ thống tự phục hồi khi Redis hoạt động trở lại mà không cần can thiệp thủ công

---

### Requirement: Thống kê & Báo cáo cho Ban tổ chức
Hệ thống PHẢI cung cấp API thống kê chi tiết cho ORGANIZER và ADMIN để theo dõi mức độ tham gia và hiệu suất workshop.

#### Scenario: Organizer xem thống kê workshop
- GIVEN một người dùng đã xác thực có role `ORGANIZER` hoặc `ADMIN`
- WHEN người dùng gọi `GET /admin/workshops/:id/stats`
- THEN backend tính toán và trả về đối tượng thống kê:
  - `total_registrations`: tổng số registration (bao gồm cả expired/cancelled)
  - `confirmed_count`: số chỗ đã xác nhận
  - `pending_payment_count`: số chỗ đang giữ chờ thanh toán
  - `checkin_count`: số lượt đã check-in thực tế
  - `capacity`: sức chứa tối đa
  - `utilization_pct`: `(confirmed_count / capacity) * 100`

---

## Technical Constraints

| Tham số | Giá trị | Lý do / Ghi chú triển khai |
| --- | --- | --- |
| **Status Enum** | `DRAFT`, `OPEN`, `CANCELLED` | Luồng trạng thái tuyến tính, không cho phép quay lại DRAFT từ OPEN/CANCELLED |
| **Public Listing Filter** | Chỉ trả về `status = OPEN` | Đảm bảo sinh viên chỉ thấy workshop đang mở đăng ký |
| **Detail Payload** | Bao gồm `room_map_url`, `ai_summary` | Cung cấp đầy đủ thông tin cho quyết định đăng ký |
| **List Payload** | Loại trừ `ai_summary` | Giảm kích thước payload cho danh sách phân trang |
| **SSE Channel** | `ws:{workshop_id}:seats` | Pattern rõ ràng, dễ subscribe/unsubscribe theo workshop |
| **Realtime Source** | PostgreSQL transaction là source of truth | SSE chỉ là view projection, không dùng để validate capacity |
| **Rate Limit Public** | 60 burst / 10 refill/s (theo IP) | Bảo vệ endpoint danh sách khỏi bot/crawl quá tải |
| **Stats Calculation** | Query aggregate trực tiếp hoặc cache ngắn hạn | Đảm bảo số liệu chính xác theo thời điểm gọi |
| **Event Emission** | `WorkshopUpdated`, `WorkshopCancelled` | Decoupled communication với Notification module |
| **Capacity Invariant** | `confirmed_count + held_count ≤ capacity` | Ràng buộc cứng được enforce ở Registration module, Catalog chỉ đọc và hiển thị |

---

## Integration Points & Dependencies

### Phụ thuộc hệ thống
| Thành phần | Vai trò | Giao thức / Interface |
| --- | --- | --- |
| **Registration Module** | Cập nhật `confirmed_count`, `held_count` | DB transaction, emit seat change event |
| **Redis Pub/Sub** | Phân phối event số chỗ đến các API instance | `ioredis`, channel `ws:{workshop_id}:seats` |
| **Notification Module** | Nhận event `WorkshopCancelled/Updated` | Outbox pattern, gửi thông báo cho registrants |
| **AI Summary Module** | Ghi `ai_summary` và `summary_status` | Direct DB update trên bảng `workshops` |
| **Student Web App** | Hiển thị danh sách, chi tiết, kết nối SSE | `fetch`, `EventSource`, React UI |
| **Admin Web App** | CRUD, mở/hủy workshop, xem stats | `fetch`, React Admin UI, RBAC guards |

### Endpoint API liên quan
| Method | Endpoint | Role yêu cầu | Mô tả |
| --- | --- | --- | --- |
| `POST` | `/admin/workshops` | `ORGANIZER`, `ADMIN` | Tạo workshop mới (status `DRAFT`) |
| `PATCH` | `/admin/workshops/:id` | `ORGANIZER`, `ADMIN` | Cập nhật thông tin, phòng, giờ |
| `POST` | `/admin/workshops/:id/open` | `ORGANIZER`, `ADMIN` | Chuyển `DRAFT` → `OPEN` |
| `POST` | `/admin/workshops/:id/cancel` | `ORGANIZER`, `ADMIN` | Chuyển → `CANCELLED`, emit event |
| `GET` | `/workshops` | Public | Danh sách phân trang workshop `OPEN` |
| `GET` | `/workshops/:id` | Public | Chi tiết workshop + `ai_summary` |
| `GET` | `/workshops/:id/seats` | Public (SSE) | Stream cập nhật số chỗ realtime |
| `GET` | `/admin/workshops/:id/stats` | `ORGANIZER`, `ADMIN` | Thống kê đăng ký, check-in, utilization |

### Event Domain phát ra
| Event | Trigger bởi | Payload chính | Consumer |
| --- | --- | --- | --- |
| `WorkshopUpdated` | `PATCH /admin/workshops/:id` (đổi phòng/giờ) | `{ workshop_id, old_values, new_values, updated_by }` | Notification Module, Audit Log |
| `WorkshopCancelled` | `POST /admin/workshops/:id/cancel` | `{ workshop_id, cancelled_at, cancelled_by }` | Notification Module, Registration (lock new reg) |
| `SeatCountChanged` | Registration/Payment/Expire commit | `{ workshop_id, confirmed_count, held_count, remaining_seats }` | SSE Module (push to clients) |

---

## Error Handling Matrix

| Lỗi | Hành động hệ thống | Phản hồi cho user | Hành động khắc phục |
| --- | --- | --- | --- |
| **Thiếu trường bắt buộc khi tạo** | Validation middleware chặn | `400 Bad Request` + chi tiết trường thiếu | Organizer bổ sung đầy đủ dữ liệu |
| **Chuyển trạng thái không hợp lệ** | Ví dụ: OPEN → DRAFT hoặc CANCELLED → OPEN | `409 Conflict` + message "Invalid status transition" | Hệ thống chỉ cho phép luồng tuyến tính DRAFT → OPEN → CANCELLED |
| **SSE disconnect** | Server cleanup subscription | UI tạm thời không nhảy số chỗ | `EventSource` auto-reconnect + fetch snapshot REST |
| **Redis Pub/Sub down** | Publish fail, log warning | Realtime degrade, danh sách/chi tiết vẫn load được | Redis restart, tự động subscribe lại, không mất dữ liệu DB |
| **Truy cập stats sai role** | `RolesGuard` chặn | `403 Forbidden` | Chỉ `ORGANIZER`/`ADMIN` mới có quyền |
| **Workshop không tồn tại** | Query `workshops` trả null | `404 Not Found` | Kiểm tra ID trước khi gọi API |
| **Capacity = 0 hoặc âm** | Validation chặn tại `DRAFT`/`OPEN` | `400 Bad Request` | Organizer sửa capacity > 0 |

---

## Tiêu chí chấp nhận (Acceptance Criteria)

- [ ] Organizer tạo workshop thành công → trạng thái `DRAFT`, không hiển thị ở danh sách công khai.
- [ ] `POST /admin/workshops/:id/open` chuyển workshop sang `OPEN` → xuất hiện trong `GET /workshops`.
- [ ] `POST /admin/workshops/:id/cancel` chuyển sang `CANCELLED` → chặn đăng ký mới, phát event `WorkshopCancelled`.
- [ ] `GET /workshops` trả về danh sách phân trang, chỉ chứa workshop `OPEN`, không bao gồm `ai_summary`.
- [ ] `GET /workshops/:id` trả về chi tiết đầy đủ, bao gồm `room_map_url` và `ai_summary` (nếu có).
- [ ] Nhiều client kết nối SSE nhận được cập nhật `remaining_seats` trong vòng vài giây sau khi có registration/payment/expire.
- [ ] Client mất kết nối SSE có thể tự động reconnect và đồng bộ snapshot mới nhất.
- [ ] `GET /admin/workshops/:id/stats` trả về chính xác `total_registrations`, `confirmed_count`, `pending_payment_count`, `checkin_count`, `utilization_pct`.
- [ ] Rate limit public listing hoạt động đúng: vượt 60 burst → `429` + `Retry-After: 5s`.
- [ ] Thay đổi phòng/giờ phát ra event `WorkshopUpdated`, Notification module nhận và xử lý độc lập.
- [ ] `ai_summary` chỉ được tính toán và lưu bởi AI Module, Catalog module chỉ đọc và hiển thị.
- [ ] Database không cho phép `confirmed_count + held_count > capacity` (enforce ở Registration, Catalog hiển thị phản ánh đúng).

---

## Ghi chú triển khai (Implementation Notes)

### 1. State Machine & Transition Guard
```typescript
enum WorkshopStatus { DRAFT = 'DRAFT', OPEN = 'OPEN', CANCELLED = 'CANCELLED' }

const VALID_TRANSITIONS = {
  DRAFT: ['OPEN', 'CANCELLED'],
  OPEN: ['CANCELLED'],
  CANCELLED: [] // Terminal state
};

async function transitionWorkshop(id: string, targetStatus: WorkshopStatus) {
  const workshop = await prisma.workshops.findUnique({ where: { id } });
  if (!VALID_TRANSITIONS[workshop.status].includes(targetStatus)) {
    throw new ConflictException(`Cannot transition from ${workshop.status} to ${targetStatus}`);
  }
  return await prisma.workshops.update({ where: { id }, data: { status: targetStatus } });
}
```

### 2. SSE Endpoint & Redis Pub/Sub
```typescript
@Sse('/workshops/:id/seats')
seatStream(@Param('id') workshopId: string, @Req() req: any): Observable<MessageEvent> {
  const channel = `ws:${workshopId}:seats`;
  return new Observable(observer => {
    const listener = (message: string) => observer.next({ data: message });
    
    redis.subscribe(channel, () => {
      redis.on('message', listener);
    });

    req.on('close', () => {
      redis.unsubscribe(channel);
      redis.off('message', listener);
      observer.complete();
    });
  });
}
// Khi Registration commit thành công:
// redis.publish(`ws:${workshopId}:seats`, JSON.stringify({ remaining_seats, held_count, confirmed_count }));
```

### 3. Stats Calculation (Optimized Query)
```typescript
async function getWorkshopStats(workshopId: string) {
  const workshop = await prisma.workshops.findUnique({ where: { id: workshopId } });
  const checkinCount = await prisma.checkin_events.count({
    where: { workshop_id: workshopId, status: 'ACCEPTED' }
  });
  
  return {
    total_registrations: await prisma.registrations.count({ where: { workshop_id: workshopId } }),
    confirmed_count: workshop.confirmed_count,
    pending_payment_count: workshop.held_count,
    checkin_count: checkinCount,
    capacity: workshop.capacity,
    utilization_pct: workshop.capacity > 0 
      ? Math.round((workshop.confirmed_count / workshop.capacity) * 100) 
      : 0
  };
}
```

### 4. Schema Ràng buộc (Prisma)
```prisma
model workshops {
  id                String   @id @default(uuid())
  title             String
  speaker_name      String
  room_name         String
  room_map_url      String?
  capacity          Int
  confirmed_count   Int      @default(0)
  held_count        Int      @default(0)
  fee_type          String   // FREE, PAID
  price             Decimal?
  starts_at         DateTime
  ends_at           DateTime
  status            String   // DRAFT, OPEN, CANCELLED
  summary_status    String?  // PENDING, AI_GENERATED, ADMIN_EDITED, SUMMARY_FAILED
  ai_summary        String?  @db.Text
  
  registrations     registration[]
  checkin_events    checkin_events[]
  documents         workshop_documents[]

  @@index([status]) // Tối ưu query GET /workshops
}
```

### 5. Lưu ý Performance & Cache
- **Danh sách phân trang:** Dùng `skip`/`take` hoặc cursor-based pagination. Thêm index trên `status` và `starts_at`.
- **SSE Connection Management:** Giới hạn số lượng kết nối SSE trên mỗi API instance (ví dụ: 5000/client node) để tránh cạn kiệt file descriptor. Dùng `ulimit` và connection pool tuning.
- **Stats Caching (Optional):** Nếu truy vấn stats quá nặng, có thể cache kết quả `GET /admin/workshops/:id/stats` trong Redis với TTL `5 phút`, invalidate khi có registration/checkin mới.
