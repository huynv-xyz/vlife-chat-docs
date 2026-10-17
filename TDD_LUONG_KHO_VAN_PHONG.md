# TDD — Lương kho, văn phòng, nghỉ phép và ngày phép

Cập nhật: 17/10/2026. Phiên bản 1.7 — rút gọn toàn bộ mô hình dữ liệu. Trạng thái: thiết kế đề xuất, chưa triển khai schema/code.

Nghiệp vụ: [Phân tích chuyển chức năng cũ](phan-tich-chuyen-luong-kho-van-phong.md).

## 1. Phạm vi

Chuyển chức năng cũ: hồ sơ lương, Công việc/Chi tiết công việc, Đóng thùng/Chi tiết đóng thùng, công/tăng ca, phát sinh, Nghỉ phép, Ngày phép, bảng lương kho/văn phòng, in/xuất và lịch sử. Làm lương kho trước.

Nhập sản lượng qua phiếu công việc/đóng thùng. Không tích hợp sổ kho trong đợt này. Module độc lập với lương sale: không gọi pipeline sale, không ghi `payrolls`, không dùng các màn hình/cấu hình lương sale.

Tên bảng dùng đầy đủ theo nghiệp vụ: `employee_work_*`, `employee_leave_*`, `employee_time_records`, `employee_pay_items` và `employee_payroll_*`. API và route `/employee-payroll`. Dùng lại bảng `employees` và màn hình/API nhân viên hiện có; bổ sung cột còn thiếu theo thiết kế mục 5.1. Không tạo bảng hồ sơ nhân viên thứ hai.

## 2. Kết quả rà soát thiết kế

Bản trước chia 35 bảng cho cả những dữ liệu luôn đọc/ghi cùng nhau. Bản này giảm còn **12 bảng mới và mở rộng `employees`**, giữ chi tiết quan hệ ở chỗ cần báo cáo theo nhân viên và dùng JSON có cấu trúc cho cấu hình/snapshot.

| Mục | Trước | Sau | Cách gộp |
| --- | ---: | ---: | --- |
| 5.1 Hồ sơ và đơn giá | 10 | 2 | Dùng employees; thêm cột thiếu; danh mục và đơn giá dùng chung |
| 5.2 Đầu vào tính lương | 7 | 4 | Phiếu công việc/đóng thùng chung bảng; phân bổ chung bảng; công/tăng ca chung bảng |
| 5.3 Nghỉ phép/ngày phép | 8 | 3 | Loại phép trong danh mục có phiên bản đi kèm mã nguồn; một bản nghỉ theo ngày; số dư và giao dịch phép riêng |
| 5.4 Kết quả và lịch sử | 10 | 3 | Snapshot/breakdown/lỗi nằm trong phiên/kết quả; dùng audit_logs hiện có |
| Tổng | 35 | 12 | Không tạo bảng giữ chỗ nghỉ, lô phép hoặc điều phối cấu hình riêng |

Giữ 12 bảng mới cho danh mục, dữ liệu công, số dư phép và kết quả lương; hồ sơ nhân viên dùng employees hiện có. Không đưa tất cả vào một bảng JSON tổng hợp. Số màn hình không quyết định số bảng: Công việc và Đóng thùng vẫn là hai màn hình dù dùng chung cấu trúc lưu trữ.

## 3. Tổ chức mã

Backend Java/Micronaut theo repo hiện tại:

- Entity/DAO: `vl-shared/.../jdbc/entity/employeepayroll`, `jdbc/dao/employeepayroll`.
- Service: `vl-shared/.../service/employeepayroll` gồm EmployeeConfig, EmployeeWork, TimePayItem, Leave, Payroll và Import.
- Calculator kho/văn phòng tách riêng, nhận snapshot và trả breakdown, không truy vấn DB.
- Controller: `vl-api/.../controller/employeepayroll`; dùng cơ chế auth, quyền và `ApiResponse` hiện có.
- Migration: `vl-api/src/main/resources/db/migration`.
- Frontend: `vl-cms/src/features/employee-payroll`, route trong `_authenticated/employee-payroll`.

React/TypeScript, TanStack Router/Query, React Hook Form, Zod, ExcelJS theo phiên bản hiện tại. Không bổ sung framework hay nâng dependency để làm module.

## 4. Quy ước

- Sản lượng `DECIMAL(20,6)`, giờ/ngày công `DECIMAL(8,2)`, tiền trung gian `DECIMAL(20,6)`, tiền VND kết quả `DECIMAL(20,0)`; backend BigDecimal, API decimal là chuỗi.
- Ngày nghiệp vụ DATE, kỳ YYYY-MM; audit timestamp theo convention hệ thống, hiển thị Asia/Ho_Chi_Minh.
- Tên bảng snake_case số nhiều, mô tả nghiệp vụ đầy đủ; cột FK dùng tên đối tượng như work_type_id, work_sheet_id, employee_id. Bảng có audit và version; update yêu cầu expected_version. Trạng thái và chuyển kỳ được kiểm tra tại service, không dùng CRUD generic để bỏ qua nghiệp vụ.
- Cấu hình hiệu lực `[from,to)`, không chồng khoảng trên cùng khóa nghiệp vụ. Snapshot lưu cả nội dung và ID/version.
- JSON phải có schema_version và DTO validate; không nhận JSON tùy ý hoặc script công thức. Trường dùng filter/index vẫn là cột thường.
- Mỗi bảng cần chuyển dữ liệu có `legacy_source`, `legacy_id`, `import_batch`, `source_hash`; unique `(legacy_source, legacy_id)` trên từng bảng. Source phân biệt loại bảng cũ khi gộp.
- Các thao tác tạo có request_key: lưu khóa này trên bản ghi đích và unique theo loại thao tác. Batch dùng batch_key + số thứ tự phần tử; giao dịch phép dùng event_key. Retry cùng khóa nhưng khác payload phải báo xung đột.
- Bản ghi đã tham gia lương không hard delete; hủy/ngừng hiệu lực và giữ lịch sử.

### Quy ước thuật ngữ theo nghiệp vụ

| Thuật ngữ | Dùng cho | Tên đã thống nhất |
| --- | --- | --- |
| sheets / sheet_items | Phiếu công việc và các dòng chia sản lượng của phiếu | employee_work_sheets / employee_work_sheet_items |
| requests | Đơn nghỉ phép có trạng thái gửi/duyệt/hủy | employee_leave_requests |
| records | Ghi nhận công/tăng ca theo người/ngày/ca | employee_time_records |
| balances / transactions | Số dư phép và giao dịch cộng/trừ | employee_leave_balances / employee_leave_transactions |
| pay_items | Các khoản thu nhập/khấu trừ phát sinh theo nhân viên/kỳ | employee_pay_items |
| periods / runs / results | Kỳ lương, lần tính và kết quả | employee_payroll_periods / employee_payroll_runs / employee_payroll_results |

Dùng cùng thuật ngữ trong tên bảng, FK, API và mapping migration. Không dùng entries/records/requests thay thế nhau cho cùng đối tượng. Đổi tên không thay cấu trúc: mỗi đơn nghỉ vẫn là một ngày với hai buổi; thao tác nhiều ngày dùng batch_key như thiết kế hiện tại.

## 5. Mô hình dữ liệu — 12 bảng mới và mở rộng employees

Đã rà tên 12 bảng với mã Java/SQL hiện có trong `vl-shared/src/main` và `vl-api/src/main`: chưa thấy trùng tên. Cần kiểm tra schema DB thực tế trước tạo migration. `employee_work_*` chứa cả công việc và đóng thùng, phân biệt qua kind; `employee_payroll_*` dành cho kỳ/kết quả lương kho và văn phòng. Nhật ký sử dụng `audit_logs` hiện có.

### 5.1. Dùng lại employees; thêm 2 bảng danh mục/đơn giá

| Bảng | Nội dung | Khóa/ràng buộc |
| --- | --- | --- |
| `employee_work_types` | code, name, kind `CATEGORY/WORK/CRATING`, parent_id, unit, rate_scope `CATEGORY/ITEM`, active | unique code; parent của WORK chỉ là CATEGORY; không chu trình |
| `employee_work_rates` | work_type_id, day_shift_type, price, coefficient, divisor, effective_from/to | FK type; không chồng hiệu lực theo type/ca; divisor > 0 |

**Hồ sơ dùng lại `employees`:** đã xác minh từ `Employee.java` các cột `code`, `name`, `labor_type`, `basic_salary`, `allowance_salary`, `insurance_base`, `dependent_count`, `personal_insurance_from/to`, `personal_bhxh/bhyt/bhtn/kpcd`, `joined_at`, `left_at`, `status`. Dùng lại cột đúng ý nghĩa; không sao chép sang bảng hồ sơ khác. Trong công thức, base_salary ánh xạ tới employees.basic_salary; ngày vào/nghỉ việc dùng joined_at/left_at.

Đây là **các cột bổ sung trên bảng employees**, không phải bảng mới. Cột thuộc nhóm nghiệp vụ lương dùng payroll_ khi cần làm rõ ý nghĩa; không dùng tiền tố staff_. Kiểm tra schema DB trước migration:

| Cột | Kiểu đề xuất | Mục đích |
| --- | --- | --- |
| `payroll_group` | VARCHAR(20) nullable trong giai đoạn chuyển | Chọn bộ tính SALE/WAREHOUSE/OFFICE và lọc đúng nhân viên; hoàn tất mapping trước tính lương |
| `salary_fund` | DECIMAL(20,6) NOT NULL DEFAULT 0 | Phần lương quỹ của văn phòng: cộng với basic_salary để tính căn cứ lương ngày theo code cũ; không có thì 0 |
| `payroll_allowances` | JSON nullable | Lưu riêng từng khoản ăn, điện thoại, đồng phục, nhà trọ… để tính đúng căn cứ và tránh cộng trùng; `{code, amount, treatment, tax_treatment}` |
| `payroll_effective_from` | DATE nullable | Ngày thiết lập lương hiện hành bắt đầu áp dụng; không thay joined_at. Kỳ trước ngày này phải dùng snapshot/đầu vào lịch sử |
| `version` | BIGINT NOT NULL DEFAULT 0, nếu chưa có | Số lần cập nhật hàng, phục vụ optimistic lock: hai người sửa cùng hồ sơ thì bản gửi sau bị báo xung đột; không phải thông số lương |

Phụ cấp JSON phải validate code duy nhất, số tiền và treatment IN_DAY_BASE/ADD_GROSS/DEDUCTION_ONLY. `allowance_salary` hiện là tổng phụ cấp dùng ở sale; với kho/văn phòng chi tiết payroll_allowances là nguồn tính, không cộng thêm allowance_salary vào gross. Không tự chuyển tổng allowance_salary thành phụ cấp ăn/đồng phục/điện thoại khi chưa có mapping.

`payroll_group` là nhóm lương, không dùng thay `labor_type` (loại lao động). Nhân viên chưa phân nhóm phải xuất hiện trong báo cáo cần xử lý trước vận hành. MVP chưa hỗ trợ đồng thời nhiều nhóm hoặc chuyển nhóm giữa tháng.

**Tác động cần xử lý trên lương sale:** `PayrollService.aggregateScopes` hiện UNION cả nhân viên status=1 có basic_salary/allowance_salary/insurance_base khác 0. Khi bổ sung nhân viên kho/văn phòng vào employees, phải lọc nhóm SALE ở tập nhân viên cuối cùng của luồng sale; rà soát các bước upstream và truy vấn tính theo nhân viên khác để không kéo kho/văn phòng vào ngân sách/kết quả sale. Không thay công thức sale hoặc đưa kết quả kho/văn phòng vào bảng sale.

Migration phân nhóm theo dữ liệu đã đối chiếu, không mặc định mọi nhân viên là SALE. Bật lọc SALE sau khi backfill đầy đủ và so sánh tập nhân viên/kết quả sale trước/sau. Đây là điều chỉnh bắt buộc để dùng chung employees an toàn, nằm trong phạm vi triển khai.

**Danh mục:** cùng bảng chứa nhóm việc, công việc và loại đóng thùng. Phiếu trỏ đến WORK hoặc CRATING; nhóm việc lấy từ parent. Đơn giá nhóm việc áp theo CATEGORY khi rate_scope=CATEGORY; áp theo WORK khi rate_scope=ITEM. Không fallback âm thầm giữa hai cách. Đóng thùng dùng coefficient=1, divisor=1; công việc giữ divisor cấu hình, mẫu cũ là 1000.

**Quy tắc và danh mục đi kèm mã nguồn:** không tạo bảng settings tổng quát. Trong phạm vi rà soát mã nguồn mới chưa thấy danh mục ngày lễ/loại phép có thể dùng lại. Đợt chuyển này xuất danh mục từ code/database cũ thành tệp dữ liệu có schema và phiên bản, ví dụ `payroll-rules/<version>/holidays.json`, `leave-types.json`; đặt trong resources của module. Kế toán đối chiếu trước triển khai, server validate khi tải. Loại phép dùng mã ổn định; ngày lễ phải có năm áp dụng, không tự coi danh sách năm trước là đúng cho năm mới. Lịch chuẩn và cách quy đổi công bám luồng cũ đã xác nhận.

Công thức, hệ số tăng ca, quy tắc khấu trừ và làm tròn triển khai trong calculator theo `formula_version`; thông số theo kỳ nằm trong bộ quy tắc có phiên bản đi kèm mã nguồn, không dùng hằng số hiện tại cho mọi kỳ lịch sử. Chưa có UI sửa các quy tắc này; thay đổi cần cập nhật phiên bản, kiểm thử và triển khai. Nếu chưa có quy tắc/ngày lễ cho kỳ được chọn thì báo thiếu dữ liệu trước khi tính. Snapshot lưu các giá trị thực tế đã dùng để giải thích kết quả.

MVP lưu thiết lập hiện hành trên employees và snapshot đầy đủ trên từng kết quả lương. Thay đổi thiết lập có hiệu lực đầu tháng, lưu before/after và ngày hiệu lực trong audit; không hỗ trợ đặt nhiều phiên tương lai trên một hàng employees. Kỳ cũ lấy snapshot của lần tính tương ứng khi cần đối chiếu/tính lại, không tự lấy lương hiện tại. Nếu kỳ cũ chưa có snapshot thì yêu cầu nhập/khôi phục bộ đầu vào lịch sử được xác nhận trước khi tính. Vào/nghỉ việc vẫn có thể giữa tháng; chuyển nhóm giữa tháng chưa hỗ trợ thì báo lỗi.

### 5.2. Công việc, đóng thùng, công và phát sinh — 4 bảng

| Bảng | Cột nghiệp vụ chính | Ràng buộc/index |
| --- | --- | --- |
| `employee_work_sheets` | kind `WORK/CRATING`, work_date, work_type_id, shift_code, quantity, unit, reported_hours, note, status | index kind/date/status; type phù hợp kind |
| `employee_work_sheet_items` | work_sheet_id, employee_id, quantity | unique work_sheet/employee; tổng bằng phiếu |
| `employee_time_records` | employee_id, work_date, slot_code, kind `ATTENDANCE_ADJUSTMENT/OVERTIME`, payable_days_delta, hours, source_refs_json, reason, status | unique employee/date/slot/kind; chỉ trường phù hợp kind được nhập |
| `employee_pay_items` | employee_id, period, effective_date, kind `INCOME/DEDUCTION`, code, amount, tax_treatment, reason, status, origin_result_id | index employee/period/status |

`employee_work_sheets` lưu phiếu tổng; `employee_work_sheet_items` lưu dòng chi tiết sản lượng chia cho từng nhân viên. Công việc và đóng thùng dùng cùng cặp bảng, phân biệt bằng kind. Vẫn giữ bảng `employee_work_sheet_items` vì phải lọc/tổng hợp sản lượng theo nhân viên thường xuyên.

Bảng thời gian gộp công điều chỉnh và tăng ca vì cùng quản lý theo người/ngày/ca. Không đưa nghỉ phép vào bảng này. Chấm công mặc định tính từ lịch và nghỉ, chỉ lưu điều chỉnh có lý do, không sinh dữ liệu công rỗng cho từng ngày.

Chi tiết phân bổ được sinh từ phiếu và không sửa trực tiếp. Phiếu DRAFT/CONFIRMED/CANCELLED; chỉ CONFIRMED tham gia tính. Không cần một bảng trạng thái/duyệt riêng.

### 5.3. Nghỉ phép và ngày phép — 3 bảng

| Bảng | Cột nghiệp vụ chính | Ràng buộc/index |
| --- | --- | --- |
| `employee_leave_requests` | employee_id, leave_date, morning, afternoon, leave_type_code, paid, deduct_balance, reason, status, policy_snapshot_json, approved_by/at, batch_key | index employee/date/status; ít nhất một buổi |
| `employee_leave_balances` | employee_id, leave_type_code, year, balance, version | unique employee/type/year |
| `employee_leave_transactions` | balance_id, leave_request_id nullable, operation, signed_days, event_key, reversal_of nullable, reason | unique event_key; index balance_id; giao dịch bất biến |

Một bản nghỉ theo ngày như `OnLeaveRegister` cũ; sáng/chiều mỗi buổi 0,5 ngày. UI nhập nhiều ngày tạo nhiều bản ghi cùng batch_key, xử lý trong một transaction khi lưu/duyệt cả nhóm. Không cần header đơn + bảng ngày riêng.

Loại phép và quy tắc lấy từ danh mục có phiên bản đi kèm mã nguồn ở mục 5.1; không có bảng cấu hình tổng quát hoặc hai bảng loại/chính sách phép riêng. Số dư dùng balance để hiển thị/khóa; giao dịch phép để đối soát và hoàn phép. Hai bảng này cần giữ vì chỉ lưu số dư sẽ mất căn cứ điều chỉnh.

Operation: OPENING, GRANT, USE, REFUND, ADJUST, TRANSFER_IN, TRANSFER_OUT, EXPIRE. Phép hết hạn/chuyển năm được ghi nhận bằng giao dịch do người phụ trách xác nhận. Không thiết kế lô phép, thứ tự tiêu thụ lô hoặc scheduler tích lũy tự động ở MVP; nếu nghiệp vụ thực tế cần hạn riêng từng đợt cấp thì phải bổ sung thiết kế trước khi bật tính năng đó.

Nghỉ có lương và trừ quỹ là hai thuộc tính độc lập. Không có bảng reservations: kiểm tra trùng sáng/chiều trong transaction đã khóa nhân viên và kỳ liên quan, xem mục 7.

### 5.4. Kỳ và kết quả — 3 bảng mới; dùng audit_logs hiện có

| Bảng | Cột nghiệp vụ chính | Ràng buộc/index |
| --- | --- | --- |
| `employee_payroll_periods` | period, payroll_group, status `OPEN/LOCKED`, input_revision, current_run_id | unique period/group |
| `employee_payroll_runs` | period_id, sequence, status, input_revision, request_key, rules_snapshot_json, errors_json, origin `NEW/LEGACY` | unique period/sequence; unique request_key |
| `employee_payroll_results` | run_id, employee_id, work_days, gross_total, deduction_total, net_total, input_snapshot_json, breakdown_json, errors_json | unique run/employee; index run |

Snapshot đầu vào và chi tiết khoản tiền lưu ngay trên kết quả nhân viên vì chỉ đọc cùng kết quả; không có bảng input_snapshots/result_lines riêng. Lỗi từng người nằm tại result, lỗi toàn kỳ nằm tại run. Số tổng là cột thường phục vụ bảng lương và báo cáo năm.

**Nhật ký dùng lại hạ tầng hiện có:** đã kiểm tra `audit_logs`, `AuditService`, `AuditLogDao` và `AuditCatalog`. Bảng hiện có đủ module, entity_type/id, action, old_values/new_values, changed_fields, changed_by/at, source_type, result_status, summary, request_id và detail_ref_type/id. Không tạo bảng audit riêng cho lương.

- Đăng ký module `employee_payroll` và các entity/action trong AuditCatalog; dùng giao diện nhật ký hiện có với bộ lọc module.
- Ghi sửa công, thay đơn giá, duyệt/hủy nghỉ, cấp/hoàn phép, tính/chốt/mở kỳ và import vào audit_logs. Lý do ghi trong summary và dữ liệu thay đổi; detail_ref trỏ kỳ/run hoặc bản ghi liên quan.
- Thao tác HTTP dùng AuditService.record/recordMaps. Các hàm này phụ thuộc ServerRequestContext; import/job không có request phải gọi AuditLogDao với actor/source_type/request_id được truyền rõ, tránh mất log.
- Ghi log SUCCESS cùng transaction nghiệp vụ khi dùng cùng kết nối; kiểm thử rollback để bảo đảm không lưu log thành công cho thao tác thất bại. Log FAILED/DENIED theo đường xử lý lỗi riêng và hạ tầng hiện có.
- Audit ở API và log ở service phải thống nhất một nơi ghi cho mỗi thao tác, tránh trùng. Bổ sung quyền/scope xem log lương để người xem log nghiệp vụ khác không mặc nhiên đọc được dữ liệu lương.
- Audit phục vụ truy vết thao tác. Snapshot lương và giao dịch cộng/trừ phép vẫn nằm trong bảng nghiệp vụ tương ứng để tính và đối soát, không dựng kết quả nghiệp vụ từ log.

Không có bảng import_batches/legacy_maps riêng. Mapping legacy nằm trên từng bản ghi đích; manifest JSON/CSV của lần chuyển lưu thành artifact có checksum, batch ID và báo cáo đối soát, audit ghi thao tác apply. Đây là công cụ chuyển dữ liệu theo đợt, chưa xây hệ thống quản lý ETL tổng quát.

## 6. Ghi công việc/đóng thùng và tăng ca

1. Validate loại phiếu, ngày, đơn vị, người, nhóm lương và lượng > 0.
2. Khóa các nhân viên và kỳ liên quan theo mục 9, kiểm tra kỳ OPEN và version.
3. Chia lượng cho N người với scale 6; phần dư phân phối theo employee_id tăng dần, bảo toàn tổng. Lưu phiếu và phân bổ cùng transaction.
4. Sửa phiếu thay toàn bộ chi tiết phân bổ; hủy giữ chi tiết để tra cứu nhưng loại khỏi tính mới. Tăng input_revision và đánh dấu run nháp bị ảnh hưởng STALE.
5. Chi tiết công việc/đóng thùng đọc `employee_work_sheet_items` JOIN `employee_work_sheets`; không có endpoint update chi tiết.

Giờ tăng ca kho đối chiếu theo người/ngày/ca trong `employee_time_records`. Nhiều việc một ca chỉ có một bản giờ; cùng giờ thì liên kết thêm nguồn, khác giờ thì báo cần xác nhận. Sửa/hủy nguồn đặt giờ liên quan về trạng thái cần kiểm tra, không tự lấy SUM/MAX. Tăng ca văn phòng nhập riêng qua UI nhưng dùng cùng bảng. Dữ liệu cũ không xác định ca được phải đưa vào báo cáo đối chiếu.

## 7. Nghỉ phép/ngày phép

Trạng thái đề xuất DRAFT → SUBMITTED → APPROVED/REJECTED; có CANCELLED. Người nhập hộ có quyền có thể chọn “Lưu và duyệt” để giữ thao tác gọn. APPROVED không sửa tại chỗ: hủy/hoàn rồi tạo bản thay thế có liên kết audit.

**Submit/duyệt:** khóa hàng employees của nhân viên rồi các kỳ liên quan theo mục 9; đọc lại trạng thái/version, lịch, hồ sơ, các bản nghỉ SUBMITTED/APPROVED trùng ngày và buổi. Loại buổi không làm việc theo lịch, báo rõ số ngày tính. Kiểm tra xung đột công/tăng ca; chặn khi chưa xử lý. SUBMITTED không trừ quỹ.

**Duyệt:** kiểm tra kỳ OPEN; lấy/tạo balance của năm ngày nghỉ, kiểm tra số dư và policy. Nếu deduct_balance=true, ghi USE và cập nhật balance trong cùng transaction. Event key theo leave_request_id/version/USE chống retry. Hai đơn duyệt đồng thời đi qua cùng hàng khóa nên không vượt số dư ngoài policy. API batch nhiều ngày/năm thành công toàn bộ hoặc rollback.

**Hủy:** ghi REFUND đối ứng USE gốc đúng một lần, cập nhật balance, chuyển trạng thái và audit. Nếu kỳ lương đã LOCKED phải mở kỳ có quyền trước hoặc ghi điều chỉnh kỳ sau. Nếu quỹ năm cũ đã chuyển/hết hạn thì chặn hoàn tự động và yêu cầu người phụ trách xác nhận phương án điều chỉnh các năm liên quan; không cộng bừa vào số dư năm hiện tại.

**Cấp/điều chỉnh/chuyển phép:** xem trước rồi apply với event_key ổn định. Chuyển năm ghi TRANSFER_OUT/IN cùng transaction. Phép tồn được nhập OPENING tại cutover hoặc dựng từ lịch sử, không kết hợp hai cách cho cùng phạm vi. Balance phải bằng SUM(signed_days); có truy vấn kiểm tra sai lệch.

## 8. Công thức lương

Calculator chỉ đọc snapshot đã capture, không đọc live DB. Các thông số cũ dùng làm mẫu đối chiếu; không tự seed thành chính sách production khi chưa được kế toán xác nhận.

### 8.1. Công tính lương

- D = công chuẩn toàn tháng theo lịch, phải > 0.
- E = công lịch trong thời gian nhân viên làm việc trong tháng.
- L = tổng ngày nghỉ hợp lệ; P = phần hưởng lương thuộc L; A = điều chỉnh công đã xác nhận, không trùng phần nghỉ.
- C = E − L + P + A. Kiểm tra công không âm/vượt giới hạn đã duyệt.

Không yêu cầu nhập đủ công từng ngày để thay công lịch. Nghỉ hưởng lương không tự sinh tiền công việc hoặc đóng thùng.

### 8.2. Kho

```text
Lương ngày      = base_salary / D
Lương công      = Lương ngày × C
Tiền công việc  = SUM(lượng phân bổ × giá × hệ số / divisor)
Tiền đóng thùng = SUM(lượng phân bổ × giá đóng thùng)
Tiền tăng ca    = SUM(Lương ngày / giờ chuẩn ngày × giờ tăng ca × hệ số)
Gross           = Lương công + công việc + đóng thùng + tăng ca
                  + phụ cấp ADD_GROSS + phát sinh tăng
Net             = Gross − bảo hiểm − thuế − phát sinh giảm
```

Mẫu cũ: divisor=1000; đóng thùng ZIN=400, HOP=1200; OT dùng 8 giờ và hệ số 1,5/2/3, ngoại lệ thủ kho ID 28 dùng hệ số 1. Map ngoại lệ sang hồ sơ/cấu hình, không hardcode ID cũ.

Gom tiền công theo nhân viên/việc/loại ca/phiên đơn giá trước làm tròn. Giá thay giữa tháng tách nhóm. Sửa lỗi cũ cộng double vào int phải có báo cáo chênh lệch.

### 8.3. Văn phòng

```text
Căn cứ ngày = base_salary + salary_fund + phụ cấp IN_DAY_BASE
Lương công = Căn cứ ngày / D × C
Căn cứ OT  = insurance_base / D               [theo luồng gọi cũ]
Tiền OT    = SUM(Căn cứ OT / giờ chuẩn ngày × giờ × hệ số)
Gross      = Lương công + OT + phụ cấp ADD_GROSS + phát sinh tăng
Net        = Gross − bảo hiểm − thuế − phát sinh giảm
```

Ăn/đồng phục/điện thoại nằm trong căn cứ ngày theo code cũ; không cộng lại vào gross. Policy khấu trừ riêng theo nhóm, không mặc định kho và văn phòng giống nhau.

### 8.4. Chính sách và breakdown

Bộ quy tắc có phiên bản đi kèm mã nguồn chứa căn cứ/tỷ lệ/bậc khấu trừ, miễn/giảm, hệ số OT, giờ chuẩn và chế độ làm tròn. Chính sách theo kỳ phải được kế toán duyệt; tài liệu không xác nhận mức pháp lý hiện hành. MVP hỗ trợ nghỉ hưởng lương 0 hoặc 100%.

BigDecimal cho trung gian, lưu raw_amount và rounded_amount từng khoản trong breakdown_json. Chốt rounding_mode/scale/stage sau đối chiếu `NumberUtils` cũ. Tổng gross/net cộng từ khoản đã làm tròn để phiếu in khớp.

Mỗi dòng breakdown chứa code, quantity, base, rate, coefficient, divisor, raw/rounded_amount, formula_version và source IDs/versions. Thiếu hồ sơ/rate/policy, D/divisor=0, giờ mâu thuẫn hoặc đổi nhóm giữa kỳ chưa hỗ trợ đều báo lỗi, không tự coi bằng 0.

## 9. Transaction, tính lại và chốt

Không có hàng khóa toàn module. Dùng các hàng đã tồn tại: employees để tuần tự hóa sửa hồ sơ/nghỉ/công của một người; employee_payroll_periods để điều phối đầu vào và chốt; employee_leave_balances để bảo vệ số dư phép.

Thứ tự khóa cho mọi thao tác ghi/import: employees theo ID tăng dần → kỳ lương theo period/group → số dư phép theo employee/type/year. Tạo kỳ/số dư còn thiếu bằng unique key và upsert, sau đó SELECT FOR UPDATE; tuyệt đối không chỉ kiểm tra trong bộ nhớ. Sửa phiếu phải khóa cả người/kỳ cũ và mới. Các thao tác chỉ cần khóa kỳ không được quay lại lấy khóa nhân viên sau đó. Submit/duyệt nghỉ luôn khóa hàng employees kể cả khi chưa có balance, nên kiểm tra trùng buổi có hiệu lực khi duyệt đồng thời.

Ghi đầu vào tăng revision kỳ liên quan, làm STALE run nháp. Thay đổi danh mục/đơn giá phải khóa các kỳ hiện hữu chịu ảnh hưởng và cập nhật revision; bảo vệ khoảng hiệu lực bằng khóa hàng loại việc cha. Khi capture/chốt, đọc lại nguồn và kiểm tra fingerprint cả đơn giá để phát hiện thay đổi hoặc bản ghi thêm mới. Chặn sửa hồi tố ảnh hưởng kỳ LOCKED. Thay phiên bản quy tắc qua triển khai làm run nháp cũ không còn hợp lệ nếu chưa tính lại với phiên bản được chọn; run đã chốt tiếp tục dùng snapshot cũ.

Chạy lương:

1. Transaction khóa kỳ, kiểm tra OPEN, capture bộ quy tắc và đầu vào từng người; tạo CALCULATING và các bản ghi `employee_payroll_results` chứa snapshot, commit.
2. Tính ngoài transaction khóa; mỗi người ghi breakdown/số tổng hoặc errors. Kết quả chỉ hiện là khả dụng khi cả run hoàn tất.
3. Transaction khóa kỳ, so input_revision, phiên bản quy tắc và fingerprint đầu vào/đơn giá. Không đổi và không lỗi → READY; đã đổi → STALE; lỗi → FAILED. Retry cùng request_key trả lại run, không tạo bản trùng.
4. Chốt khóa kỳ, kiểm tra READY, revision/phiên quy tắc/fingerprint khớp và không có lỗi, set FINALIZED/current_run_id và LOCKED cùng transaction.
5. Mở kỳ có quyền và lý do, giữ kết quả cũ. Khi chốt phiên thay thế, run cũ SUPERSEDED; không xóa lịch sử. Điều chỉnh kỳ sau có origin_result_id và chỉ nhập chênh lệch.

Không cần bảng job riêng ở MVP: chạy đồng bộ theo request, run được lưu trước tính. Nếu xử lý bị ngắt, run CALCULATING quá hạn được đánh FAILED có audit và tạo phiên mới; không phục hồi bằng cộng kết quả cũ. Chỉ bổ sung job bền vững khi dữ liệu thực tế đòi hỏi.

## 10. API và giao diện

API `/employee-payroll`, paging/ApiResponse theo convention repo, body ghi có expected_version; create/batch có request_key hoặc event_key được lưu để chống lặp. Quyền kiểm tra tại backend, không chỉ ẩn nút.

| Nhóm API | Thao tác |
| --- | --- |
| `/work-types`, `/work-rates` | List/get/create/update; cấu hình có hiệu lực và validate kind |
| `/work-sheets?kind=WORK/CRATING` | List/create/update; `/{id}/confirm`, `/{id}/cancel` |
| `/work-sheet-items?kind=...&employee_id=...` | Chi tiết phân bổ chỉ đọc |
| `/attendance`, `/time-records`, `/pay-items` | Xem bảng công, ghi giờ/điều chỉnh/phát sinh; confirm/cancel |
| `/leave-requests` | List/create/update nháp; submit/approve/reject/cancel; batch nhiều ngày |
| `/leave-balances`, `/leave-balances/{id}/transactions` | Ngày phép và lịch sử |
| `/leave-grants/preview`, `/leave-grants/apply`, `/leave-adjustments`, `/leave-transfers` | Cấp/điều chỉnh/chuyển phép, event key ổn định |
| `/periods`, `/runs` | Xem kỳ, tạo run theo kỳ/nhóm |
| `/runs/{id}`, `/runs/{id}/results`, `/runs/{id}/results/{employeeId}` | Trạng thái, bảng và chi tiết/errors |
| `/runs/{id}/finalize`, `/periods/{id}/reopen` | Chốt/mở kỳ |
| `/reports/year`, `/runs/{id}/export`, `/runs/{id}/payslips/{employeeId}` | Tổng hợp năm, xuất/in theo phiên |

List/detail dùng GET, create và hành động dùng POST, update dùng PUT /{id}. Các API danh mục không cho sửa trạng thái nghiệp vụ trực tiếp. Mã lỗi: PERIOD_LOCKED, VERSION_CONFLICT, LEAVE_OVERLAP, INSUFFICIENT_LEAVE, MISSING_RATE, AMBIGUOUS_OVERTIME, STALE_RUN, INVALID_PROFILE.

Hồ sơ nhân viên tiếp tục dùng `/employees` và feature `employee` hiện có, bổ sung filter payroll_group và các trường mới; không tạo CRUD `/employee-payroll/profiles`. DTO create/update chỉ cập nhật các trường được phép theo nhóm/quyền, trường không gửi giữ nguyên để UI cũ không xóa dữ liệu mới. Mọi đường sửa employees có ảnh hưởng lương kho/văn phòng (kể cả import và API cũ) phải gọi hook kiểm tra kỳ/version, audit và làm STALE theo mục 9. Thay đổi nhân viên SALE kiểm thử hồi quy theo hành vi hiện tại.

UI giữ tên quen thuộc: Nhân viên; Công việc; Chi tiết công việc; Đóng thùng; Chi tiết đóng thùng; Chấm công; Tăng ca; Phát sinh; Nghỉ phép; Ngày phép; Cấu hình; Lương kho; Lương văn phòng. Hai màn hình công việc/đóng thùng gọi chung API với kind khác nhau. Không cần màn hình cho từng bảng DB.

Query key bắt đầu employee-payroll. Decimal giữ dạng chuỗi, backend là nguồn tính chính thức. Phiếu in dùng snapshot run; bảng năm chỉ cộng current_run FINALIZED của mỗi kỳ, không đếm phiên cũ. Quyền tách view/manage/confirm cho đầu vào; leave.approve, leave.balance.manage, workrate.manage, payroll.view/calculate/finalize/reopen/export. Scope nhân viên/nhóm áp dụng cả list/detail/export.

## 11. Migration và truy vết

Công cụ chuyển dữ liệu theo đợt: export chỉ đọc → manifest/staging file → preview → apply → báo cáo đối soát. Chưa cần CRUD quản lý import trong ứng dụng. Manifest gồm batch ID, checksum, thống kê và mapping nhân viên đã xác nhận; log apply vào audit. Mọi apply dùng khóa và kiểm tra kỳ như nhập tay.

| Nguồn entity cũ | Đích |
| --- | --- |
| Staff nhóm 2/3 | Map sang employees, phân WAREHOUSE/OFFICE, dùng cột sẵn có và cột bổ sung; nếu trùng nhân viên SALE thì báo xung đột để xử lý, không ghi đè |
| WhJobCategory/WhJob và loại đóng thùng | `employee_work_types` theo kind/parent |
| WhCoefficient, giá đóng thùng | `employee_work_rates` |
| WhQuantity/WhCrating | `employee_work_sheets` theo kind; legacy_source phân biệt nguồn |
| WhQuantityDetail/WhCratingDetail | `employee_work_sheet_items`, giữ số gốc để đối chiếu |
| OnLeaveType | Danh mục leave-types có phiên bản đi kèm mã nguồn; giữ mã/nhãn cũ qua mapping |
| OnLeave | `employee_leave_balances` + `employee_leave_transactions` OPENING/GRANT theo phương án cutover |
| OnLeaveRegister | `employee_leave_requests`; trạng thái và ảnh hưởng số dư theo quy tắc import |
| Overtime/ConstsIncurred | `employee_time_records` / `employee_pay_items` |
| WhSalary/SalaryOffice | `employee_payroll_runs` origin LEGACY + `employee_payroll_results`, không tính lại |

Tên bảng nguồn phải xác nhận qua DAO/schema thật trước script. Không assume trùng ID nhân viên. Import nhiều lần cùng legacy_source/id không nhân bản; source_hash đổi thì preview lại, không ghi đè dữ liệu đã vận hành.

Nếu dùng OPENING, nghỉ trước ngày chuyển chỉ tra cứu và không ghi USE lần nữa; lưu cờ `affects_balance=false` trên bản legacy để phân biệt. Snapshot lịch sử chỉ chứa dữ liệu có thật, đánh dấu thiếu chi tiết thay vì dựng giả. Import rollback chỉ được đảo batch chưa có tham chiếu vận hành, có audit; sau vận hành dùng điều chỉnh.

## 12. Kiểm thử trọng tâm

- Chia 1.000 cho 3 bảo toàn lượng; sửa/hủy phiếu không để chi tiết trùng; kind WORK/CRATING không chọn nhầm rate.
- Công tháng ngắn, lễ trùng Chủ nhật, vào/nghỉ giữa tháng; D=0 báo lỗi.
- Một người nhiều việc một ca không nhân giờ; giờ mâu thuẫn cần xác nhận.
- Nghỉ 0,5/1 ngày, qua năm, có lương không trừ quỹ; hai đơn chồng buổi bị chặn trong giao dịch đồng thời.
- Retry duyệt/hủy/cấp/chuyển phép không trừ/cộng hai lần; balance bằng tổng giao dịch phép.
- Hủy phép năm đã chuyển/hết hạn bị chặn đến khi có điều chỉnh được xác nhận.
- Sửa nguồn trong lúc tính → STALE; chốt và sửa đồng thời không cùng vượt kiểm tra kỳ; FAILED không được chốt.
- Bộ quy tắc/ngày lễ thiếu hoặc thay phiên bản phải chặn tính/chốt không hợp lệ; snapshot/breakdown JSON validate schema và hiển thị lại đúng dù cấu hình đã đổi; tổng phiếu in = tổng cột kết quả.
- Run mới không xóa phiên cũ; bảng năm không cộng hai lần; dữ liệu/kết quả sale không thay đổi khi chạy kho/văn phòng.
- Thêm nhân viên WAREHOUSE/OFFICE có basic_salary/insurance_base > 0 không làm họ xuất hiện trong luồng sale; backfill nhóm không loại nhầm nhân viên sale hiện hữu.
- Update employees qua API/UI cũ không xóa cột mới; sửa lương tạo STALE đúng kỳ; kỳ cũ dùng snapshot, không dùng lương mới. Kiểm tra quyền sửa trường lương và optimistic locking.
- Import hai lần không trùng; opening không trừ lịch sử; thiếu mapping báo lỗi.
- API trực tiếp không có quyền/sai scope bị chặn, bao gồm xuất bảng lương và xem audit lương.
- Sửa/chốt/hủy/import có log đúng actor, nguồn, before/after và liên kết run; không log trùng; import không có HTTP context vẫn ghi log. Kiểm thử transaction rollback không để lại log SUCCESS sai.

Unit test calculator bằng fixture kế toán duyệt; integration test transaction/khóa/unique trên DB cùng loại production; build/typecheck frontend và kiểm tra luồng nhập → nghỉ/phát sinh → tính → chốt → in. Đây là kế hoạch kiểm thử khi triển khai, chưa phải kết quả đã chạy.

## 13. Lộ trình

1. Chốt fixture và mapping; mở rộng employees và tạo 12 bảng; cập nhật API/UI nhân viên, phân nhóm và lọc đối tượng sale trước khi nhập dữ liệu kho/văn phòng.
2. Hồ sơ/đơn giá, công việc/đóng thùng và chi tiết, calculator kho để demo.
3. Nghỉ phép/ngày phép, công/tăng ca/phát sinh, snapshot/khóa kỳ và in/xuất; hoàn thiện kho.
4. Calculator/giao diện văn phòng, đối chiếu từng khoản.
5. Chuyển lịch sử/số dư, nghiệm thu và cutover theo BA. Chỉ ngừng dịch vụ cũ sau kiểm kê phụ thuộc và thử khôi phục backup.

Cần chốt trước vận hành: đơn vị/divisor, đơn giá, làm tròn, ngoại lệ thủ kho, giờ ca trùng, chính sách khấu trừ, định mức/quy tắc chuyển phép và kỳ mẫu. Tách các mở rộng như tích lũy phép tự động hoặc hạn theo từng đợt cấp ra khỏi MVP; không thiết kế bảng sẵn cho nhu cầu chưa được xác nhận.
