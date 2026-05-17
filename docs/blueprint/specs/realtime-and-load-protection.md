# Delta for Realtime & Load Protection

## Mô tả tính năng
Tính năng Realtime & Load Protection kết hợp hai lớp bảo vệ then chốt: (1) đẩy cập nhật số chỗ workshop theo thời gian thực qua **SSE + Redis Pub/Sub**, và (2) bảo vệ backend khỏi tải đột biến 12.000 sinh viên thông qua **Virtual Queue**, **Token Bucket Rate Limiting** và **Idempotency Key**. Thiết kế đảm bảo công bằng phân phối chỗ ngồi, ngăn chặn API bị quá tải hoặc spam, đồng thời duy trì hoạt động ổn định ngay cả khi một số thành phần hạ tầng (như Redis Pub/Sub) gặp sự cố tạm thời.

---

## ADDED Requirements

### Requirement: Realtime Seat Count via SSE & Redis Pub/Sub
Hệ thống PHẢI đẩy số chỗ còn lại của workshop đến client theo thời gian thực thông qua Server-Sent Events (SSE), sử dụng Redis Pub/Sub để phân phối sự kiện đồng bộ giữa các API instance.

#### Scenario: Số chỗ cập nhật sau khi có đăng ký hoặc hết hạn giữ chỗ
- GIVEN một hoặc nhiều sinh viên đang kết nối SSE tới trang chi tiết workshop
- WHEN registration được tạo, hold slot hết hạn hoặc thanh toán thành công làm thay đổi seat count
- THEN Backend API phát event nội bộ thay đổi số chỗ
- AND publish payload `{ remaining_seats, held_count, confirmed_count }` tới Redis Pub/Sub channel `ws:{workshop_id}:seats`
- AND tất cả API instance đang subscribe channel đều nhận event và push dữ liệu mới đến client SSE
- AND client tự động cập nhật UI mà không cần reload trang

#### Scenario: Kết nối SSE bị mất
- GIVEN một client đang lắng nghe SSE stream
- WHEN kết nối mạng bị gián đoạn hoặc trình duyệt đóng tab
- THEN client tự động reconnect (sử dụng `EventSource` native hoặc retry logic)
- AND sau khi kết nối lại, client fetch snapshot hiện tại từ `GET /workshops/:id` để đồng bộ UI
- AND tiếp tục lắng nghe event realtime mới

#### Scenario: Redis Pub/Sub tạm thời không khả dụng
- GIVEN Redis Pub/Sub bị mất kết nối hoặc crash
- WHEN seat count thay đổi trong database
- THEN registration/hold/expire vẫn được xử lý đúng qua PostgreSQL transaction
- AND cập nhật realtime bị degrade (client không nhận push mới ngay)
- AND hệ thống tự động phục hồi realtime khi Redis Pub/Sub hoạt động trở lại mà không cần can thiệp thủ công

---

### Requirement: Virtual Queue cho giờ cao điểm
Hệ thống PHẢI triển khai hàng đợi ảo (Virtual Queue) để điều tiết luồng request đăng ký, ngăn 12.000 sinh viên truy cập đồng thời trực tiếp vào Registration API.

#### Scenario: Sinh viên xin queue token khi mở đăng ký
- GIVEN sinh viên vào màn hình đăng ký trong khung giờ cao điểm
- WHEN frontend gọi `POST /workshops/:id/queue-token`
- THEN backend tạo key Redis `qt:{user_id}:{workshop_id}` chứa `{issued_at, expires_at}`
- AND TTL của key được đặt chính xác là 120 giây
- AND trả về queue token cho client

#### Scenario: Queue token hợp lệ được sử dụng để đăng ký
- GIVEN sinh viên sở hữu queue token hợp lệ và chưa bị dùng
- WHEN sinh viên gửi request `POST /registrations` kèm token
- THEN backend kiểm tra token tồn tại, đúng `user_id` và `workshop_id`, chưa hết hạn
- AND xử lý luồng đăng ký bình thường
- AND ngay sau khi transaction commit thành công, backend `DEL` token khỏi Redis (one-time-use)

#### Scenario: Queue token hết hạn hoặc bị lạm dụng
- GIVEN queue token đã quá TTL 120 giây hoặc được gửi bởi user/workshop khác với payload gốc
- WHEN sinh viên cố gắng đăng ký với token này
- THEN backend từ chối request, trả về mã lỗi `QUEUE_TOKEN_EXPIRED` hoặc `INVALID_TOKEN`
- AND sinh viên phải xin token mới (quay lại đầu hàng đợi ảo)

---

### Requirement: Token Bucket Rate Limiting
Hệ thống PHẢI áp dụng giới hạn tốc độ theo thuật toán Token Bucket, lưu trữ trạng thái trên Redis và cấu hình ngưỡng riêng biệt theo từng nhóm endpoint.

#### Scenario: Giới hạn áp dụng cho endpoint công khai (xem lịch)
- GIVEN một client (theo IP) đang browse danh sách workshop
- WHEN client gửi > 60 request burst với refill rate 10 token/s
- THEN backend trả `429 Too Many Requests`
- AND kèm header `Retry-After: 5s`

#### Scenario: Giới hạn áp dụng cho endpoint đăng ký workshop
- GIVEN một sinh viên cố gắng đăng ký cho cùng một workshop
- WHEN sinh viên gửi > 5 request burst với refill rate 1 token/30s (keyed by `user_id+workshop_id`)
- THEN backend trả `429 Too Many Requests`
- AND kèm header `Retry-After: 30s`

#### Scenario: Giới hạn áp dụng cho endpoint admin & đăng nhập
- GIVEN admin thực hiện thao tác quản trị hoặc user đăng nhập liên tục
- WHEN vượt ngưỡng cấu hình (Admin: 30 burst / 5 token/s keyed by `user_id`; Login: 10 burst / 1 token/s keyed by `ip`)
- THEN backend trả `429 Too Many Requests` với `Retry-After` tương ứng (5s cho admin, 10s cho login)

---

### Requirement: Idempotent Registration Retry
Hệ thống PHẢI xử lý an toàn các request retry do timeout mạng hoặc client click nhiều lần, đảm bảo không tạo bản ghi trùng lặp hoặc thay đổi seat count sai lệch.

#### Scenario: Client retry request sau khi timeout
- GIVEN client đã gửi `POST /registrations` nhưng không nhận phản hồi do network timeout
- WHEN client gửi lại request chính xác với cùng header `Idempotency-Key` (UUID v4) trong vòng 24 giờ
- THEN backend kiểm tra `registrations.idempotency_key` trong database
- AND trả về kết quả registration đã tạo lần đầu tiên
- AND KHÔNG tạo registration mới, KHÔNG thay đổi `confirmed_count`/`held_count`, KHÔNG gọi lại payment gateway

#### Scenario: Idempotency-Key quá hạn 24 giờ
- GIVEN `Idempotency-Key` đã tồn tại trong DB quá 24 giờ
- WHEN client gửi request mới với cùng key
- THEN backend xử lý như một request đăng ký hoàn toàn mới (key cũ được archive hoặc bỏ qua)
- AND cho phép sinh viên đăng ký lại nếu workshop còn chỗ

---

## Technical Constraints

| Tham số | Giá trị | Lý do / Ghi chú triển khai |
| --- | --- | --- |
| **Realtime Transport** | SSE (Server-Sent Events) | Push một chiều, nhẹ, phù hợp seat update; client hỗ trợ `EventSource` native |
| **Cross-instance Sync** | Redis Pub/Sub | Đảm bảo nhiều API instance đều nhận và push event đồng bộ |
| **Realtime Channel** | `ws:{workshop_id}:seats` | Key pattern rõ ràng, tự động unsubscribe khi client disconnect |
| **Source of Truth** | PostgreSQL Transaction | SSE chỉ là view; quyết định cuối cùng về capacity nằm ở DB |
| **Queue Token TTL** | 120 giây | Cửa sổ đủ để sinh viên hoàn tất thao tác đăng ký, tránh tồn đọng token rác |
| **Queue Token Binding** | `qt:{user_id}:{workshop_id}` | Gắn chặt với danh tính & workshop, ngăn chia sẻ/lạm dụng token |
| **Rate Limit Algorithm** | Token Bucket (Redis) | Cho phép burst hợp lý khi mở trang, nhưng kiểm soát tốc độ trung bình |
| **Rate Limit Keying** | `ip` (public/login), `user_id+workshop_id` (registration), `user_id` (admin) | Phân tầng chính xác theo ngữ cảnh sử dụng |
| **Idempotency TTL** | 24 giờ | Đủ cho session đăng ký; sau đó hệ thống cho phép retry/new attempt |
| **Retry-After Header** | Bắt buộc khi trả `429` | Giúp client/frontend tính toán thời gian chờ hợp lý, tránh spam loop |
| **Graceful Degradation** | Redis Pub/Sub down → realtime mất, core API vẫn chạy | Đảm bảo tính sẵn sàng của hệ thống đăng ký không phụ thuộc tuyệt đối vào cache |

---

## Integration Points & Dependencies

### Phụ thuộc hệ thống
| Thành phần | Vai trò | Giao thức / Interface |
| --- | --- | --- |
| **Student Web App** | Kết nối SSE, quản lý queue token, sinh `Idempotency-Key`, hiển thị `Retry-After` | `EventSource` API, `fetch` headers, auto-reconnect logic |
| **Backend API (NestJS)** | Endpoint SSE, Rate Limit Guard, Queue Token Service, Idempotency Interceptor | REST + SSE, `ioredis`, `Prisma`, Guards/Interceptors |
| **Redis 7 (Docker)** | Pub/Sub distribution, Token Bucket state, Virtual Queue storage | `redis-server`, Lua scripts (atomic check-and-decrement) |
| **Registration Module** | Emit seat change events, validate & store idempotency keys | Internal event bus, DB transaction scope |

### Redis Key Patterns
| Mục đích | Key Pattern | TTL | Ghi chú |
| --- | --- | --- | --- |
| Token Bucket Rate Limit | `rl:{key_type}:{identifier}` | 60s (tự động expire khi không dùng) | Lưu `tokens, last_refill_ts` |
| Virtual Queue Token | `qt:{user_id}:{workshop_id}` | 120s | Payload JSON, xóa ngay sau khi dùng |
| SSE Pub/Sub Channel | `ws:{workshop_id}:seats` | — (managed by pub/sub) | Publish event, subscribe per API instance |

---

## Error Handling Matrix

| Lỗi | Hành động hệ thống | Phản hồi cho user/client | Hành động khắc phục |
| --- | --- | --- | --- |
| **SSE disconnect** | Server cleanup subscription, client auto-reconnect | UI tạm thời không update, hiển thị snapshot cũ | Client fetch `GET /workshops/:id` để sync, reconnect SSE |
| **Redis Pub/Sub crash** | Event publish fail, API vẫn xử lý registration | Realtime số chỗ không nhảy, sinh viên vẫn đăng ký được | Redis restart tự động, subscription tái lập, không mất dữ liệu DB |
| **Queue token hết hạn** | Token không tồn tại trong Redis hoặc TTL expire | `QUEUE_TOKEN_EXPIRED` → yêu cầu xin token mới | Frontend hiển thị loading queue, gọi lại `POST /queue-token` |
| **Client vượt Rate Limit** | Token Bucket = 0, middleware chặn request | `429 Too Many Requests` + `Retry-After: Xs` | Client đợi đúng thời gian header chỉ định trước khi retry |
| **Idempotency retry timeout** | Backend tìm thấy key trong DB (≤24h) | Trả về `200 OK` + registration cũ | Không tạo record mới, giữ nguyên seat count, tránh oversell |
| **Redis hoàn toàn down** | Rate limit & queue token bypass (fallback mode) | Hệ thống vào chế độ bảo vệ cơ bản (chỉ dựa DB lock) | Ưu tiên khôi phục Redis; DB transaction vẫn chống oversell |

---

## Tiêu chí chấp nhận (Acceptance Criteria)

- [ ] Nhiều client kết nối SSE nhận được cập nhật `remaining_seats` trong vòng vài giây sau khi có registration/hold/expire.
- [ ] Client mất kết nối SSE có thể tự động reconnect và fetch snapshot mới nhất để đồng bộ UI.
- [ ] Khi Redis Pub/Sub tạm ngưng, API vẫn xử lý đăng ký bình thường qua DB; realtime chỉ bị degrade tạm thời.
- [ ] 12.000 sinh viên truy cập cùng lúc không được gửi thẳng toàn bộ request vào `POST /registrations` mà phải đi qua Virtual Queue.
- [ ] Queue token chỉ dùng được 1 lần, gắn đúng `user_id` và `workshop_id`, tự động xóa khỏi Redis sau khi dùng thành công.
- [ ] Queue token hết hạn 120s hoặc bị lạm dụng bị từ chối ngay, client phải xin token mới.
- [ ] Rate limit Token Bucket hoạt động chính xác theo 4 tier (public, login, registration, admin) với burst/refill đúng cấu hình.
- [ ] Client vượt ngưỡng rate limit nhận `429` kèm header `Retry-After` chính xác theo tier.
- [ ] Retry request với cùng `Idempotency-Key` trong 24h trả về kết quả cũ, không tạo registration mới, không thay đổi seat count.
- [ ] `Idempotency-Key` không bị lưu vô hạn; cơ chế archive/cleanup hoặc TTL 24h được áp dụng để tránh phình DB.
- [ ] Frontend hiển thị thông báo trạng thái queue/loading phù hợp khi chờ token hoặc khi bị rate limit.

---

## Ghi chú triển khai (Implementation Notes)

### 1. Token Bucket Rate Limiter (Redis + Lua Script)
Để đảm bảo tính nguyên tử (atomic) và tránh race condition khi nhiều request cùng check-consume token, cần dùng Lua script:
```typescript
const LUA_SCRIPT = `
local tokens = tonumber(redis.call('hget', KEYS[1], 'tokens'))
local last_refill = tonumber(redis.call('hget', KEYS[1], 'last_refill_ts'))
local now = math.floor(redis.call('time')[1])
local elapsed = now - last_refill
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])

tokens = math.min(capacity, tokens + (elapsed * refill_rate))
if tokens >= 1 then
  tokens = tokens - 1
  redis.call('hset', KEYS[1], 'tokens', tokens, 'last_refill_ts', now)
  redis.call('expire', KEYS[1], 60)
  return 1
else
  redis.call('expire', KEYS[1], 60)
  return 0
end
`;
// Middleware gọi: redis.eval(LUA_SCRIPT, 1, key, capacity, refill_rate)
// Trả 1 → cho qua. Trả 0 → throw 429 + Retry-After.
```

### 2. SSE Endpoint & Redis Pub/Sub Subscription
```typescript
@Sse('/workshops/:id/seats')
async seatStream(@Param('id') workshopId: string, @Req() req: any): Promise<Observable<MessageEvent>> {
  const channel = `ws:${workshopId}:seats`;
  const observer = new Observable<MessageEvent>(subscriber => {
    const callback = (msg: string) => subscriber.next({ data: JSON.parse(msg) });
    redis.subscribe(channel, () => {
      redis.on('message', callback);
    });
    req.on('close', () => {
      redis.unsubscribe(channel);
      redis.removeListener('message', callback);
      subscriber.complete();
    });
  });
  return observer;
}
```

### 3. Idempotency Interceptor Flow
```typescript
async function handleIdempotency(req: Request, next: CallHandler) {
  const key = req.headers['idempotency-key'];
  if (!key) return next.handle();

  const cached = await prisma.registrations.findUnique({ where: { idempotency_key: key } });
  if (cached) return cached; // Trả ngay kết quả cũ, bypass toàn bộ logic

  const result = await next.handle().toPromise();
  return result; // Logic tạo registration sẽ lưu key vào DB trong transaction
}
```

### 4. Virtual Queue Token Issuance
```typescript
async function issueQueueToken(userId: string, workshopId: string) {
  const key = `qt:${userId}:${workshopId}`;
  const payload = JSON.stringify({ issued_at: Date.now(), expires_at: Date.now() + 120_000 });
  await redis.set(key, payload, 'EX', 120);
  return { token: key, expires_in: 120 };
}
// Validation: redis.get(key) → check payload.user_id/workshop_id → redis.del(key) sau commit
```

