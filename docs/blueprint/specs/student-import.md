# Delta for Student Import

## Mô tả tính năng
Tính năng Student Import xử lý việc đồng bộ dữ liệu sinh viên từ hệ thống cũ của trường thông qua file CSV xuất ban đêm. Do hệ thống cũ không cung cấp API, luồng nhập dữ liệu áp dụng mô hình **Staging-to-Production** kết hợp **Batch Sequential Pipeline**: file CSV được lưu vào object storage, worker parse và validate từng dòng vào bảng trung gian (`student_import_rows`), kiểm tra tỷ lệ lỗi, sau đó promotion atomic vào bảng `students`. Thiết kế đảm bảo hệ thống không bị chặn khi import chạy, file lỗi không làm hỏng dữ liệu production, checksum chống import trùng, và admin có thể truy vết chi tiết lỗi theo từng dòng.

---

## ADDED Requirements

### Requirement: Nhập dữ liệu sinh viên từ CSV (CSV-Based Data Import)
Hệ thống PHẢI phát hiện và xử lý file CSV mới từ hệ thống cũ theo lịch cố định, sử dụng checksum để tránh xử lý trùng lặp.

#### Scenario: Scheduler phát hiện file CSV mới
- GIVEN hệ thống cũ export file CSV vào bucket `student-imports` của Supabase Storage
- WHEN cron scheduler chạy định kỳ (2:00 AM hàng ngày)
- THEN scheduler tính SHA-256 checksum của file
- AND kiểm tra checksum trong bảng `student_import_batches`
- AND nếu chưa tồn tại, tạo record `PARSING` và đẩy job vào BullMQ queue `student-import`
- AND nếu checksum đã tồn tại, bỏ qua hoặc đánh dấu `DUPLICATE`

#### Scenario: Worker parse và validate file hợp lệ
- GIVEN một file CSV mới được phát hiện và job đã được enqueue
- WHEN worker đọc file và parse theo batch nhỏ
- THEN worker kiểm tra header bắt buộc (`student_code, email, full_name, faculty`)
- AND validate từng dòng: định dạng email (RFC 5322), `student_code` không rỗng/ký tự hợp lệ, phát hiện trùng trong cùng batch
- AND ghi kết quả vào `student_import_rows` với `row_status` = `VALID`, `ERROR`, hoặc `DUPLICATE` kèm `error_message` chi tiết

---

### Requirement: Promotion từ Staging sang Production (Staging-to-Production Promotion)
Hệ thống PHẢI chỉ cập nhật bảng `students` thông qua transaction atomic sau khi validation đạt ngưỡng an toàn. Tuyệt đối KHÔNG insert trực tiếp vào bảng production.

#### Scenario: Batch có tỷ lệ lỗi dưới ngưỡng được promote
- GIVEN batch đã parse xong và tỷ lệ lỗi ≤ 20% (`error_threshold_pct`)
- WHEN worker kích hoạt giai đoạn promotion
- THEN worker mở một database transaction duy nhất
- AND thực hiện upsert tất cả dòng `VALID` vào bảng `students` (`ON CONFLICT student_code DO UPDATE`)
- AND cập nhật `source_batch_id` và `last_seen_in_import_at` cho các record liên quan
- AND đánh dấu các sinh viên cũ không còn trong CSV mới là `INACTIVE` (theo chính sách)
- AND commit transaction, chuyển batch sang `PROMOTED`, ghi báo cáo admin

#### Scenario: Batch vượt ngưỡng lỗi bị reject
- GIVEN batch có thiếu cột bắt buộc hoặc tỷ lệ lỗi > 20%
- WHEN worker đánh giá điều kiện promotion
- THEN batch chuyển trạng thái `REJECTED`
- AND bảng `students` KHÔNG bị thay đổi
- AND admin có thể xem báo cáo lỗi chi tiết trong `student_import_rows`

---

### Requirement: Xử lý Crash và Phục hồi Worker (Worker Crash Recovery)
Hệ thống PHẢI đảm bảo an toàn dữ liệu khi worker bị crash trong quá trình parse hoặc promotion.

#### Scenario: Worker crash khi đang parse
- GIVEN worker đang đọc CSV và ghi vào staging table thì bị crash
- WHEN worker được restart hoặc scheduler chạy lại
- THEN worker tiếp tục parse từ đầu hoặc từ checkpoint (nếu hỗ trợ)
- AND staging table được dọn dẹp hoặc ghi đè an toàn
- AND không phát sinh record rác trong production

#### Scenario: Worker crash khi đang promote
- GIVEN worker đang thực hiện transaction promotion thì bị crash
- WHEN transaction bị ngắt giữa chừng
- THEN PostgreSQL tự động rollback toàn bộ thay đổi
- AND bảng `students` giữ nguyên trạng thái trước khi promote
- AND batch giữ trạng thái trung gian, cho phép retry an toàn

---

### Requirement: Xử lý sinh viên không còn trong file mới (Inactive Student Handling)
Hệ thống PHẢI KHÔNG xóa sinh viên đã import trước đó nếu họ vắng mặt trong file CSV mới. Thay vào đó, hệ thống có thể chuyển trạng thái sang `INACTIVE`.

#### Scenario: Sinh viên cũ không xuất hiện trong CSV mới
- GIVEN một sinh viên đã được import từ batch trước (`students.status = ACTIVE`)
- WHEN batch mới được parse và sinh viên không có trong danh sách
- THEN hệ thống KHÔNG thực hiện `DELETE`
- AND cập nhật `students.status = INACTIVE` trong transaction promotion (nếu cấu hình bật)
- AND giữ nguyên lịch sử `source_batch_id` để audit

---

### Requirement: Import không chặn hệ thống (Non-Blocking Import)
Hệ thống PHẢI cho phép sinh viên đăng ký workshop và các nghiệp vụ khác hoạt động bình thường trong lúc worker đang import CSV.

#### Scenario: Sinh viên đăng ký trong lúc import chạy
- GIVEN một batch CSV đang ở trạng thái `PARSING` hoặc `PROMOTING`
- WHEN sinh viên gửi request `POST /registrations`
- THEN backend xử lý registration bình thường
- AND không bị block bởi lock của bảng `students` hoặc transaction import
- AND validation đăng ký sử dụng snapshot dữ liệu nhất quán tại thời điểm request

---

## Technical Constraints

| Tham số | Giá trị | Lý do / Ghi chú triển khai |
| --- | --- | --- |
| **Import target** | `student_import_rows` (staging table) | Cách ly hoàn toàn với bảng `students` production |
| **Promotion method** | Atomic DB Transaction (`Prisma.$transaction`) | Đảm bảo all-or-nothing, rollback tự động khi crash |
| **Error threshold** | ≤ 20% (`error_rows / total_rows`) | Ngưỡng an toàn; vượt quá → `REJECTED` |
| **Duplicate detection** | SHA-256 checksum (`student_import_batches.checksum UNIQUE`) | Chống import trùng file ban đêm |
| **Required columns** | `student_code, email, full_name, faculty` | Bắt buộc có để upsert; thiếu 1 cột → batch reject |
| **Validation rules** | Email: RFC 5322 regex; `student_code`: alphanumeric; nội bộ batch: không trùng code | Giảm thiểu dữ liệu rác trước khi promotion |
| **Student code constraint** | `UNIQUE` trên `students.student_code` | Ngăn trùng lặp production, hỗ trợ upsert an toàn |
| **Concurrency** | Import không block API | Chạy trên BullMQ worker độc lập, không dùng table lock dài |
| **Audit fields** | `source_batch_id`, `last_seen_in_import_at` | Truy vết nguồn dữ liệu và lần import gần nhất |
| **Inactive policy** | Không delete, chuyển `INACTIVE` | Bảo toàn lịch sử đăng ký/check-in cũ, tuân thủ GDPR/data retention |

---

## Integration Points & Dependencies

### Phụ thuộc hệ thống
| Thành phần | Vai trò | Giao thức / Interface |
| --- | --- | --- |
| **Supabase Storage** | Lưu file CSV export từ hệ thống cũ | HTTP REST SDK, bucket `student-imports` |
| **BullMQ + Redis** | Queue xử lý batch tuần tự | Queue name `student-import`, 1 worker concurrency |
| **PostgreSQL** | Lưu staging table, production `students`, batch audit | Prisma ORM, atomic transaction, unique constraints |
| **Cron Scheduler** | Phát hiện file mới định kỳ | `@nestjs/schedule`, chạy 2:00 AM |
| **Admin Web App** | Hiển thị báo cáo import, lỗi từng dòng | `GET /admin/imports/students`, `GET /admin/imports/students/:id` |

### Endpoint API liên quan
| Method | Endpoint | Role yêu cầu | Mô tả |
| --- | --- | --- | --- |
| `GET` | `/admin/imports/students` | `ORGANIZER`, `ADMIN` | Danh sách phân trang các batch import (checksum, status, row counts) |
| `GET` | `/admin/imports/students/:batchId` | `ORGANIZER`, `ADMIN` | Chi tiết batch + danh sách dòng lỗi (`student_import_rows`) |
| `POST` | `/admin/imports/students/retry` | `ADMIN` | (Optional) Chạy lại batch `REJECTED` hoặc `FAILED` thủ công |

### Event & State Mapping
| Trạng thái Batch | Ý nghĩa | Hành động tiếp theo |
| --- | --- | --- |
| `PARSING` | Worker đang đọc CSV & ghi staging | Tiếp tục parse → validate → check threshold |
| `PROMOTING` | Đang chạy transaction upsert | Commit → `PROMOTED` / Rollback → `FAILED` |
| `PROMOTED` | Thành công, dữ liệu đã vào `students` | Admin xem báo cáo, kết thúc pipeline |
| `REJECTED` | Lỗi > 20% hoặc thiếu header | `students` không đổi, admin xem log lỗi |
| `DUPLICATE` | Checksum trùng batch cũ | Bỏ qua, không tạo job mới |

---

## Error Handling Matrix

| Lỗi | Hành động hệ thống | Phản hồi cho admin | Hành động khắc phục |
| --- | --- | --- | --- |
| **File CSV thiếu cột bắt buộc** | Batch → `REJECTED`, không parse dòng | Admin thấy "Missing required columns: email, faculty" | Liên hệ hệ thống cũ xuất đúng schema, upload file mới |
| **Tỷ lệ lỗi > 20%** | Validation dừng, batch → `REJECTED` | Admin thấy "Error rate 24.5% exceeds 20% threshold" | Export lại CSV từ hệ thống cũ, kiểm tra dữ liệu nguồn |
| **Worker crash khi promotion** | DB transaction rollback tự động | Batch giữ trạng thái trung gian, không ảnh hưởng `students` | Scheduler/worker restart, retry an toàn từ file gốc |
| **Checksum trùng file cũ** | Scheduler bỏ qua, đánh dấu `DUPLICATE` | Không hiển thị batch mới trong danh sách | Hệ thống tự động bỏ qua, tiết kiệm tài nguyên |
| `student_code` trùng trong batch | Dòng đánh dấu `DUPLICATE`, không upsert | Admin thấy "Duplicate student_code within batch" | Hệ thống cũ cần chuẩn hóa dữ liệu trước khi export |
| **Import chạy song song với đăng ký** | Worker dùng snapshot/isolation, không lock table | Sinh viên đăng ký bình thường, không timeout | Thiết kế transaction ngắn, dùng `READ COMMITTED` hoặc `REPEATABLE READ` |

---

## Tiêu chí chấp nhận (Acceptance Criteria)

- [ ] Scheduler phát hiện file CSV mới theo checksum, tạo batch `PARSING` và đẩy job vào queue.
- [ ] File trùng checksum đã import bị bỏ qua tự động, không tạo batch rác.
- [ ] CSV thiếu header bắt buộc hoặc lỗi cú pháp → batch `REJECTED`, bảng `students` không đổi.
- [ ] Dòng CSV lỗi được ghi rõ `error_message` trong `student_import_rows`, admin có thể xem & export.
- [ ] Tỷ lệ lỗi ≤ 20% → worker mở transaction upsert vào `students`, commit thành công → batch `PROMOTED`.
- [ ] Worker crash giữa chừng khi promotion → transaction rollback, dữ liệu production an toàn.
- [ ] Sinh viên không còn trong CSV mới → chuyển `INACTIVE`, KHÔNG bị `DELETE`.
- [ ] Sinh viên có thể đăng ký workshop bình thường trong lúc worker đang import CSV.
- [ ] Admin truy cập `GET /admin/imports/students/:batchId` xem được báo cáo: tổng dòng, dòng thành công, dòng lỗi, checksum, thời gian chạy.
- [ ] `student_import_batches.checksum` là UNIQUE constraint, ngăn insert trùng lặp ở cấp DB.
- [ ] `students.student_code` là UNIQUE, hỗ trợ `ON CONFLICT DO UPDATE` an toàn khi promotion.

---

## Ghi chú triển khai (Implementation Notes)

### Luồng Cron & Parser (Pseudo-code)
```typescript
@Cron('0 2 * * *') // 2:00 AM hàng ngày
async function checkAndQueueNewCSV() {
  const files = await storage.list('student-imports');
  for (const file of files) {
    const checksum = computeSHA256(file.path);
    const exists = await prisma.student_import_batches.findUnique({ where: { checksum } });
    if (exists) continue;

    await prisma.student_import_batches.create({
      data: {
        file_path: file.path,
        checksum,
        status: 'PARSING',
        error_threshold_pct: 20,
      },
    });
    await bullQueue.add('process-csv-batch', { batchId: checksum, filePath: file.path });
  }
}
```

### Validation & Atomic Promotion (Pseudo-code)
```typescript
async function promoteValidRows(batchId: string) {
  const batch = await prisma.student_import_batches.findUnique({ where: { id: batchId } });
  const validRows = await prisma.student_import_rows.findMany({
    where: { batch_id: batchId, row_status: 'VALID' },
  });

  // Check threshold trước khi mở transaction
  if (batch.error_rows / batch.total_rows > batch.error_threshold_pct / 100) {
    await prisma.student_import_batches.update({ where: { id: batchId }, data: { status: 'REJECTED' } });
    return;
  }

  // Atomic promotion
  await prisma.$transaction(async (tx) => {
    for (const row of validRows) {
      await tx.students.upsert({
        where: { student_code: row.student_code },
        create: { student_code: row.student_code, email: row.email, full_name: row.full_name, faculty: row.faculty, status: 'ACTIVE', source_batch_id: batchId, last_seen_in_import_at: new Date() },
        update: { email: row.email, full_name: row.full_name, faculty: row.faculty, status: 'ACTIVE', source_batch_id: batchId, last_seen_in_import_at: new Date() },
      });
    }
    // Optional: mark absent students as INACTIVE
    await tx.students.updateMany({
      where: { status: 'ACTIVE', last_seen_in_import_at: { lt: new Date(batch.created_at) } },
      data: { status: 'INACTIVE' },
    });
    await tx.student_import_batches.update({ where: { id: batchId }, data: { status: 'PROMOTED', completed_at: new Date() } });
  });
}
```

### Schema Ràng buộc (Prisma)
```prisma
model student_import_batches {
  id                String   @id @default(uuid())
  file_path         String
  checksum          String   @unique
  status            String   // PARSING, PROMOTING, PROMOTED, REJECTED, FAILED
  total_rows        Int
  valid_rows        Int
  error_rows        Int
  error_threshold_pct Int @default(20)
  started_at        DateTime @default(now())
  completed_at      DateTime?
}

model student_import_rows {
  id                String   @id @default(uuid())
  batch_id          String
  row_number        Int
  student_code      String
  email             String
  full_name         String
  faculty           String
  row_status        String   // VALID, ERROR, DUPLICATE
  error_message     String?
  
  @@unique([batch_id, row_number])
}
```

### Lưu ý Performance & Isolation
- **Không dùng `LOCK TABLE students`**: Transaction promotion chỉ dùng row-level upsert, cho phép read/write đồng thời từ API registration.
- **Batch processing**: Parse CSV theo chunk (ví dụ 1000 dòng/chunk) để tránh OOM worker khi file > 50MB.
- **Idempotent retry**: Nếu batch `PROMOTING` bị crash, worker restart có thể kiểm tra `status` và chạy lại từ đầu hoặc tiếp tục chunk tiếp theo.

