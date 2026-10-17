# TDD — Quản lý công việc nội bộ CMS

- Ngày: 17/10/2026. Trạng thái: đề xuất kỹ thuật, chưa có migration/code triển khai.
- TDD ở đây là Technical Design Document; kế hoạch kiểm thử ở mục 9.
- Đặc tả: [DAC_TA_QUAN_LY_CONG_VIEC_KANBAN.md](DAC_TA_QUAN_LY_CONG_VIEC_KANBAN.md).

## 1. Cơ sở từ mã nguồn hiện tại

| Thành phần | Bằng chứng trong repo | Hướng dùng |
| --- | --- | --- |
| Tài khoản CMS | `vl-shared/.../jdbc/entity/User.java`: users, id Long, employee_id Integer, status | FK tới users; join employees chỉ khi cần tên/mã nhân viên |
| Audit dùng chung | `vl-api/.../service/AuditService.java`, `vl-shared/.../jdbc/dao/AuditLogDao.java` | Ghi module tasks, entity_type task; không thêm task_audit_logs |
| Phân quyền | PermissionDao, roles/permissions và các controller hiện tại | Thêm resource tasks và 5 action tương ứng đặc tả |
| Task tư vấn kinh doanh | Migration `V20260924_001__growth_advisor_tasks.sql` | Giữ riêng: bảng này có opportunity/revenue, không phải yêu cầu CMS |
| Upload theo module | AdministrativeBoundaryFileService và các controller upload nghiệp vụ | Có tiền lệ multipart; chưa xác nhận kho tệp riêng tư dùng chung đáp ứng task, cần adapter bảo vệ quyền |
| Frontend | React/TypeScript, TanStack Router/Query, component UI chung | Feature tasks độc lập, dùng lại modal/filter/calendar/pagination |
| Production đang dùng | MariaDB 5.5.68 theo đợt triển khai hiện tại | Thiết kế SQL tương thích, không yêu cầu nâng DB |

Các tên bảng/API bên dưới là thiết kế mới. Trước khi tạo migration cần kiểm tra lại schema thực tế, kiểu khóa users và convention quyền trên nhánh main mới nhất; không suy ra đã có module chỉ vì có bảng growth_advisor_tasks.

## 2. Kiến trúc và phạm vi mã

- `vl-cms/src/features/tasks`: board, list, detail modal, editor, comments, attachments, API hooks/schema.
- Route `_authenticated/tasks`; query được validate bằng schema, không lưu riêng bản sao filter trong local state.
- `vl-api/.../controller/tasks/TaskController`: auth, DTO, HTTP và audit context.
- `vl-shared/.../service/tasks/TaskService`: validation, quyền theo đối tượng, chuyển trạng thái, transaction.
- `vl-shared/.../jdbc/entity/tasks` và `jdbc/dao/tasks`: truy vấn có tham số, phân trang, khóa phiên bản.
- Tệp qua adapter lưu trữ ở API, nội dung ngoài web root; metadata và phân quyền trong task service.
- Migration ở `vl-api/src/main/resources/db/migration`; không sửa migration đã áp dụng.

Không sửa vl-app/vl-desktop. Không trộn với employee_work_sheets (công kho) hoặc growth_advisor_tasks (tư vấn doanh thu).

## 3. Mô hình dữ liệu: chỉ 3 bảng mới

### 3.1. tasks — yêu cầu/công việc độc lập

| Cột | Kiểu đề xuất | Ý nghĩa |
| --- | --- | --- |
| id | BIGINT PK AUTO_INCREMENT | Định danh; mã hiển thị `TASK-<id>` tính từ ID, không cần bảng sinh mã |
| type | VARCHAR(20) | FEATURE / BUG / IMPROVEMENT / TASK |
| title | VARCHAR(200) | Tiêu đề |
| description | TEXT | Mô tả |
| module_code | VARCHAR(80) NULL | Mã từ danh mục module cho phép, null = Khác |
| status | VARCHAR(20) | INBOX / TODO / IN_PROGRESS / REVIEW / DONE / CANCELLED |
| priority | VARCHAR(10) | URGENT / HIGH / NORMAL / LOW |
| reporter_user_id | BIGINT NOT NULL FK users | Người gửi, lấy từ authentication |
| assignee_user_id | BIGINT NULL FK users | Một người phụ trách |
| due_date | DATE NULL | Hạn hoàn thành |
| source_path | VARCHAR(500) NULL | Đường dẫn CMS đã lọc, không URL có token |
| reproduction_steps, actual_result, expected_result | TEXT NULL từng cột | Dữ liệu riêng báo lỗi |
| acceptance_criteria | TEXT NULL | Tiêu chí nghiệm thu loại việc còn lại |
| resolution | TEXT NULL | Kết quả xử lý/hướng dẫn kiểm tra |
| delivery_environment | VARCHAR(80) NULL | Môi trường đã triển khai |
| delivered_at | DATETIME NULL | Thời điểm triển khai được xác nhận |
| blocked_reason | VARCHAR(1000) NULL | null = không chặn; có nội dung = bị chặn |
| duplicate_of_id | BIGINT NULL FK tasks | Task gốc khi hủy vì trùng |
| completed_at, completed_by | DATETIME NULL, BIGINT NULL FK users | Lần hoàn thành hiện tại |
| archived_at, archived_by | DATETIME NULL, BIGINT NULL FK users | Lưu trữ mềm |
| request_key | VARCHAR(64) ASCII NOT NULL | Khóa chống tạo trùng theo người gửi |
| request_hash | CHAR(64) ASCII NOT NULL | SHA-256 của payload tạo đã chuẩn hóa |
| version | BIGINT NOT NULL DEFAULT 0 | Optimistic locking |
| created_at, updated_at | DATETIME NOT NULL từng cột | Backend gán thời gian theo convention chung |
| updated_by | BIGINT NOT NULL FK users | Người cập nhật cuối |

Index: unique `(reporter_user_id, request_key)`; `(archived_at,status,id)`; `(assignee_user_id,status,due_date)`; `(reporter_user_id,created_at)`; `(module_code,status)`; `(due_date,status)`. Với V1 dùng tìm ID chính xác hoặc LIKE tiêu đề; chưa thêm FULLTEXT cho InnoDB MariaDB 5.5. Đo EXPLAIN trước khi thêm index lớn hoặc search engine.

Không thêm bảng trạng thái/ưu tiên/loại việc/module; các lựa chọn cố định V1 được backend trả qua options. Không thêm bảng board/member vì chỉ có một bảng chung. Không thêm sort_rank vì chưa hỗ trợ kéo thứ tự trong cột.

### 3.2. task_comments — trao đổi trên công việc

| Cột | Kiểu/ý nghĩa |
| --- | --- |
| id, task_id | BIGINT PK auto, BIGINT FK tasks |
| author_user_id | BIGINT FK users |
| body | TEXT, 1–10.000 ký tự |
| request_key, request_hash | VARCHAR(64) ASCII, CHAR(64) ASCII, chống tạo trùng |
| version | BIGINT DEFAULT 0 |
| created_at, updated_at | DATETIME, backend gán |
| deleted_at, deleted_by | DATETIME NULL, BIGINT NULL FK users; ẩn mềm |

Unique `(author_user_id,request_key)`, index `(task_id,created_at,id)`. Không cho đổi task_id/author sau tạo. Sau khi ẩn, API thường chỉ trả placeholder “Bình luận đã được gỡ”, không trả body cũ. Audit giữ lịch sử theo quyền hiện có; activity công khai trong task không lộ body đã gỡ.

### 3.3. task_attachments — tệp của công việc

| Cột | Kiểu/ý nghĩa |
| --- | --- |
| id, task_id | BIGINT PK auto, BIGINT FK tasks |
| uploaded_by | BIGINT FK users |
| original_name | VARCHAR(255), chỉ là nhãn, không dùng làm đường dẫn |
| storage_key | VARCHAR(190) ASCII UNIQUE, khóa ngẫu nhiên do server sinh |
| content_type, byte_size, checksum | VARCHAR(100), BIGINT, CHAR(64) ASCII |
| request_key | VARCHAR(64) ASCII, unique `(uploaded_by,request_key)` |
| created_at, deleted_at, deleted_by | DATETIME, DATETIME NULL, BIGINT NULL FK users |

Index `(task_id,deleted_at,id)`. Không lưu blob trong DB hoặc URL công khai vĩnh viễn. V1 tệp gắn task, không cần cột comment_id hay bảng liên kết phụ. Retry khóa upload cũ chỉ trả tệp cũ khi task và checksum khớp, ngược lại 409.

### 3.4. Quy ước tương thích DB

InnoDB, utf8mb4; khóa FK cùng kiểu/signedness với bảng đích. `SET NAMES utf8mb4` rõ ràng khi áp dụng SQL để tránh lỗi tiếng Việt đã gặp. Dùng DATETIME do service gán; không phụ thuộc JSON column/function, CHECK được DB thực thi, generated columns, nhiều TIMESTAMP auto-update hoặc index vượt 767 byte. Giới hạn enum, độ dài và state transition được service kiểm tra bắt buộc. FK dùng RESTRICT, tài khoản bị khóa thay vì xóa mất lịch sử.

## 4. Phân quyền và tính nhất quán

Dùng resource `tasks`, action `view/create/update/comment/manage` theo đặc tả. Options người phụ trách chỉ trả tài khoản active có view và update hoặc manage; không trả toàn bộ user entity/password. Người tạo lấy từ auth, không tin reporter_user_id từ client.

POST luôn đặt status=INBOX, reporter=current user. Người tạo được chọn ưu tiên ban đầu; assignee/due_date chỉ nhận khi có manage. PATCH của người tạo ở INBOX chỉ cho type/title/description/module/source_path và các trường mô tả theo loại; priority/due_date thuộc manage. Với người được giao, PATCH chỉ cho resolution/blocked_reason/delivery_environment/delivered_at. Từ chối field ngoài allowlist, không silently bỏ qua.

duplicate_of_id chỉ dùng khi chuyển CANCELLED vì trùng: phải tồn tại, khác chính nó, chưa lưu trữ và không tạo chu trình liên kết. Với V1 yêu cầu trỏ thẳng task gốc không có duplicate_of_id để tránh chuỗi gián tiếp. Khôi phục CANCELLED về INBOX xóa liên kết trùng hiện hành và lưu lý do trong audit.

- Mọi endpoint phải kiểm tra view trước, sau đó kiểm tra action và quan hệ với task.
- tasks.manage điều phối toàn bảng; tasks.update chỉ sửa kết quả/cờ chặn/chuyển xử lý cho task mình phụ trách.
- tasks.create cho sửa nội dung nghiệp vụ của chính mình ở INBOX và thực hiện nghiệm thu/trả lại/mở lại theo đặc tả.
- tasks.comment thêm bình luận/tệp khi task chưa lưu trữ; sửa/gỡ bình luận chính mình. Người tải lên được gỡ tệp mình tải; manage kiểm duyệt toàn bộ.
- DONE và CANCELLED khóa thông tin nghiệp vụ nhưng cho trao đổi/tệp đến khi lưu trữ. Mở lại/khôi phục trước khi sửa.
- Lịch sử task trả qua endpoint đã lọc module/entity/id và danh sách field an toàn; không yêu cầu cấp quyền xem toàn bộ audit_logs cho người dùng task.

Update dùng `WHERE id=:id AND version=:expected_version`, tăng version 1; không cập nhật được trả 409 TASK_VERSION_CONFLICT. State transition dùng cùng cơ chế; client không được PATCH status/assignee/archived ngoài endpoint chuyên biệt.

Tạo task/comment có request_key UUID do client giữ suốt lần gửi lại; unique key bảo vệ race. Cùng khóa + cùng hash trả lại kết quả cũ; cùng khóa khác hash trả 409 IDEMPOTENCY_CONFLICT. Unique violation phải xử lý ngoài transaction đã rollback rồi đọc bản ghi đã commit, không tiếp tục transaction lỗi.

Mutation task và ghi audit phải cùng transaction/JDBC connection. Cần xác minh propagation của AuditLogDao trong test rollback; không coi gọi AuditService sau commit là atomic. Nếu cần, gọi DAO trong service với actor/context truyền từ controller và đánh dấu request đã audit để MutationAuditFilter không ghi trùng. Không tạo bảng audit mới.

Audit action: CREATE, UPDATE, ASSIGN, TRANSITION, BLOCK, UNBLOCK, ARCHIVE, RESTORE, COMMENT_CREATE/EDIT/REMOVE, ATTACHMENT_ADD/REMOVE. Mọi activity gắn entity_type=task, entity_id=task.id; detail_ref trỏ comment/attachment. Ghi trạng thái trước/sau và lý do. Khi mở lại, xóa completed_at/by hiện hành nhưng lịch sử hoàn thành cũ còn trong audit.

## 5. Hợp đồng API đề xuất

Prefix `/tasks`; giữ envelope ApiResponse và chuẩn lỗi hiện có.

| Method / path | Ý nghĩa |
| --- | --- |
| GET /tasks/options | Loại/ưu tiên/trạng thái/module và capabilities người dùng |
| GET /tasks/assignees?keyword=&page=&size= | Người có thể nhận việc, phân trang, DTO tối thiểu |
| GET /tasks | Danh sách: page/size, keyword,type,module,status,assignee,reporter,priority,due_from,due_to,overdue,archived |
| GET /tasks/board | Cùng filter; 20 card đầu mỗi cột + total + next_cursor từng cột |
| GET /tasks/board/columns/{status}?cursor=&size= | Tải thêm một cột với cùng filter, size tối đa 50 |
| POST /tasks | Tạo: request_key, type,title,description và thuộc tính được phép |
| GET /tasks/{id} | Chi tiết + allowed_actions, version; không trả audit/tệp vô hạn |
| PATCH /tasks/{id} | Sửa field cho phép + expected_version |
| POST /tasks/{id}/assign | assignee_user_id nullable, expected_version; quyền manage |
| POST /tasks/{id}/transition | target_status, expected_version, reason, resolution/delivery nếu bước đó cần |
| POST /tasks/{id}/archive | expected_version; chỉ DONE/CANCELLED |
| POST /tasks/{id}/restore | expected_version; gỡ lưu trữ, giữ trạng thái nghiệp vụ |
| GET/POST /tasks/{id}/comments | Cursor đọc hoặc tạo có request_key |
| PATCH/DELETE /tasks/{id}/comments/{commentId} | Kiểm tra comment thuộc task; expected_version |
| GET/POST /tasks/{id}/attachments | Danh sách hoặc multipart, request_key |
| GET/DELETE /tasks/{id}/attachments/{attachmentId} | Download có auth hoặc ẩn metadata; kiểm tra FK |
| GET /tasks/{id}/activity | Lịch sử đã lọc an toàn, cursor |

GET /tasks size mặc định20, tối đa100. Board sort priority theo CASE rồi created_at/id tăng dần; cursor chứa tuple sort và filter fingerprint, validate server. Không OFFSET toàn bộ bảng để tải thêm card. Khi task đổi ưu tiên/trạng thái trong lúc phân trang, FE dedupe theo ID và refetch cột; count phản ánh thời điểm truy vấn, không cam kết snapshot xuyên nhiều HTTP request.

Query `assignee=me`/`reporter=me` được resolve server từ auth; giá trị unassigned chỉ hợp lệ với assignee. archived mặc định false, true = chỉ lưu trữ. Search keyword hỗ trợ TASK-123/id chính xác và tiêu đề; escape ký tự wildcard nếu tìm literal. Sort/list filter đều whitelist, không nối SQL trực tiếp.

Ví dụ transition:

```json
{
  "target_status": "REVIEW",
  "expected_version": 7,
  "resolution": "Đã sửa bộ lọc tháng; kiểm tra bằng đổi tháng rồi F5."
}
```

200/201 thành công; 400 dữ liệu không hợp lệ; 401 chưa đăng nhập; 403 thiếu quyền; 404 task/tệp không tồn tại; 409 sai phiên bản, khóa gửi lại khác payload hoặc trạng thái đã đổi; 413 quá dung lượng; 415 loại tệp không hỗ trợ. Transition sai điều kiện trả 400 với mã TASK_TRANSITION_INVALID và field_errors dễ hiển thị.

## 6. Frontend và thao tác Kanban

- State từ URL: view, keyword, type, module, assignee, reporter, priority, status, due_from/to, overdue, archived, page, size, task.
- Khi đổi filter reset page/cursor; debounce tìm kiếm 300ms; cache key phải chứa toàn bộ filter và phạm vi người đăng nhập.
- Modal mở dùng push history; đóng bằng Back nếu có entry mở modal, nếu truy cập deep link trực tiếp thì xóa task param bằng replace.
- Form create chia thông tin chính và mô tả theo loại. Dùng calendar, searchable select, modal chung CMS. Không tạo UI nhập mã task.
- Query detail/comments/activity/attachments phân trang riêng, chỉ fetch khi mở modal.
- Kéo card lấy snapshot cache, cập nhật tạm, gọi transition. Lỗi thì rollback; 409 hiện thông báo và refetch. Ngăn kéo task đang có mutation pending.
- Khi chuyển cần thêm lý do/kết quả/nghiệm thu, mở dialog trước khi gửi; hủy không thay dữ liệu.
- Thành công invalidate board, list, detail, activity và count liên quan. Không đánh dấu hoàn thành trước khi server xác nhận.
- DnD: package hiện tại chưa thấy thư viện chuyên dụng trong package.json; đề xuất bổ sung @dnd-kit/core ở bước triển khai sau khi kiểm tra tương thích lockfile. Không nâng React/router để làm tính năng này. Có nút chuyển trạng thái và thông báo screen reader, không chỉ HTML drag chuột.
- Màn hình nhỏ cho cuộn ngang cột hoặc dùng list; không ép năm cột nhỏ đến mức không đọc được.

## 7. Upload, bảo mật và lưu trữ

- Kiểm tra auth và quyền task trước đọc body lớn; giới hạn tổng10 tệp còn hiệu lực,10 MB mỗi tệp, kiểm tra cả MIME/signature thật.
- Chỉ PNG/JPEG/WebP/PDF; không nhận HTML/SVG/script. Tên tệp hiển thị escape, đường dẫn ngẫu nhiên chống traversal.
- Ghi tệp staging trước; trong transaction khóa hàng task để kiểm tra quota và lưu metadata; publish/đổi tên nguyên tử cùng filesystem trước commit. Lỗi rollback phải dọn tệp; có job dọn file mồ côi quá24h đối chiếu metadata, không xóa file đang được tham chiếu.
- Download qua endpoint có auth; `Content-Disposition: attachment` cho PDF, ảnh preview qua endpoint riêng được bảo vệ, `X-Content-Type-Options: nosniff`, cache private. Không để static web server expose thư mục upload.
- Ẩn tệp làm endpoint thường trả404; dữ liệu vật lý giữ theo chính sách backup trước khi có tác vụ dọn có chủ đích. Không hứa undo nếu đã xóa vật lý.
- Markdown render disable raw HTML, link chỉ http/https hoặc route nội bộ; không cho javascript URL. source_path loại token và query nhạy cảm bằng whitelist.
- Không bổ sung cơ chế fetch URL phía server từ nội dung task; không upload thẳng sang dịch vụ công khai.

## 8. Hiệu năng và vận hành

Board truy vấn count theo group status và giới hạn card từng cột, batch join users/count comments/attachments theo tập ID để tránh N+1. Không gửi description dài trên card. DONE cũng chỉ tải20 card đầu; lưu trữ chủ động giúp giảm nhiễu, không tự mất việc cũ.

V1 refetch khi focus và sau mutation; chưa cần WebSocket. Chỉ polling khi có nhu cầu thực tế. Ngày hiện tại/quá hạn tính server theo Asia/Ho_Chi_Minh.

Backup DB và thư mục tệp cùng quy trình. Log request ID và task ID, tránh ghi nguyên mô tả/tệp chứa dữ liệu nội bộ. Audit không bị xóa khi task lưu trữ.

## 9. Kế hoạch kiểm thử

| Nhóm | Ca cần chứng minh |
| --- | --- |
| State machine | Mỗi cạnh hợp lệ; chặn mọi cạnh khác; REVIEW cần resolution; DONE cần xác nhận triển khai; mở lại giữ audit |
| Quyền | Người tạo, người được giao, người khác, manage, không view; gọi API trực tiếp vẫn bị chặn; options không lộ user secret |
| Race | Hai update cùng version chỉ một thành công; retry create/comment cùng khóa chỉ một bản ghi; cùng khóa khác payload bị409 |
| Transaction | Lỗi ghi audit rollback mutation; không có audit thành công cho mutation rollback |
| Board | Count khớp filter; tải thêm không hiện ID trùng; đổi cột cập nhật hai count; lỗi mạng/409 hoàn nguyên card |
| Upload | Hai upload đồng thời không vượt quota; signature giả/oversize/traversal bị từ chối; tệp task khác/đã gỡ không tải được |
| XSS | Tiêu đề, Markdown, bình luận, tên tệp và link độc hại không thực thi |
| UX | F5/deep link/Back giữ đúng context; ESC form chưa lưu; kéo thả và bàn phím; mobile |
| DB | Migration trên MariaDB5.5 và DB dev; tiếng Việt roundtrip; FK/index phù hợp, không JSON SQL |
| Regression | Auth/audit hiện tại, menu Nhân sự đầu tiên, công kho và lương không thay đổi |

Mục tiêu tải thử: 10.000 task và 50.000 bình luận, trang board đầu phản hồi API p95 dưới1 giây trong môi trường staging được ghi rõ cấu hình; đây là mục tiêu nghiệm thu cần đo, không phải kết quả đã có.

## 10. Các bước triển khai sau khi duyệt tài liệu

1. Đồng bộ main, tạo nhánh riêng `feature/internal-task-kanban` cho các repo liên quan; chuyển hai tài liệu sang nhánh đó, không trộn implementation với nhánh lương.
2. Hoàn thiện migration3 bảng, permission, service/state machine/audit và test backend.
3. Làm danh sách, tạo/sửa/chi tiết và filter URL trước; sau đó board/DnD dùng cùng API nghiệp vụ.
4. Bổ sung bình luận, tệp bảo vệ quyền và activity; kiểm tra upload/rollback/concurrency.
5. QA với dữ liệu test ở staging, không thêm task giả vào production.
6. Backup, migration additive, deploy API rồi CMS; cấp quyền ADMIN trước, xác nhận smoke rồi mới mở quyền cho nhóm nội bộ.
7. Nếu rollback ứng dụng, giữ bảng/tệp mới để không mất dữ liệu đã nhập; không rollback bằng DROP TABLE.

Lần làm hiện tại chỉ viết đặc tả và TDD. Chưa tạo bảng, thêm dependency, sửa nghiệp vụ hoặc triển khai module mới lên production.
