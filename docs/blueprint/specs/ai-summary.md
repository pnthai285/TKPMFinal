# Delta for AI Summary

## Mô tả tính năng
Ban tổ chức (Organizer/Admin) có thể tải lên file PDF giới thiệu workshop. Hệ thống xử lý bất đồng bộ thông qua background worker theo mô hình pipe-and-filter: trích xuất văn bản → làm sạch → chia chunk → gọi AI model → validate output → lưu summary. Thiết kế đảm bảo luồng xử lý AI hoàn toàn tách biệt với nghiệp vụ cốt lõi: không block request upload, không ảnh hưởng khả năng xem/đăng ký workshop khi AI gặp sự cố.

---

## ADDED Requirements

### Requirement: Xử lý PDF bất đồng bộ (Async PDF Processing)
Hệ thống PHẢI xử lý file PDF được upload thông qua background worker mà không block request upload.

#### Scenario: Organizer upload PDF hợp lệ
- GIVEN một ORGANIZER hoặc ADMIN đã xác thực
- WHEN organizer upload file PDF ≤ 20 MB với MIME type `application/pdf` qua endpoint `POST /admin/workshops/:id/documents`
- THEN backend validate MIME type và kích thước file
- AND lưu file vào Supabase Storage bucket `workshop-docs`
- AND tạo record `workshop_document` với `upload_status = UPLOADED` và `summary_status = UPLOADED`
- AND publish job `AI_SUMMARY_REQUESTED` vào BullMQ queue `ai-summary` với payload chứa `workshop_id` và `file_path`
- AND trả response `201 Created` trong vòng ≤ 2 giây, không chờ AI xử lý xong

#### Scenario: PDF vượt giới hạn kích thước
- GIVEN một ORGANIZER hoặc ADMIN đã xác thực
- WHEN organizer upload file PDF > 20 MB
- THEN backend reject request ngay tại middleware validation
- AND trả về `413 Payload Too Large` với message "File vượt quá giới hạn 20 MB"

#### Scenario: File không phải định dạng PDF
- GIVEN một ORGANIZER hoặc ADMIN đã xác thực
- WHEN organizer upload file có MIME type khác `application/pdf`
- THEN backend reject request ngay tại middleware validation
- AND trả về `400 Bad Request` với error detail "Chỉ chấp nhận file định dạng PDF"

#### Scenario: Upload PDF mới khi job cũ đang chạy
- GIVEN một workshop có job `AI_SUMMARY_REQUESTED` đang ở trạng thái `active` hoặc `waiting` trong queue `ai-summary`
- WHEN admin upload file PDF mới thay thế cho file cũ
- THEN backend cancel job cũ trong BullMQ (sử dụng `queue.removeJob(jobId)`)
- AND tạo record `workshop_document` mới với `upload_status = UPLOADED`
- AND publish job `AI_SUMMARY_REQUESTED` mới cho file PDF mới
- AND giữ nguyên `ai_summary` cũ cho đến khi job mới thành công

---

### Requirement: Trích xuất và tóm tắt văn bản bằng AI
Hệ thống PHẢI trích xuất, làm sạch và chia nhỏ văn bản PDF trước khi gọi AI model, sau đó validate output trước khi lưu vào database.

#### Scenario: Worker xử lý PDF đọc được thành công
- GIVEN một job `AI_SUMMARY_REQUESTED` tồn tại trong queue `ai-summary`
- WHEN BullMQ worker pick up job và bắt đầu xử lý
- THEN worker tải PDF từ Supabase Storage và trích xuất raw text bằng thư viện `pdf-parse`
- AND làm sạch văn bản: loại bỏ header/footer lặp, chuẩn hóa whitespace, strip ký tự điều khiển
- AND chia văn bản sạch thành các chunk ≤ 3.000 tokens mỗi chunk (sử dụng `tiktoken` hoặc ước lượng ký tự)
- AND gọi AI provider (Gemini/OpenAI) tuần tự cho từng chunk, ghép kết quả trả về theo thứ tự
- AND validate output ghép: độ dài ≥ 100 ký tự, không lỗi encoding, không bị truncated
- AND lưu `ai_summary` vào trường `workshops.ai_summary`
- AND cập nhật `workshops.summary_status = AI_GENERATED`
- AND Student Web App tự động hiển thị summary trên trang chi tiết workshop qua endpoint `GET /workshops/:id`

#### Scenario: Gọi AI bị timeout
- GIVEN worker đang gọi AI provider cho một chunk văn bản
- WHEN response từ AI provider vượt quá 30 giây
- THEN worker coi đây là lỗi và ghi log `AI_TIMEOUT`
- AND retry theo exponential backoff: 1 phút → 5 phút → 15 phút (tối đa 3 attempts)
- AND sau 3 lần thất bại liên tiếp, cập nhật `summary_status = SUMMARY_FAILED` và lưu `error_reason = "AI_PROVIDER_TIMEOUT"`

#### Scenario: AI trả về output không hợp lệ
- GIVEN worker nhận được response từ AI provider
- WHEN output rỗng, < 100 ký tự, hoặc bị truncated/cắt cụt
- THEN worker đánh dấu job là failed
- AND ghi `error_reason` cụ thể (ví dụ: `OUTPUT_TOO_SHORT`, `OUTPUT_TRUNCATED`)
- AND KHÔNG ghi đè hoặc publish summary vào database

#### Scenario: PDF bị hỏng hoặc không đọc được
- GIVEN worker pick up job cho file PDF bị corrupted, encrypted, hoặc scan-only (không có text layer)
- WHEN quá trình trích xuất text thất bại hoặc trả về chuỗi rỗng
- THEN worker cập nhật `summary_status = SUMMARY_FAILED`
- AND lưu `error_reason` chi tiết (ví dụ: `EXTRACTION_FAILED`, `ENCRYPTED_PDF`, `SCAN_ONLY_PDF`)

#### Scenario: AI provider down hoặc không khả dụng
- GIVEN AI provider trả về lỗi kết nối hoặc HTTP 5xx liên tiếp
- WHEN worker không thể gọi thành công sau 3 retry attempts
- THEN worker cập nhật `summary_status = SUMMARY_FAILED` với `error_reason = "AI_PROVIDER_UNAVAILABLE"`
- AND workshop vẫn hiển thị bình thường với fallback message "Summary đang được xử lý" hoặc "Không khả dụng"

---

### Requirement: Theo dõi trạng thái xử lý summary (Summary Status Tracking)
Hệ thống PHẢI duy trì trường `summary_status` trên bảng `workshops` để phản ánh chính xác vòng đời xử lý AI.

#### Scenario: Vòng đời trạng thái summary_status
- GIVEN một file PDF được upload thành công
- THEN `summary_status` chuyển đổi theo luồng:
  ```
  UPLOADED 
    → PROCESSING (khi worker bắt đầu xử lý, optional) 
    → AI_GENERATED (khi thành công) 
    HOẶC SUMMARY_FAILED (khi thất bại sau 3 retry)
    → ADMIN_EDITED (nếu admin chỉnh sửa thủ công sau khi AI_GENERATED)
  ```

#### Scenario: Admin xem trạng thái xử lý
- GIVEN một ADMIN hoặc ORGANIZER mở trang chi tiết workshop trong Admin Web App
- WHEN job AI đang ở trạng thái `PENDING`, `PROCESSING`, hoặc `FAILED`
- THEN UI hiển thị rõ `summary_status` hiện tại
- AND hiển thị `error_reason` chi tiết khi `summary_status = SUMMARY_FAILED` để admin có thể debug

#### Scenario: Student xem workshop khi summary chưa sẵn sàng
- GIVEN `summary_status` là `UPLOADED`, `PROCESSING`, hoặc `SUMMARY_FAILED`
- WHEN sinh viên gọi `GET /workshops/:id`
- THEN backend trả về workshop data bình thường
- AND trường `ai_summary` trả về `null` hoặc fallback string `"Summary đang được xử lý"`
- AND không ảnh hưởng đến khả năng đăng ký workshop

---

### Requirement: Admin chỉnh sửa summary thủ công
Hệ thống PHẢI cho phép ADMIN và ORGANIZER chỉnh sửa thủ công summary đã được AI tạo ra.

#### Scenario: Admin chỉnh sửa summary đã được AI generate
- GIVEN một workshop có `summary_status = AI_GENERATED` hoặc `ADMIN_EDITED`
- WHEN admin submit nội dung summary đã chỉnh sửa qua endpoint `PATCH /admin/workshops/:id/summary`
- THEN backend cập nhật `ai_summary` trong database
- AND chuyển `summary_status = ADMIN_EDITED`
- AND ghi audit log: `editor_user_id`, `edited_at`, `previous_summary_hash`

#### Scenario: Admin chỉ được chỉnh sửa khi summary đã được tạo
- GIVEN một workshop có `summary_status = UPLOADED` hoặc `SUMMARY_FAILED`
- WHEN admin cố gắng gọi `PATCH /admin/workshops/:id/summary`
- THEN backend trả về `400 Bad Request` với message "Chỉ có thể chỉnh sửa summary đã được AI tạo hoặc đã chỉnh sửa trước đó"

---

### Requirement: Cách ly xử lý AI (AI Processing Isolation)
Hệ thống PHẢI đảm bảo lỗi xử lý AI không ảnh hưởng đến khả năng hiển thị workshop hoặc đăng ký của sinh viên.

#### Scenario: AI thất bại nhưng workshop vẫn khả dụng
- GIVEN `summary_status = SUMMARY_FAILED` trên một workshop
- WHEN sinh viên xem trang chi tiết workshop qua `GET /workshops/:id`
- THEN workshop data được trả về bình thường với đầy đủ thông tin: title, speaker, room, capacity, schedule
- AND trường `ai_summary` trả về `null` hoặc fallback message
- AND endpoint đăng ký `POST /registrations` hoạt động bình thường, không phụ thuộc vào `summary_status`

#### Scenario: AI provider lỗi kéo dài không ảnh hưởng hệ thống
- GIVEN AI provider down hoàn toàn trong 1 giờ
- WHEN nhiều organizer upload PDF mới
- THEN tất cả job đều chuyển `SUMMARY_FAILED` sau 3 retry
- AND các chức năng khác (xem lịch, đăng ký, thanh toán, check-in) vẫn hoạt động 100% bình thường
- AND admin nhận được cảnh báo qua audit log để có hành động khắc phục

---

## Technical Constraints

| Tham số | Giá trị | Lý do / Ghi chú triển khai |
| --- | --- | --- |
| **Kích thước PDF tối đa** | 20 MB | Giới hạn băng thông upload, tránh OOM worker, thời gian parse hợp lý |
| **Định dạng chấp nhận** | `application/pdf` | Chỉ hỗ trợ PDF; Word/image xử lý riêng nếu cần mở rộng |
| **Queue system** | BullMQ queue `ai-summary` | Async processing; endpoint upload KHÔNG được block chờ AI |
| **Storage bucket** | `workshop-docs` (Supabase Storage) | Lưu trữ tập trung, versioned, dễ backup/restore |
| **Chunk size** | ≤ 3.000 tokens | Phù hợp context window phổ biến của LLM; tránh truncation & kiểm soát chi phí |
| **Timeout per AI call** | 30 giây | Hard limit; vượt quá → tính là failure → kích hoạt retry |
| **Max retry attempts** | 3 lần | Backoff: 1m → 5m → 15m; sau exhaustion → `SUMMARY_FAILED` |
| **Min output length** | 100 ký tự | Lọc output rỗng/malformed từ LLM; đảm bảo chất lượng tối thiểu |
| **Status field** | `workshops.summary_status` | Enum: `UPLOADED`, `PROCESSING`, `AI_GENERATED`, `ADMIN_EDITED`, `SUMMARY_FAILED` |
| **Error reason field** | `workshop_documents.error_reason` | Lưu lý do lỗi chi tiết để admin debug và audit |
| **Isolation rule** | AI failure ≠ Workshop failure | Core registration/viewing KHÔNG bao giờ phụ thuộc vào AI provider availability |
| **Job payload** | `{ workshop_id: uuid, file_path: string, uploaded_by: uuid }` | Đủ thông tin để worker xử lý độc lập, không cần query thêm |
| **Audit logging** | Ghi log: upload, job start, job success/failure, admin edit | Truy vết toàn bộ vòng đời xử lý cho mục đích debug và compliance |

---

## Integration Points & Dependencies

### Phụ thuộc hệ thống
| Thành phần | Vai trò | Giao thức / Interface |
| --- | --- | --- |
| **Supabase Storage** | Lưu file PDF upload | HTTP REST API (`@supabase/storage-js`) |
| **BullMQ + Redis** | Queue job xử lý bất đồng bộ | `ioredis` connection, queue name `ai-summary` |
| **AI Provider (Gemini/OpenAI)** | Tạo summary từ text chunks | HTTPS REST API với API key, timeout 30s |
| **PostgreSQL** | Lưu metadata, status, summary result | Prisma ORM, transaction-safe updates |

### Endpoint API liên quan
| Method | Endpoint | Role yêu cầu | Mô tả |
| --- | --- | --- | --- |
| `POST` | `/admin/workshops/:id/documents` | ORGANIZER, ADMIN | Upload PDF, trigger AI job |
| `GET` | `/admin/workshops/:id/summary-status` | ORGANIZER, ADMIN | Xem trạng thái xử lý + error reason |
| `PATCH` | `/admin/workshops/:id/summary` | ORGANIZER, ADMIN | Chỉnh sửa summary thủ công |
| `GET` | `/workshops/:id` | Public | Trả về workshop detail + `ai_summary` (nếu có) |

### Event Domain phát ra
| Event | Trigger bởi | Payload chính | Consumer |
| --- | --- | --- | --- |
| `WorkshopDocumentUploaded` | Upload thành công | `{ workshop_id, file_path, uploaded_by }` | Notification (optional), Audit |
| `AiSummaryGenerated` | Worker xử lý thành công | `{ workshop_id, summary_length, generated_at }` | Notification (optional), Cache invalidation |
| `AiSummaryFailed` | Worker thất bại sau retry | `{ workshop_id, error_reason, failed_at }` | Alerting, Audit |

---

## Error Handling Matrix

| Lỗi | HTTP Status / Worker Action | Phản hồi cho user | Hành động hệ thống |
| --- | --- | --- | --- |
| File > 20 MB | `413 Payload Too Large` | "File vượt quá 20 MB" | Không tạo job, không lưu vào storage |
| MIME type sai | `400 Bad Request` | "Chỉ chấp nhận file PDF" | Reject ngay tại validation middleware |
| PDF corrupted | Worker: `SUMMARY_FAILED` + `error_reason` | Admin thấy status lỗi trong UI | Ghi log, không ảnh hưởng workshop |
| AI timeout | Worker: retry 3 lần → `SUMMARY_FAILED` | Admin thấy "Timeout sau 3 lần thử" | Backoff retry, sau đó fail gracefully |
| Output AI < 100 ký tự | Worker: `SUMMARY_FAILED` + `error_reason` | Admin thấy "Output không đủ độ dài" | Không publish summary, giữ nguyên trạng thái cũ |
| AI provider down | Worker: `SUMMARY_FAILED` sau 3 retry | Admin thấy "AI provider không khả dụng" | Circuit breaker ở level worker (optional), alert admin |
| Network error khi lưu DB | Worker: retry job theo BullMQ policy | Không hiển thị cho user (xử lý backend) | BullMQ retry với backoff, sau đó fail permanent |
| Admin edit khi status chưa sẵn sàng | `400 Bad Request` | "Chưa thể chỉnh sửa summary" | Validate `summary_status` trước khi update |

---

## Tiêu chí chấp nhận (Acceptance Criteria)

- [ ] Upload PDF ≤ 20 MB trả về `201 Created` trong vòng ≤ 2 giây, không block chờ AI xử lý.
- [ ] PDF > 20 MB bị reject ngay với `413 Payload Too Large`, không tạo job trong queue.
- [ ] File không phải `application/pdf` bị reject với `400 Bad Request` kèm message rõ ràng.
- [ ] Khi worker xử lý thành công, `ai_summary` tự động xuất hiện trên `GET /workshops/:id` và `summary_status` chuyển sang `AI_GENERATED`.
- [ ] AI timeout 3 lần liên tiếp → hệ thống cập nhật `summary_status = SUMMARY_FAILED` kèm `error_reason`, không retry vô hạn.
- [ ] Output AI < 100 ký tự hoặc bị truncated → worker đánh dấu failed, không publish summary.
- [ ] Lỗi xử lý AI (timeout, provider down, output invalid) KHÔNG ảnh hưởng đến khả năng xem chi tiết hoặc đăng ký workshop.
- [ ] Admin/Organizer chỉnh sửa summary thành công → `summary_status` chuyển sang `ADMIN_EDITED` và ghi đè lên `AI_GENERATED`.
- [ ] Upload PDF mới khi job cũ đang chạy → job cũ bị cancel, chỉ job mới nhất được thực thi và publish kết quả.
- [ ] Student xem workshop khi `summary_status = SUMMARY_FAILED` → vẫn thấy đầy đủ thông tin workshop, chỉ thiếu summary.
- [ ] Audit log ghi nhận đầy đủ: thời điểm upload, bắt đầu xử lý, thành công/thất bại, admin edit (nếu có).
- [ ] Worker xử lý độc lập: không phụ thuộc vào session user, chỉ dùng payload job để xử lý.

---

## Ghi chú triển khai (Implementation Notes)

### Cấu trúc job payload cho BullMQ
```typescript
interface AiSummaryJobPayload {
  workshopId: string;           // UUID
  filePath: string;             // Path trong Supabase Storage
  uploadedBy: string;           // UUID của user upload
  mimeType: 'application/pdf';  // Đã validate ở API layer
  fileSizeBytes: number;        // Đã validate ≤ 20MB
}
```

### Luồng xử lý worker (pseudo-code)
```typescript
async function processAiSummaryJob(payload: AiSummaryJobPayload) {
  // 1. Tải PDF từ Storage
  const pdfBuffer = await storage.download(payload.filePath);
  
  // 2. Extract text
  const rawText = await pdfParse.extract(pdfBuffer); // pdf-parse library
  
  // 3. Clean text
  const cleanText = textCleaner.removeHeadersFooters(rawText)
                               .normalizeWhitespace()
                               .stripControlChars();
  
  // 4. Chunking
  const chunks = textChunker.split(cleanText, { maxTokens: 3000 });
  
  // 5. Call AI với retry + timeout
  let summary = '';
  for (const chunk of chunks) {
    const response = await aiClient.summarize(chunk, { timeout: 30_000 });
    summary += response.text;
  }
  
  // 6. Validate output
  if (summary.length < 100 || !isValidEncoding(summary)) {
    throw new ValidationError('OUTPUT_INVALID');
  }
  
  // 7. Save to DB (transaction-safe)
  await prisma.workshop.update({
    where: { id: payload.workshopId },
    data: {
      ai_summary: summary,
      summary_status: 'AI_GENERATED',
    },
  });
  
  // 8. Emit domain event (optional)
  eventBus.publish('AiSummaryGenerated', { ... });
}
```

### Xử lý cancel job khi upload mới
```typescript
// Trong endpoint POST /admin/workshops/:id/documents
const existingJob = await queue.getJobs(['active', 'waiting'], 0, 1, true)
  .then(jobs => jobs.find(j => j.data.workshopId === workshopId));

if (existingJob) {
  await existingJob.discard(); // hoặc existingJob.remove() tùy BullMQ version
}

// Sau đó tạo job mới
await queue.add('AI_SUMMARY_REQUESTED', newPayload, {
  attempts: 3,
  backoff: { type: 'exponential', delay: 60_000 },
});
```

### Fallback UI cho Student Web App
```tsx
// Trong component WorkshopDetail
{workshop.summary_status === 'AI_GENERATED' && workshop.ai_summary && (
  <SummarySection content={workshop.ai_summary} />
)}
{workshop.summary_status === 'SUMMARY_FAILED' && (
  <SummaryFallback message="Summary tạm thời không khả dụng. Vui lòng thử lại sau." />
)}
{['UPLOADED', 'PROCESSING'].includes(workshop.summary_status) && (
  <SummaryFallback message="Summary đang được xử lý..." loading />
)}
```
