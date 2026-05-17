# Delta for Check-in PWA

## Mô tả tính năng
Check-in PWA cho phép nhân sự sự kiện (`CHECKIN_STAFF`) quét mã QR xác nhận sinh viên tham dự workshop trực tiếp tại cửa phòng. Ứng dụng được thiết kế theo triết lý **Offline-First**: preload dữ liệu (roster đăng ký, khóa HMAC xác thực) khi có mạng, lưu trữ cục bộ bằng IndexedDB, cho phép quét, xác thực chữ ký và ghi nhận check-in ngay cả trong vùng không có kết nối. Khi mạng được phục hồi, PWA đồng bộ batch các sự kiện lên backend một cách idempotent. Thiết kế đảm bảo dữ liệu không bị mất khi mất mạng, chống check-in trùng lặp nhờ UUID `event_id` và ràng buộc duy nhất phía server, đồng thời thay thế native mobile app bằng PWA triển khai nhanh, cross-platform.

---

## ADDED Requirements

### Requirement: Chuẩn bị PWA Offline-First (Online Preload)
Hệ thống PHẢI cung cấp cơ chế preload dữ liệu cần thiết khi PWA đang kết nối mạng, đảm bảo nhân sự có thể vận hành hoàn toàn offline sau đó.

#### Scenario: Staff preload dữ liệu workshop thành công
- GIVEN một người dùng đã xác thực có role `CHECKIN_STAFF`
- WHEN staff mở PWA qua HTTPS, đăng nhập và chọn workshop đang diễn ra/sắp diễn ra
- THEN PWA gọi `GET /checkin/preload/:workshopId` để tải roster hợp lệ (`registration_id`, `student_id`, `qr_token_hash`)
- AND tải HMAC secret dùng để xác thực chữ ký QR offline
- AND Service Worker cache toàn bộ app shell (HTML, CSS, JS, icons)
- AND lưu roster, HMAC secret, `device_id` và `sync_cursor` vào IndexedDB
- AND PWA hiển thị trạng thái "Sẵn sàng hoạt động offline"

#### Scenario: Preload thất bại do mất mạng
- GIVEN staff đang ở khu vực không có kết nối internet
- WHEN staff mở PWA lần đầu hoặc cố gắng chọn workshop mới
- THEN PWA không thể tải roster hoặc HMAC secret
- AND hiển thị cảnh báo "Cần kết nối mạng để preload workshop"
- AND không cho phép chuyển sang màn hình quét QR

---

### Requirement: Quét và Xác thực QR Offline
Hệ thống PHẢI cho phép PWA decode, xác thực chữ ký và kiểm tra tính hợp lệ của mã QR ngay trên thiết bị mà không cần gọi API.

#### Scenario: QR hợp lệ được quét offline
- GIVEN PWA đã preload xong và đang ở chế độ offline
- WHEN staff quét mã QR hợp lệ bằng camera
- THEN PWA decode payload và xác thực chữ ký HMAC-SHA256 bằng secret đã lưu trong IndexedDB
- AND kiểm tra `expiresAt` (TTL = `workshop.end_time + 30 phút`) và khớp với local roster
- AND tạo `checkin_event_id` (UUID v4), lưu event với trạng thái `PENDING_SYNC` vào IndexedDB
- AND hiển thị thông báo "Check-in thành công" cho staff ngay lập tức

#### Scenario: QR không chắc chắn hoặc roster thiếu dữ liệu
- GIVEN PWA đang offline với roster đã preload
- WHEN staff quét QR có chữ ký đúng nhưng `registration_id` không tìm thấy trong local roster (do roster cũ hoặc sync chưa kịp)
- THEN PWA không thể xác minh chắc chắn
- AND lưu event với trạng thái `NEEDS_REVIEW` vào IndexedDB
- AND hiển thị cảnh báo "Không xác minh được, đã lưu để xem xét"

#### Scenario: QR sai chữ ký hoặc đã hết hạn
- GIVEN PWA đang ở chế độ offline
- WHEN staff quét QR có chữ ký HMAC không khớp hoặc `expiresAt` đã qua
- THEN PWA từ chối xử lý ngay tại client
- AND KHÔNG tạo bất kỳ event nào trong IndexedDB
- AND hiển thị lỗi "Mã QR không hợp lệ hoặc đã hết hạn"

---

### Requirement: Đồng bộ Idempotent Khi Online
Hệ thống PHẢI đồng bộ các sự kiện check-in offline lên backend theo lô (batch) với cơ chế upsert idempotent, đảm bảo không tạo dữ liệu trùng lặp khi retry.

#### Scenario: PWA khôi phục mạng và sync batch
- GIVEN PWA có các event `PENDING_SYNC` hoặc `NEEDS_REVIEW` trong IndexedDB
- WHEN PWA phát hiện kết nối mạng trở lại hoặc staff bấm nút "Đồng bộ"
- THEN PWA gửi batch event lên `POST /checkin/sync`
- AND backend xác thực role `CHECKIN_STAFF` hoặc `ADMIN`
- AND backend upsert từng event theo `event_id` (dựa trên UNIQUE constraint)
- AND trả về kết quả chi tiết (`ACCEPTED`, `DUPLICATE`, `REJECTED`) cho từng event
- AND PWA cập nhật trạng thái local tương ứng trong IndexedDB

#### Scenario: Sync bị ngắt giữa chừng và retry
- GIVEN PWA đang gửi batch sync thì mất kết nối mạng
- WHEN PWA ổn định kết nối và gửi lại cùng batch
- THEN backend xử lý idempotent: các event đã `ACCEPTED` lần trước được bỏ qua hoặc trả `DUPLICATE`
- AND chỉ các event chưa được xử lý mới được ghi nhận
- AND không phát sinh check-in trùng lặp trong database

---

### Requirement: Xử lý Tranh chấp & Xung đột Check-in
Hệ thống PHẢI giải quyết xung đột khi nhiều thiết bị PWA offline quét cùng một mã QR trước khi đồng bộ.

#### Scenario: Hai thiết bị quét cùng QR offline
- GIVEN hai thiết bị PWA khác nhau cùng preload cùng workshop và cùng quét một QR hợp lệ khi offline
- WHEN cả hai thiết bị sync lên backend
- THEN backend chấp nhận event đầu tiên nhận được (gán `ACCEPTED`)
- AND event thứ hai có cùng `registration_id` + `workshop_id` bị đánh dấu `DUPLICATE`
- AND số lượt check-in thực tế trong hệ thống chỉ tăng 1 lần

---

### Requirement: Tính bền vững dữ liệu cục bộ (Data Persistence)
Hệ thống PHẢI đảm bảo dữ liệu check-in offline tồn tại qua các phiên làm việc của PWA, trừ khi trình duyệt chủ động xóa storage.

#### Scenario: PWA bị đóng khi đang offline
- GIVEN PWA đang lưu các event `PENDING_SYNC` trong IndexedDB
- WHEN staff đóng trình duyệt hoặc ứng dụng PWA hoàn toàn
- THEN toàn bộ event vẫn được bảo toàn trong IndexedDB
- AND khi mở lại, PWA khôi phục trạng thái scan và cho phép tiếp tục sync khi có mạng

#### Scenario: Trình duyệt xóa storage cục bộ
- GIVEN trình duyệt (hoặc hệ điều hành) xóa toàn bộ site data/cache
- WHEN staff mở lại PWA
- THEN toàn bộ roster, HMAC secret và event log bị mất
- AND PWA yêu cầu staff preload lại từ đầu khi có kết nối mạng

---

## Technical Constraints

| Tham số | Giá trị | Lý do / Ghi chú triển khai |
| --- | --- | --- |
| **Giao thức bắt buộc** | HTTPS | Yêu cầu tối thiểu cho Camera API, Service Worker và PWA install |
| **Thư viện quét QR** | `html5-qrcode` (ZXing) | Cross-browser ổn định, không phụ thuộc `Barcode Detection API` native |
| **Định danh event** | UUID v4 mỗi lần quét | Đảm bảo idempotent sync, tránh duplicate khi retry batch |
| **QR Payload & TTL** | `regId + wsId + stuId + expiresAt`, ký HMAC-SHA256, TTL = `ends_at + 30 phút` | Chống giả mạo, cho phép buffer check-in cuối giờ workshop |
| **Local Storage** | IndexedDB (qua thư viện `idb`) | Lưu lượng lớn, hỗ trợ transaction, tồn tại sau restart PWA |
| **Preload Strategy** | Chỉ tải workshop được chọn | Tối ưu băng thông, giảm tải bộ nhớ thiết bị, phù hợp staff phân ca |
| **Source of Truth** | Backend PostgreSQL | PWA chỉ là cache cục bộ; server giải quyết xung đột và là nguồn sự thật cuối cùng |
| **Sync Strategy** | Batch + Upsert theo `event_id` | Giảm số round-trip mạng, xử lý retry an toàn, đảm bảo idempotency |
| **Role cho phép** | `CHECKIN_STAFF`, `ADMIN` | Phân quyền rõ ràng tại API Guard, ngăn truy cập trái phép |
| **Conflict Resolution** | `UNIQUE(registration_id, workshop_id, status='ACCEPTED')` | Server tự động reject duplicate, trả kết quả minh bạch cho PWA |

---

## Integration Points & Dependencies

### Phụ thuộc hệ thống
| Thành phần | Vai trò | Giao thức / Interface |
| --- | --- | --- |
| **Check-in PWA** | Quét QR, lưu local, sync batch, verify offline | IndexedDB, Service Worker, `html5-qrcode`, `idb` |
| **Backend API** | Cung cấp roster/HMAC, nhận & xử lý batch sync | `GET /checkin/preload/:workshopId`, `POST /checkin/sync` |
| **PostgreSQL** | Lưu `checkin_events`, enforce unique `event_id` & conflict rules | Prisma ORM, `upsert`, unique constraints |
| **JWT Auth** | Xác thực nhân sự tại mọi endpoint | `Authorization: Bearer <token>`, `RolesGuard` |
| **HMAC Secret** | Ký QR tại backend, xác thực offline tại PWA | Shared secret per workshop/session, preload qua API an toàn |

### Endpoint API liên quan
| Method | Endpoint | Role yêu cầu | Mô tả |
| --- | --- | --- | --- |
| `GET` | `/checkin/preload/:workshopId` | `CHECKIN_STAFF`, `ADMIN` | Trả về roster + HMAC secret cho workshop chỉ định |
| `POST` | `/checkin/sync` | `CHECKIN_STAFF`, `ADMIN` | Nhận batch event offline, upsert idempotent, trả kết quả chi tiết |
| `GET` | `/checkin/workshops/active` | `CHECKIN_STAFF`, `ADMIN` | Danh sách workshop đang diễn ra/sắp diễn ra để chọn preload |

### Event & State Mapping
| Trạng thái PWA (IndexedDB) | Trạng thái Server (DB) | Ý nghĩa |
| --- | --- | --- |
| `PENDING_SYNC` | `ACCEPTED` | Event hợp lệ, đã được server ghi nhận |
| `PENDING_SYNC` | `DUPLICATE` | Đã có thiết bị khác sync trước đó |
| `NEEDS_REVIEW` | `NEEDS_REVIEW` | Roster offline thiếu/khớp, cần admin xử lý thủ công |
| `SYNC_FAILED` | Không thay đổi | Lỗi mạng/server, giữ lại để retry sau |

---

## Error Handling Matrix

| Lỗi | Hành động hệ thống | Phản hồi cho staff | Hành động khắc phục |
| --- | --- | --- | --- |
| **QR hết hạn/sai HMAC** | PWA reject ngay tại client, không lưu event | "Mã QR không hợp lệ hoặc đã hết hạn" | Sinh viên liên hệ ban tổ chức để cấp QR mới |
| **Sync mất mạng giữa chừng** | PWA giữ nguyên batch trong IndexedDB, đánh dấu `SYNC_FAILED` | "Đồng bộ thất bại, sẽ thử lại khi có mạng" | Cơ chế auto-retry hoặc bấm sync thủ công sau |
| **Hai thiết bị quét cùng QR** | Server nhận event đầu (`ACCEPTED`), event sau → `DUPLICATE` | TB1: "Thành công". TB2: "Đã check-in trước đó" | Tránh tăng ảo số lượng người tham dự, giữ nguyên data |
| **Roster preload lỗi/thiếu** | PWA đánh dấu event `NEEDS_REVIEW` | "Không xác minh được, đã lưu để review" | Admin export log sau sự kiện, đối soát thủ công |
| **Browser xóa IndexedDB** | Dữ liệu local mất hoàn toàn | "Dữ liệu cục bộ không còn, vui lòng preload lại" | Staff install PWA ("Add to Home Screen"), bật persistent storage |
| **Backend 5xx khi sync** | PWA giữ event, không xóa local | "Lỗi máy chủ, đã lưu để sync sau" | Retry tự động khi kết nối ổn định, idempotent đảm bảo an toàn |

---

## Tiêu chí chấp nhận (Acceptance Criteria)

- [ ] Staff preload xong workshop có thể quét QR và lưu event thành công khi tắt WiFi/mất mạng hoàn toàn.
- [ ] PWA bị đóng khi offline vẫn giữ nguyên các event `PENDING_SYNC` trong IndexedDB khi mở lại ứng dụng.
- [ ] Khi có mạng, batch sync lên `POST /checkin/sync` thành công, server upsert idempotent theo `event_id`.
- [ ] Gửi cùng một `event_id` nhiều lần không tạo bản ghi check-in trùng lặp trong database.
- [ ] Hai thiết bị PWA khác nhau quét cùng 1 QR offline → server chỉ ghi nhận 1 check-in (`ACCEPTED`), thiết bị thứ 2 nhận phản hồi `DUPLICATE`.
- [ ] QR có chữ ký HMAC sai hoặc `expiresAt` đã qua bị PWA từ chối ngay tại client, không tạo event.
- [ ] Preload chỉ tải đúng roster của workshop được chọn, không tải toàn bộ dữ liệu hệ thống.
- [ ] PWA bắt buộc chạy trên HTTPS, sử dụng thư viện quét QR cross-browser (`html5-qrcode`), không phụ thuộc API native.
- [ ] Endpoint `POST /checkin/sync` chỉ chấp nhận role `CHECKIN_STAFF` hoặc `ADMIN`, xác thực JWT hợp lệ.
- [ ] Mọi event `NEEDS_REVIEW` được lưu rõ ràng trong IndexedDB và đồng bộ lên server để admin truy vết sau sự kiện.

---

## Ghi chú triển khai (Implementation Notes)

### Cấu trúc Batch Sync Payload
```typescript
interface SyncBatchPayload {
  workshopId: string;
  events: Array<{
    eventId: string;        // UUID v4, unique per scan
    registrationId: string;
    scannedAt: string;      // ISO 8601 timestamp
    status: 'PENDING_SYNC' | 'NEEDS_REVIEW';
    qrPayload?: string;     // Optional: để server verify lại nếu cần
  }>;
}
```

### Luồng Xử lý Sync tại Backend (Pseudo-code)
```typescript
async function syncCheckinEvents(batch: SyncBatchPayload, staffUserId: string) {
  const results = [];
  
  for (const event of batch.events) {
    try {
      // Upsert idempotent theo event_id
      const record = await prisma.checkin_events.upsert({
        where: { event_id: event.eventId },
        create: {
          event_id: event.eventId,
          registration_id: event.registrationId,
          workshop_id: batch.workshopId,
          staff_user_id: staffUserId,
          scanned_at: new Date(event.scannedAt),
          status: 'ACCEPTED',
        },
        update: {
          // Nếu đã tồn tại, đánh dấu là duplicate để báo lại cho PWA
          status: 'DUPLICATE',
        },
      });
      results.push({ eventId: event.eventId, status: record.status === 'ACCEPTED' ? 'ACCEPTED' : 'DUPLICATE' });
    } catch (error) {
      // Log lỗi, trả REJECTED cho event cụ thể, không fail cả batch
      results.push({ eventId: event.eventId, status: 'REJECTED', reason: error.message });
    }
  }
  
  return { processed: results.length, details: results };
}
```

### Logic Xác thực QR Offline tại PWA
```typescript
import { createHmac, timingSafeEqual } from 'crypto'; // hoặc lib tương đương cho browser

function verifyQRSignature(qrToken: string, hmacSecret: string): boolean {
  const [header, payload, signature] = qrToken.split('.');
  const expectedSig = createHmac('sha256', hmacSecret)
    .update(`${header}.${payload}`)
    .digest('hex');
  return timingSafeEqual(Buffer.from(expectedSig), Buffer.from(signature));
}

// Trong PWA scan handler:
if (!verifyQRSignature(decodedQr.token, indexedDB.hmacSecret)) return 'INVALID_SIGNATURE';
if (new Date(decodedQr.expiresAt) < new Date()) return 'EXPIRED';
if (!localRoster.has(decodedQr.registrationId)) return 'NOT_IN_ROSTER';
return 'VALID';
```

### Schema IndexedDB (via `idb`)
```typescript
// stores:
// 1. 'workshops' -> { workshopId, roster: Map<regId, studentInfo>, hmacSecret, preloadedAt }
// 2. 'events' -> { eventId, registrationId, workshopId, scannedAt, status, synced: boolean }
// 3. 'syncCursor' -> { lastSyncTimestamp, pendingCount }
```
