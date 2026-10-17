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

## 3. Mô hình dữ liệu: 4 bảng mới

### 3.1. tasks — yêu cầu/công việc độc lập

| Cột | Kiểu đề xuất | Ý nghĩa |
| --- | --- | --- |
| id | BIGINT PK AUTO_INCREMENT | Định danh; mã hiển thị `TASK-<id>` tính từ ID, không cần bảng sinh mã |
| type | VARCHAR(20) | FEATURE / BUG / IMPROVEMENT / TASK |
| title | VARCHAR(200) | Tiêu đề |
| description | TEXT | Mô tả |
| module_code | VARCHAR(80) NULL | Mã từ danh mục module cho phép, null = Khác |
| status | VARCHAR(20) | INBOX / TODO / IN_PROGRESS / REVIEW / DONE / REJECTED / CANCELLED |
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
| rejection_reason | VARCHAR(2000) NULL | Lý do từ chối hiện tại |
| rejected_at, rejected_by | DATETIME NULL, BIGINT NULL FK users | Thời điểm/người từ chối hiện tại |
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

### 3.4. user_notifications — thông báo trong CMS

Rà soát tên file và migration hiện tại chưa thấy module/bảng notification dùng chung. Đề xuất một bảng dùng chung cho CMS, V1 phát sinh sự kiện từ tasks; cần kiểm tra schema main trước khi tạo để tránh trùng bảng đã bổ sung ở nhánh khác. Không cần bảng subscribers hoặc outbox riêng cho thông báo in-app.

| Cột | Kiểu/ý nghĩa |
| --- | --- |
| id | BIGINT PK AUTO_INCREMENT |
| recipient_user_id | BIGINT NOT NULL FK users |
| actor_user_id | BIGINT NULL FK users |
| module, entity_type, entity_id | VARCHAR(50), VARCHAR(50), BIGINT; V1 tasks/task/task.id |
| event_type | VARCHAR(50), enum do backend cho phép |
| event_key | VARCHAR(100) ASCII NOT NULL; định danh event bền vững, không UUID mới khi retry |
| title | VARCHAR(255), nội dung tối thiểu tại thời điểm event |
| created_at | DATETIME NOT NULL |
| read_at | DATETIME NULL, null = chưa đọc |

Unique `(recipient_user_id,event_key)`; index `(recipient_user_id,read_at,id)` và `(recipient_user_id,id)`. Không FK entity_id đa module; V1 backend kiểm tra task tồn tại và quyền truy cập khi đọc. Link đích sinh từ module/entity ID phía server, không nhận URL tùy ý từ client. Không thêm bảng notification_settings ở V1.

### 3.5. Quy ước tương thích DB

InnoDB, utf8mb4; khóa FK cùng kiểu/signedness với bảng đích. `SET NAMES utf8mb4` rõ ràng khi áp dụng SQL để tránh lỗi tiếng Việt đã gặp. Dùng DATETIME do service gán; không phụ thuộc JSON column/function, CHECK được DB thực thi, generated columns, nhiều TIMESTAMP auto-update hoặc index vượt 767 byte. Giới hạn enum, độ dài và state transition được service kiểm tra bắt buộc. FK dùng RESTRICT, tài khoản bị khóa thay vì xóa mất lịch sử.

## 4. Phân quyền và tính nhất quán

Dùng resource `tasks`, action `view/create/update/comment/manage` theo đặc tả. Options người phụ trách chỉ trả tài khoản active có view và update hoặc manage; không trả toàn bộ user entity/password. Người tạo lấy từ auth, không tin reporter_user_id từ client.

POST luôn đặt status=INBOX, reporter=current user. Người tạo được chọn ưu tiên ban đầu; assignee/due_date chỉ nhận khi có manage. PATCH của người tạo ở INBOX/REJECTED chỉ cho type/title/description/module/source_path và các trường mô tả theo loại; priority/due_date thuộc manage. Với người được giao, PATCH chỉ cho resolution/blocked_reason/delivery_environment/delivered_at. Từ chối field ngoài allowlist, không silently bỏ qua.

duplicate_of_id chỉ dùng khi chuyển CANCELLED vì trùng: phải tồn tại, khác chính nó, chưa lưu trữ và không tạo chu trình liên kết. Với V1 yêu cầu trỏ thẳng task gốc không có duplicate_of_id để tránh chuỗi gián tiếp. Khôi phục CANCELLED về INBOX xóa liên kết trùng hiện hành và lưu lý do trong audit.

- Mọi endpoint phải kiểm tra view trước, sau đó kiểm tra action và quan hệ với task.
- tasks.manage điều phối toàn bảng; tasks.update chỉ sửa kết quả/cờ chặn/chuyển xử lý cho task mình phụ trách.
- tasks.create cho sửa nội dung nghiệp vụ của chính mình ở INBOX/REJECTED, gửi lại từ REJECTED và thực hiện nghiệm thu/trả lại/mở lại theo đặc tả.
- tasks.comment thêm bình luận/tệp khi task chưa lưu trữ; sửa/gỡ bình luận chính mình. Người tải lên được gỡ tệp mình tải; manage kiểm duyệt toàn bộ.
- DONE và CANCELLED khóa thông tin nghiệp vụ nhưng cho trao đổi/tệp đến khi lưu trữ. Mở lại/khôi phục trước khi sửa.
- Lịch sử task trả qua endpoint đã lọc module/entity/id và danh sách field an toàn; không yêu cầu cấp quyền xem toàn bộ audit_logs cho người dùng task.

Update dùng `WHERE id=:id AND version=:expected_version`, tăng version 1; không cập nhật được trả 409 TASK_VERSION_CONFLICT. State transition dùng cùng cơ chế; client không được PATCH status/assignee/archived ngoài endpoint chuyên biệt.

Tạo task/comment có request_key UUID do client giữ suốt lần gửi lại; unique key bảo vệ race. Cùng khóa + cùng hash trả lại kết quả cũ; cùng khóa khác hash trả 409 IDEMPOTENCY_CONFLICT. Unique violation phải xử lý ngoài transaction đã rollback rồi đọc bản ghi đã commit, không tiếp tục transaction lỗi.

Mutation task và ghi audit phải cùng transaction/JDBC connection. Cần xác minh propagation của AuditLogDao trong test rollback; không coi gọi AuditService sau commit là atomic. Nếu cần, gọi DAO trong service với actor/context truyền từ controller và đánh dấu request đã audit để MutationAuditFilter không ghi trùng. Không tạo bảng audit mới.

Audit action: CREATE, UPDATE, ASSIGN, TRANSITION, REJECT, RESUBMIT, REQUEST_CHANGES, BLOCK, UNBLOCK, ARCHIVE, RESTORE, COMMENT_CREATE/EDIT/REMOVE, ATTACHMENT_ADD/REMOVE. Mọi activity gắn entity_type=task, entity_id=task.id; detail_ref trỏ comment/attachment. Ghi trạng thái trước/sau và lý do. Khi mở lại, xóa completed_at/by hiện hành nhưng lịch sử hoàn thành cũ còn trong audit.

### 4.1. Giao việc

`/assign` kiểm tra task chưa archived và status thuộc INBOX/TODO/IN_PROGRESS/REVIEW. Người nhận active, có view + (update hoặc manage). manage được giao/đổi người; assignee=null chỉ hợp lệ tại INBOX/TODO. Chỉ có update được tự nhận khi status=TODO và assignee hiện tại null; assignee gửi lên phải bằng current user. Hai người nhận đồng thời dùng expected_version nên chỉ một người thành công.

Đổi người tại IN_PROGRESS/REVIEW bắt buộc note bàn giao 5–2.000 ký tự; những lần giao khác note tùy chọn tối đa2.000. POST task có assignee do manage chọn cũng phát thông báo giao việc, gộp người nhận cùng sự kiện tạo tránh gửi hai noti cho một người. Giao lại đúng người hiện tại là no-op sau kiểm tra quyền/version, không tăng version hoặc gửi noti mới. Server trả allowed_actions.claim/assign/unassign để UI hiển thị đúng.

Audit lưu assignee cũ/mới và note trong cùng transaction với task + notification; không thêm bảng task_assignments. GET assignees nhận keyword tìm tên/email/mã nhân viên, chỉ trả id/name/email/employee_code và avatar nếu có; không trả trường nhạy cảm.

### 4.2. Reject và gửi lại

| Từ → Đến | Quyền và dữ liệu |
| --- | --- |
| INBOX/TODO → REJECTED | manage; UI ô Lý do từ chối / Ghi chú ánh xạ reason, trim dài 5–2.000, bắt buộc cả FE và BE; server gán rejected_by/at và rejection_reason |
| REJECTED → INBOX | reporter có create hoặc manage; reason mô tả bổ sung 5–2.000; task chưa archived; validate nội dung cơ bản |
| REVIEW → IN_PROGRESS | reporter có create hoặc manage; reason bắt buộc, action REQUEST_CHANGES |

reason thiếu/trống trả400 field_errors.reason, không đổi status, không ghi audit thành công hoặc gửi noti. Reject lưu nguyên note hợp lệ vào rejection_reason và audit; thông báo mời người gửi mở task xem note. Không cho assignee chỉ có update tự reject hoặc tự xác nhận kết quả không đạt. Gửi lại giữ ID, xóa rejection_reason/rejected_at/rejected_by hiện hành và assignee/due_date, lưu đầy đủ trước/sau trong audit. Lịch sử mỗi lần từ chối vẫn đọc được. Cờ blocked_reason được xóa khi REJECTED/CANCELLED/DONE; không giữ cờ chặn ở trạng thái kết thúc.

REJECTED loại khỏi overdue và năm cột board mặc định; API trả count riêng cho lối tắt, filter status=REJECTED dùng list. archived cho phép DONE/REJECTED/CANCELLED. Trạng thái archived phải được restore trước mọi sửa/chuyển; restore không tự gửi lại. Thời điểm/lý do reject không nhận qua PATCH.

### 4.3. Phát sinh thông báo

- Bảng nhận theo ma trận mục6 đặc tả; resolve recipients từ users và quyền thực tế. Gộp IDs, bỏ actor và tài khoản inactive/không có view.
- Cùng transaction: mutation task/comment/tệp → audit → INSERT user_notifications cho từng người nhận → commit. Nếu ghi noti lỗi phải rollback cả mutation; UI chỉ báo thành công sau commit.
- event_key xác định theo đối tượng/sự kiện: `task:{id}:version:{version}:transition`, `comment:{id}:created`, `attachment:{id}:created`. Một update đổi nhiều field được tổng hợp thành một event và một noti/người; không bắn riêng từng field.
- Retry create/comment/upload trả bản ghi cũ không phát lại event. Transition retry với expected_version cũ trả 409; client refetch, không tạo thông báo thứ hai. Unique recipient/event_key là lớp bảo vệ cuối.
- Không dùng mutation filter để phát noti lần thứ hai; logic recipient nằm trong service.
- Không gửi thông báo ngoài app trong transaction; nếu sau này thêm email/push mới thiết kế outbox và retry riêng.

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
| POST /tasks/{id}/assign | assignee_user_id nullable, expected_version, note; manage giao người khác, update chỉ tự nhận TODO chưa được giao |
| POST /tasks/{id}/transition | target_status, expected_version, reason, resolution/delivery nếu bước đó cần |
| POST /tasks/{id}/archive | expected_version; chỉ DONE/REJECTED/CANCELLED |
| POST /tasks/{id}/restore | expected_version; gỡ lưu trữ, giữ trạng thái nghiệp vụ |
| GET/POST /tasks/{id}/comments | Cursor đọc hoặc tạo có request_key |
| PATCH/DELETE /tasks/{id}/comments/{commentId} | Kiểm tra comment thuộc task; expected_version |
| GET/POST /tasks/{id}/attachments | Danh sách hoặc multipart, request_key |
| GET /tasks/{id}/attachments/{attachmentId}/preview | Ảnh PNG/JPEG/WebP đã kiểm tra quyền, trả inline; PDF không có preview ảnh |
| GET/DELETE /tasks/{id}/attachments/{attachmentId} | Download có auth hoặc ẩn metadata; kiểm tra FK |
| GET /tasks/{id}/activity | Lịch sử đã lọc an toàn, cursor |

GET /tasks size mặc định 20, tối đa 100. Board sort priority theo CASE rồi created_at/id tăng dần; cursor chứa tuple sort và filter fingerprint, validate server. Không OFFSET toàn bộ bảng để tải thêm card. Khi task đổi ưu tiên/trạng thái trong lúc phân trang, FE dedupe theo ID và refetch cột; count phản ánh thời điểm truy vấn, không cam kết snapshot xuyên nhiều HTTP request.

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

### 5.1. API thông báo dùng chung

| Method / path | Ý nghĩa |
| --- | --- |
| GET /notifications?unread=&cursor=&size= | Danh sách của người đăng nhập; size 20, max 100, id DESC |
| GET /notifications/unread-count | Số chưa đọc mà người dùng còn được phép truy cập + high_watermark_id |
| PATCH /notifications/{id}/read | `{ "read": true/false }`; chỉ chủ thông báo; idempotent |
| POST /notifications/read-all | `{ "through_id": 123 }`; đánh dấu đến watermark đã tải, không nuốt noti mới đến |

Không nhận recipient_user_id từ client. Mỗi truy vấn list/count phải giới hạn recipient=current user và quyền module/entity hiện tại; count loại thông báo đã mất quyền giống list. Với V1 mọi task dùng chung view nên có thể lọc bằng quyền tasks.view; phải kiểm tra tồn tại entity, không chỉ tin metadata. API read không thuộc user trả 404. Phần chuông dành cho tài khoản đăng nhập, không cần cấp quyền audit hoặc tasks.manage. Khi mất tasks.view, noti task không hiển thị nội dung hoặc đếm vào badge; khi cấp lại có thể xuất hiện lại.

read-all dùng id <= through_id và recipient=current user; chỉ đánh dấu các notification còn được phép xem, không đánh dấu dữ liệu đã mất quyền. Thao tác read/read-all không tạo audit nghiệp vụ hay phát notification mới.

GET detail task trả thêm rejection_reason/by/at và allowed_actions gồm reject/resubmit/request_changes khi phù hợp. /transition dùng target_status=REJECTED hoặc INBOX theo cạnh hợp lệ; reason bắt buộc cho reject/resubmit/request_changes. Không cần endpoint reject thứ hai trùng logic.

## 6. Frontend và thao tác Kanban

- Feature `src/features/notifications`, chuông chung header và route `_authenticated/notifications`; Query cache theo current user, xóa khi logout/đổi tài khoản.
- Popover không tự mark all; click noti mở `/tasks?task={id}` và PATCH read; lỗi mark-read không ngăn xem task, báo nhẹ và refetch badge. Tách trạng thái sự kiện cũ trong noti với trạng thái task hiện tại.
- Reject dialog cần lý do; detail REJECTED hiện banner người/thời điểm/lý do và nút Sửa/Gửi lại theo quyền. REVIEW hiện Xác nhận hoàn thành/Yêu cầu sửa lại.
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

### 6.1. Ảnh trong mô tả

- Dùng token Markdown nội bộ `![chú thích](attachment:123)`; server parse bằng parser, chỉ chấp nhận attachment tồn tại, chưa gỡ, đúng task và MIME ảnh hợp lệ. Không lưu base64/blob URL/object URL vào DB.
- Renderer resolve ID thành preview endpoint có auth, giữ alt text, thumbnail và lightbox. Với auth bearer, tải blob bằng API client rồi tạo object URL và revoke khi unmount; không nhúng token vào query URL. Không load ảnh remote tùy ý từ Markdown.
- Editor nhận input file/drop/paste ảnh, tránh chặn paste văn bản bình thường. Ảnh mới có placeholder local; upload thành công mới đổi thành attachment token. Validate trước save không còn placeholder chưa resolve.
- Form tạo: lưu nội dung văn bản trước bằng request_key; dùng ID/version trả về để upload ảnh; sau đó PATCH mô tả chứa token với expected_version. Lỗi ở bước sau hiển thị “Task đã tạo, ảnh/mô tả ảnh chưa lưu” và cho retry vào đúng ID; không gửi POST task lần nữa. Khi409 giữ draft và yêu cầu đối chiếu trước khi lưu, không ghi đè.
- Quyền upload được mở thêm cho reporter có create tại INBOX/REJECTED và manage tại trạng thái chưa kết thúc để hỗ trợ ảnh mô tả, ngoài comment theo quyền đính kèm hiện tại. Quyền sửa mô tả vẫn độc lập với quyền upload.
- Thêm/gỡ attachment và PATCH mô tả khóa cùng hàng task trong transaction khi kiểm tra reference/quota. Không cho DELETE tệp còn token trong description; trả409 ATTACHMENT_IN_USE. PATCH đồng thời phải kiểm tra lại tệp chưa bị gỡ dưới khóa; không để race sinh ảnh hỏng.
- Upload đã thành công nhưng hủy sửa để lại attachment hợp lệ, không bị job file mồ côi xóa vì có metadata. Gỡ token không xóa tệp vật lý. Không thêm bảng ảnh riêng.

## 7. Upload, bảo mật và lưu trữ

- Kiểm tra auth và quyền task trước đọc body lớn; giới hạn tổng 10 tệp còn hiệu lực,10 MB mỗi tệp, kiểm tra cả MIME/signature thật.
- Chỉ PNG/JPEG/WebP/PDF; không nhận HTML/SVG/script. Tên tệp hiển thị escape, đường dẫn ngẫu nhiên chống traversal.
- Ghi tệp staging trước; trong transaction khóa hàng task để kiểm tra quota và lưu metadata; publish/đổi tên nguyên tử cùng filesystem trước commit. Lỗi rollback phải dọn tệp; có job dọn file mồ côi quá 24h đối chiếu metadata, không xóa file đang được tham chiếu.
- Download qua endpoint có auth; `Content-Disposition: attachment` cho PDF, ảnh preview qua endpoint riêng được bảo vệ, `X-Content-Type-Options: nosniff`, cache private. Không để static web server expose thư mục upload.
- Ẩn tệp làm endpoint thường trả404; dữ liệu vật lý giữ theo chính sách backup trước khi có tác vụ dọn có chủ đích. Không hứa undo nếu đã xóa vật lý.
- Markdown render disable raw HTML, link chỉ http/https hoặc route nội bộ; không cho javascript URL. source_path loại token và query nhạy cảm bằng whitelist.
- Không bổ sung cơ chế fetch URL phía server từ nội dung task; không upload thẳng sang dịch vụ công khai.

## 8. Hiệu năng và vận hành

Board truy vấn count theo group status và giới hạn card từng cột, batch join users/count comments/attachments theo tập ID để tránh N+1. Không gửi description dài trên card. DONE cũng chỉ tải 20 card đầu; lưu trữ chủ động giúp giảm nhiễu, không tự mất việc cũ.

V1 refetch khi focus và sau mutation; polling trạng thái/task đang mở và thông báo mỗi 30 giây khi tab hiển thị, dừng khi tab ẩn hoặc logout; chưa cần WebSocket. Không ghi đè form dirty khi refetch. Ngày hiện tại/quá hạn tính server theo Asia/Ho_Chi_Minh.

Backup DB và thư mục tệp cùng quy trình. Log request ID và task ID, tránh ghi nguyên mô tả/tệp chứa dữ liệu nội bộ. Audit không bị xóa khi task lưu trữ.

## 9. Kế hoạch kiểm thử

| Nhóm | Ca cần chứng minh |
| --- | --- |
| Giao việc/ảnh | Giao khi tạo, giao lại có note, tự nhận chỉ TODO chưa giao, race hai người nhận; paste/drop/upload/preview ảnh; retry không tạo task trùng; attachment task khác bị từ chối; race xóa tệp/chèn ảnh không sinh reference hỏng |
| Reject/noti | Chỉ manage reject; reject/gửi lại/request_changes bắt buộc lý do; gửi lại giữ ID và audit, reset phân công/hạn; archived phải restore trước; REJECTED không quá hạn |
| Thông báo | Đúng ma trận nhận, bỏ actor/gộp trùng; retry không nhân đôi; lỗi insert noti rollback mutation; mất quyền/inactive không lộ dữ liệu; read của user khác 404; read-all watermark không đánh dấu noti mới; F5 giữ đã đọc; count khớp list |
| State machine | Mỗi cạnh hợp lệ; chặn mọi cạnh khác; REVIEW cần resolution; DONE cần xác nhận triển khai; mở lại giữ audit |
| Quyền | Người tạo, người được giao, người khác, manage, không view; gọi API trực tiếp vẫn bị chặn; options không lộ user secret |
| Race | Hai update cùng version chỉ một thành công; retry create/comment cùng khóa chỉ một bản ghi; cùng khóa khác payload bị409 |
| Transaction | Lỗi ghi audit rollback mutation; không có audit thành công cho mutation rollback |
| Board | Count khớp filter; tải thêm không hiện ID trùng; đổi cột cập nhật hai count; lỗi mạng/409 hoàn nguyên card |
| Upload | Hai upload đồng thời không vượt quota; signature giả/oversize/traversal bị từ chối; tệp task khác/đã gỡ không tải được |
| XSS | Tiêu đề, Markdown, bình luận, tên tệp và link độc hại không thực thi |
| UX | F5/deep link/Back giữ đúng context; ESC form chưa lưu; kéo thả và bàn phím; mobile |
| DB | Migration trên MariaDB 5.5 và DB dev; tiếng Việt roundtrip; FK/index phù hợp, không JSON SQL |
| Regression | Auth/audit hiện tại, menu Nhân sự đầu tiên, công kho và lương không thay đổi |

Mục tiêu tải thử: 10.000 task và 50.000 bình luận, trang board đầu phản hồi API p95 dưới 1 giây trong môi trường staging được ghi rõ cấu hình; đây là mục tiêu nghiệm thu cần đo, không phải kết quả đã có.

## 10. Các bước triển khai sau khi duyệt tài liệu

1. Đồng bộ main, tạo nhánh riêng `feature/internal-task-kanban` cho các repo liên quan; chuyển hai tài liệu sang nhánh đó, không trộn implementation với nhánh lương.
2. Hoàn thiện migration 4 bảng, permission, service/state machine/audit và test backend.
3. Làm danh sách, tạo/sửa/chi tiết và filter URL trước; sau đó board/DnD dùng cùng API nghiệp vụ.
4. Bổ sung bình luận, tệp bảo vệ quyền, activity và chuông/trang thông báo; kiểm tra upload/rollback/concurrency.
5. Hoàn thiện và QA trên local trước: FE/API/shared + DB local, chạy đầy đủ giao việc/reject note/upload ảnh/noti/quyền và báo cáo kết quả. Nếu có staging thì kiểm tra bổ sung sau local; không dùng production để thử hoặc thêm task giả.
6. Chỉ sau khi local hoàn tất và các kiểm thử đạt mới deploy production: backup, migration additive, deploy API rồi CMS; cấp quyền ADMIN trước, xác nhận smoke rồi mới mở quyền cho nhóm nội bộ.
7. Nếu rollback ứng dụng, giữ bảng/tệp mới để không mất dữ liệu đã nhập; không rollback bằng DROP TABLE.

Lần làm hiện tại chỉ viết đặc tả và TDD. Chưa tạo bảng, thêm dependency, sửa nghiệp vụ hoặc triển khai module mới lên production.
