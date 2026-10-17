# Phân tích chuyển lương kho, văn phòng, nghỉ phép và ngày phép

Ngày cập nhật: 17/10/2026. Trạng thái: phân tích và đề xuất, chưa triển khai.

Thiết kế kỹ thuật: [TDD lương kho và văn phòng](TDD_LUONG_KHO_VAN_PHONG.md).

## 1. Mục tiêu và phạm vi đã thống nhất

Chuyển chức năng từ trang Kế Toán VLife cũ sang hệ thống mới, ưu tiên lương kho để kế toán trao đổi với Nghĩa, sau đó hoàn thiện lương văn phòng. Chuyển đầy đủ nghỉ phép và ngày phép để vận hành độc lập, tiến tới ngừng gia hạn trang cũ.

Phạm vi đợt này là chuyển nghiệp vụ theo chức năng cũ. Theo trao đổi mới nhất, bỏ phương án tích hợp sổ chi tiết kho khỏi đợt triển khai này. Sản lượng tính lương được ghi nhận ở Công việc và Đóng thùng như luồng cũ.

| Nhóm | Chức năng chuyển |
| --- | --- |
| Nhân viên | Hồ sơ và thiết lập lương kho/văn phòng |
| Nghỉ phép | Ghi nhận nghỉ theo ngày/buổi, loại nghỉ, hưởng lương |
| Ngày phép | Cấp phép theo nhân viên/năm/loại, đã dùng, còn lại, lịch sử |
| Công và phát sinh | Công, tăng ca, thu nhập/khấu trừ phát sinh |
| Chấm công kho | Công việc, Chi tiết công việc, Đóng thùng, Chi tiết đóng thùng |
| Lương kho | Bảng tháng/năm, tính lại, chi tiết từng người, in/xuất |
| Lương văn phòng | Bảng tháng/năm, tính lại, chi tiết từng người, in/xuất |
| Dữ liệu cũ | Danh mục, đầu vào, số dư phép, kết quả lương lịch sử |

Lương kho/văn phòng có module, API, cấu hình và kết quả riêng. Các màn hình `salary-setup`, `payroll-config`, `payroll-run`, `payroll-result`, `salary-adjustment` và pipeline hiện tại thuộc lương sale, không được dùng để chạy hai nhóm mới.

## 2. Căn cứ và giới hạn

- Nguồn cũ: `/Users/huynguyen/Documents/projects/vlife`, gồm `vlife-cms`, `VLifeAPI`, `VLifeEngine`.
- Hệ thống đích: `/Users/huynguyen/Documents/vlife/git`, gồm `vl-cms`, `vl-api`, `vl-shared`.
- Đã đọc code, chưa đối soát database production hoặc chạy bảng lương mẫu.
- Thông số tìm thấy trong code phản ánh cách tính cũ; kế toán cần duyệt chính sách cho kỳ vận hành mới. Tài liệu không xác nhận hiệu lực pháp lý của các thông số cũ.
- Các cải tiến như duyệt nghỉ, khóa kỳ, nhật ký và cấu hình ngoại lệ được ghi là đề xuất; không xem là chức năng đã có ở trang cũ.

## 3. Hiện trạng hệ thống cũ đã xác minh

### 3.1. Lương kho

Luồng chính nằm ở `VLifeEngine/src/com/vlife/models/warehouse/WHSalaryModel.java`.

| Nội dung | Hành vi trong mã cũ |
| --- | --- |
| Đối tượng | Nhóm `salary_group_id = 2`, nhân viên có `status = 1` |
| Công chuẩn tháng | Số ngày tháng trừ Chủ nhật và ngày lễ trong tuần |
| Công tính lương | Công thực tế sau khi trừ nghỉ + ngày nghỉ có lương; có xét ngày vào làm |
| Lương ngày | `staff.salary / công chuẩn tháng` |
| Lương công | Lương ngày × công tính lương |
| Lương công việc | Tổng hợp từ `WhQuantityDetail` theo nhân viên, nhóm việc, công việc và loại ca |
| Lương đóng thùng | Tổng hợp từ `WhCratingDetail` theo nhân viên và loại đóng thùng |
| Tăng ca | Lấy từ chi tiết công việc kho; phân loại trong tuần, Chủ nhật, lễ |
| Phát sinh | Thu nhập/giảm trừ từ `ConstsIncurred`; có phụ cấp khác từ hồ sơ |
| Tổng thu nhập | Lương công + tăng ca + công việc + đóng thùng + thu nhập phát sinh + phụ cấp nhà trọ/lạm phát |
| Thực nhận | Tổng thu nhập trừ bảo hiểm, thuế và giảm trừ phát sinh |

Quy tắc sản lượng và phân chia đang có:

1. `WhQuantityCreateRequest` tạo chi tiết cho từng nhân viên được chọn, chia đều `amount / số nhân viên`. Giờ làm `amount_hour` được gán cho từng người, không chia theo số người.
2. `WhCratingCreateRequest` cũng chia đều số lượng đóng thùng cho danh sách nhân viên.
3. `WHQuantityUtils` tính tiền công bằng `sản lượng × đơn giá × hệ số / 1000`, sau đó gọi hàm cắt phần lẻ. Đơn giá/hệ số lấy từ `WhCoefficient`, phân theo nhóm việc, công việc và loại ca.
4. Nhóm việc có ID `3` được xử lý riêng theo từng công việc, có chú thích là trộn hàng. Chưa xác minh tên và danh mục thực tế trong database.
5. `WHCratingUtils` dùng đơn giá viết trực tiếp trong code: loại `ZIN` là 400, loại `HOP` là 1.200 trên mỗi đơn vị `amount`. Cần xác nhận cách gọi loại thùng và đơn vị đo thực tế.
6. Tăng ca thông thường dùng lương ngày / 8 nhân giờ và hệ số 1,5 / 2 / 3. Riêng `staff.id == 28` dùng hệ số 1 cho cả ba loại theo nhánh dành cho thủ kho.

Giao diện cũ có bảng theo tháng, danh sách theo năm, chi tiết nhân viên, bản in bảng lương và phiếu lương. API nằm dưới `/salary/warehouse`.

### 3.2. Lương văn phòng

Luồng chính nằm ở `VLifeEngine/src/com/vlife/models/salary/SalaryOfficeModel.java`, chọn nhóm `salary_group_id = 3` và chỉ xử lý nhân viên đang hoạt động.

- Căn cứ tính lương ngày gồm lương cơ bản + lương quỹ + tiền ăn + đồng phục + điện thoại theo `SalaryOfficeUtils.getSalaryTotal`.
- Công tính lương được tính từ lịch tháng, ngày vào làm và nghỉ phép tương tự kho.
- Tăng ca lấy từ `Overtime`; luồng thực tế dùng `WHSalaryUtils.getSalaryOvertime`, với căn cứ lương ngày tính từ `staff.salary_insurance`.
- Thu nhập phát sinh và giảm trừ lấy từ `ConstsIncurred`; phụ cấp khác được cộng vào thu nhập phát sinh.
- Có tính bảo hiểm, thuế, lương thực nhận và lưu snapshot hồ sơ nhân viên.
- API dưới `/salary/office`; có bảng tháng/năm, chi tiết và bản in.

Lưu ý khi chuyển: trong `SalaryOfficeUtils` cũng có một hàm tăng ca với biểu thức khác, nhưng model đang gọi hàm ở `WHSalaryUtils`. Phải đối chiếu luồng gọi thực tế khi tái hiện kết quả.

### 3.3. Những điểm không nên sao chép nguyên trạng

| Điểm trong code cũ | Hướng xử lý khi chuyển |
| --- | --- |
| Đơn giá đóng thùng, ID thủ kho và ID nhóm trộn được viết cố định | Chuyển thành cấu hình theo mã nghiệp vụ, nhân viên/vai trò và ngày hiệu lực |
| `WHQuantityUtils.getTotalAmount` cộng `amount` kiểu double vào biến int | Dùng số thập phân và quy tắc làm tròn được duyệt; đối chiếu chênh lệch do mất phần lẻ |
| Giờ tăng ca nằm trên từng dòng công việc | Tránh cộng trùng giờ nếu một người làm nhiều việc trong cùng ca |
| Phân loại tiền công và tăng ca có điều kiện lễ/Chủ nhật khác nhau | Chốt thứ tự ưu tiên ngày lễ, Chủ nhật, ca; có ví dụ nghiệm thu |
| Chạy lại kỳ xóa bảng kết quả kỳ rồi tạo lại | Tạo phiên tính, lưu lịch sử và khóa kỳ; không ghi đè bảng đã duyệt |
| `WHSalaryModel` đọc snapshot cũ qua `SalaryDao` thay vì `WhSalaryDao` | Rà soát nguồn snapshot; không dùng nhánh này làm chuẩn lịch sử |
| Kiểm tra kỳ cũ bằng hiệu số tháng, không xét đầy đủ năm | Dùng kỳ `YYYY-MM` và snapshot theo phiên |
| Hàm công tháng khi mới vào làm lặp đến ngày 31 | Kiểm tra tháng ngắn, ngày lễ trùng Chủ nhật và ngày vào/nghỉ việc |

Các điểm trên là rủi ro quan sát từ mã nguồn, chưa phải kết luận về sai lệch trên dữ liệu đang vận hành.

### 3.4. Nghỉ phép và ngày phép trong mã cũ

- `OnLeave` lưu số ngày được cấp (`total`) theo nhân viên, loại phép và năm. Giao diện `onleave.ts` có tổng phép, đã đăng ký (`register`), còn lại (`remain`) và chi tiết lịch sử; API dưới `/onleave`.
- `OnLeaveRegister` lưu lần nghỉ theo ngày, nhân viên, loại phép, buổi sáng/chiều và cờ `paid` (hưởng lương). API dưới `/onleave/register` có tạo, sửa, xóa, danh sách và chi tiết.
- `LeaveUtils` tính mỗi buổi là 0,5 ngày; cả sáng và chiều là 1 ngày. Ngày nghỉ hưởng lương chỉ được cộng khi `paid = true`.
- Hai model lương kho/văn phòng đọc đăng ký nghỉ trong tháng để tính công. Trong entity đăng ký đã khảo sát chưa có trạng thái phê duyệt; luồng duyệt ở phần đề xuất bên dưới là chức năng mới, không khẳng định trang cũ đã có.
- Phải phân biệt **nghỉ hưởng lương** với **nghỉ trừ quỹ phép năm**. Code có cờ hưởng lương và loại phép nhưng chưa đủ để kết luận mọi nghỉ hưởng lương đều phải trừ phép năm.

## 4. Luồng chức năng khi chuyển

### 4.1. Công việc và chi tiết công việc

Người phụ trách nhập ngày làm, ca, nhóm việc, công việc, sản lượng, giờ làm, danh sách nhân viên và ghi chú. Khi lưu, hệ thống chia đều sản lượng cho những người được chọn như cách cũ; chi tiết nhân viên được sinh tự động. Giờ làm của mỗi người không chia theo số nhân viên.

Chi tiết công việc hỗ trợ lọc tháng/nhân viên/nhóm việc/ca, tổng hợp sản lượng và tiền công. Sửa phiếu gốc phải cập nhật toàn bộ chi tiết trong cùng giao dịch; không sửa độc lập chi tiết đã sinh. Phiếu đã dùng trong kỳ chốt chỉ được xử lý qua quy trình mở kỳ hoặc điều chỉnh.

### 4.2. Đóng thùng và chi tiết đóng thùng

Nhập ngày, loại đóng thùng, số lượng, nhân viên và ghi chú. Chia đều số lượng cho từng người, áp dụng đơn giá theo loại và ngày hiệu lực. Giữ hai màn hình phiếu tổng và chi tiết theo người như cũ.

### 4.3. Công, tăng ca và phát sinh

Công tính lương giữ cơ sở lịch làm việc, ngày vào/nghỉ việc và nghỉ phép như model cũ; bảng chấm công giúp kiểm tra các ngày/buổi này. Chưa tự chuyển sang yêu cầu nhập đủ công từng ngày nếu chưa chốt nghiệp vụ.

Tăng ca kho lấy ngữ cảnh ca từ công việc nhưng phải có bản ghi giờ theo người/ngày/ca để tránh cộng trùng khi một ca có nhiều công việc. Nếu các dòng ghi giờ khác nhau, hiển thị lỗi để người phụ trách xác nhận. Tăng ca văn phòng được nhập riêng theo chức năng cũ. Thu nhập và giảm trừ phát sinh lưu theo người, ngày/kỳ, loại khoản và lý do.

### 4.4. Nghỉ phép — quản lý từng lần nghỉ

Phạm vi áp dụng: nhân viên kho và văn phòng; dữ liệu độc lập với nghiệp vụ lương sale.

- Màn hình danh sách nghỉ có lọc nhân viên, bộ phận, ngày/kỳ, loại nghỉ và trạng thái.
- Tạo đơn/ghi nhận nghỉ: nhân viên, ngày hoặc khoảng ngày, buổi sáng/chiều/cả ngày, loại nghỉ, lý do, hưởng lương hay không và có trừ quỹ phép hay không theo chính sách loại nghỉ.
- Khoảng nghỉ nhiều ngày được tách thành chi tiết từng ngày để xét lịch làm việc, ngày lễ, qua tháng và qua năm. MVP hỗ trợ đơn vị 0,5 ngày như cũ; nghỉ theo giờ chỉ bổ sung khi có yêu cầu.
- Đề xuất trạng thái: nháp → chờ duyệt → đã duyệt/từ chối; đơn đã duyệt có thao tác hủy theo quyền. Chốt với kế toán việc nhân viên tự đăng ký hay người phụ trách nhập hộ.
- Kiểm tra trùng buổi nghỉ, thời gian ngoài thời gian làm việc và số dư phép; quy tắc cho phép âm/vượt phép phải cấu hình và được duyệt.
- Chỉ nghỉ đã duyệt được đưa vào công/lương và trừ quỹ phép. Đơn chờ duyệt hiển thị riêng, không tự trừ số dư chính thức.
- Sửa đơn đã duyệt phải duyệt lại phần thay đổi; hủy phải hoàn phép đúng một lần và giữ lịch sử, không xóa dấu vết.
- Nghỉ và công/ca cùng thời gian phải được đối soát. Đối chiếu nhân viên được phân công trên phiếu công việc với thời gian nghỉ.
- Quyền đề xuất: người nhập tạo/sửa nháp; quản lý duyệt nghỉ; nhân sự/kế toán quản lý chính sách và điều chỉnh. Phân quyền cụ thể chốt với người vận hành.

### 4.5. Ngày phép — quản lý quỹ và số dư

Màn hình **Ngày phép** theo năm, nhân viên và loại phép, hiển thị: số dư đầu kỳ, phép được cấp, phép chuyển năm trước, điều chỉnh, phép đã dùng, phép hết hạn, đơn đang chờ duyệt và số còn lại. Chi tiết nhân viên truy ra từng giao dịch cấp/trừ/hoàn/điều chỉnh.

Công thức đề xuất:

`Phép còn lại = số dư đầu kỳ + cấp mới + chuyển vào + điều chỉnh tăng - điều chỉnh giảm - đã dùng + hoàn phép - hết hạn`

- Số dư đầu kỳ phải quy định rõ đã bao gồm phép chuyển năm trước hay chưa; không cộng hai lần.
- MVP cấp phép theo năm và điều chỉnh được xác nhận, có xét ngày vào làm theo chính sách đã chốt. Tích lũy tự động theo tháng là mở rộng khi có yêu cầu; không tự gán một định mức chung.
- Chính sách loại nghỉ có hai thuộc tính độc lập: tỷ lệ hưởng lương và có trừ quỹ phép. Ví dụ nghỉ hưởng lương không trừ phép và nghỉ phép năm cần được biểu diễn riêng.
- MVP chuyển năm/hết hạn bằng thao tác có xem trước và xác nhận của người phụ trách. Chưa quản lý lô phép, hạn riêng từng lần cấp hoặc thứ tự trừ theo lô; nếu cần nghiệp vụ đó phải bổ sung thiết kế trước triển khai. Hủy nghỉ của quỹ đã chuyển/hết hạn phải đối soát trước khi hoàn.
- Cấp hoặc điều chỉnh hàng loạt có xem trước, lý do, người duyệt và nhật ký. Chạy lại tác vụ cấp/chuyển phép không được cộng thêm lần nữa.
- Ghi sổ biến động phép theo giao dịch và liên kết đơn nghỉ; kiểm soát đồng thời khi duyệt nhiều đơn để không vượt số dư ngoài chính sách.

### 4.6. Liên kết nghỉ phép với lương và khóa kỳ

- Tách rõ tổng ngày nghỉ, ngày nghỉ hưởng lương, ngày nghỉ không lương và ngày trừ quỹ phép trong bảng công.
- Với chính sách tương đương mã cũ: công tính lương = công theo lịch trong thời gian làm việc − ngày nghỉ thuộc lịch làm việc + ngày nghỉ hưởng lương. Loại nghỉ hưởng một phần lương, nếu có, cần công thức riêng được duyệt.
- Ngày lễ/ngày không làm việc trong khoảng nghỉ không tự động trừ phép hoặc trừ công; dựa trên lịch áp dụng cho nhân viên.
- Nghỉ có lương không tự sinh tiền sản lượng/đóng thùng; các khoản này vẫn dựa trên công việc thực tế đã duyệt.
- Phiên tính lương lưu snapshot đơn nghỉ và chính sách đã dùng. Kỳ đã chốt không thay đổi khi sửa đơn nghỉ; xử lý qua điều chỉnh hoặc mở kỳ có quyền.
- Việc hoàn/trừ lại quỹ phép do sửa đơn và việc điều chỉnh tiền lương phải có liên kết để đối soát, tránh hoàn phép nhưng bỏ sót điều chỉnh công.

### 4.7. Bảng lương và lịch sử

Bảng lương kho có lương công, công việc, đóng thùng, tăng ca, phụ cấp, phát sinh, bảo hiểm, thuế và thực nhận. Văn phòng có các khoản tương ứng theo công thức riêng, không có tiền công việc/đóng thùng.

Người dùng chọn kỳ, tính thử, xem lỗi và chi tiết từng khoản, đối chiếu rồi chốt. Tiền công truy ngược được phiếu công việc/đóng thùng và chi tiết nhân viên. Lưu snapshot đầu vào/cấu hình để kết quả đã chốt không đổi khi hồ sơ hoặc đơn giá thay đổi.

## 5. Hướng triển khai

- Tạo nhóm menu **Lương kho và văn phòng** với các chức năng ở mục 1; giữ cách gọi Công việc, Chi tiết công việc, Đóng thùng, Chi tiết đóng thùng để người dùng nhận biết.
- Backend và bảng dữ liệu mới có namespace riêng. Không gọi service hoặc ghi bảng kết quả lương sale.
- Liên kết định danh nhân viên qua mapping; thiết lập lương, phụ cấp, thuế/bảo hiểm và lịch sử kho/văn phòng lưu riêng.
- Hai bộ tính kho/văn phòng dùng chung nền tảng công/phép và lưu phiên trong module mới, mỗi bộ có công thức riêng.
- Thiết kế rút gọn 15 bảng trong TDD: 4 bảng hồ sơ/cấu hình, 4 bảng đầu vào công/lương, 3 bảng nghỉ/ngày phép, 4 bảng kỳ/kết quả/nhật ký. Công việc và đóng thùng dùng chung bảng có loại, giao diện vẫn tách như cũ.
- Dùng số thập phân, cấu hình đơn giá có hiệu lực, nhật ký và kiểm soát sửa đồng thời. Snapshot và breakdown lưu cùng kết quả nhân viên; không tạo bảng riêng cho từng lớp dữ liệu phụ.
- Các điểm khác mã cũ như làm tròn, chống trùng giờ tăng ca hoặc ngoại lệ thủ kho cần có mẫu đối chiếu và được ghi nhận trước vận hành.

## 6. Lộ trình và bàn giao

| Giai đoạn | Công việc | Kết quả |
| --- | --- | --- |
| 0 | Chọn kỳ mẫu; chốt danh mục, đơn vị, đơn giá và mapping nhân viên | Bộ dữ liệu và kết quả đối chiếu |
| 1 | Hồ sơ, công việc/chi tiết, đóng thùng/chi tiết, đơn giá; bảng lương kho thử | Kế toán xem và trao đổi với Nghĩa |
| 2 | Nghỉ phép, ngày phép, công/tăng ca, phát sinh, khóa kỳ, in/xuất | Lương kho đầy đủ, vận hành độc lập |
| 3 | Công thức và giao diện lương văn phòng | Bảng văn phòng đối chiếu đạt |
| 4 | Chuyển lịch sử, số dư phép, chạy đối chiếu và chuyển chính thức | Có thể ngừng trang cũ khi đủ điều kiện |

## 7. Chuyển dữ liệu và ngừng trang cũ

Chuyển nhân viên, cấu hình, công việc/đóng thùng cùng chi tiết, nghỉ phép, ngày phép, tăng ca, phát sinh và bảng lương lịch sử. Lưu mapping ID cũ/mới; không dựa vào việc trùng ID giữa hai hệ thống.

Chọn số dư phép mở đầu tại ngày chuyển hoặc dựng lại từ lịch sử đầy đủ. Nếu nhập số dư mở đầu, đơn nghỉ trước ngày chuyển chỉ tra cứu và không trừ phép lần nữa; đơn vắt qua ngày chuyển phải tách phần ảnh hưởng. Trạng thái nghỉ lịch sử phải theo quy tắc chuyển được xác nhận vì entity cũ không có trạng thái duyệt.

Kết quả lương lịch sử giữ nguyên số tiền gốc, không tính lại bằng quy tắc hiện tại. Chạy đối chiếu từng nhân viên và từng khoản trước khi chuyển chính thức.

Trước khi ngừng gia hạn: kiểm kê dịch vụ/ngày hết hạn và chức năng khác còn dùng, sao lưu database/mã nguồn/tệp đính kèm, thử khôi phục, chốt thời điểm ngừng nhập cũ, chuyển phần phát sinh cuối và xác nhận nghiệm thu. Có phương án quay lại và nguồn nhập liệu duy nhất nếu chuyển thất bại. Chỉ ngừng hệ thống khi dữ liệu tra cứu được và các nghiệp vụ còn lại đã có nơi vận hành.

## 8. Nội dung cần chốt trước vận hành

- Danh mục việc, đơn vị sản lượng, ý nghĩa phép chia 1.000, đơn giá đóng thùng và hệ số hiện dùng.
- Ngoại lệ thủ kho còn áp dụng hay không; phân loại tăng ca ngày lễ/Chủ nhật, cách xử lý trùng giờ.
- Mẫu lương kho/văn phòng được duyệt, quy tắc làm tròn và các khoản phụ cấp/thuế/bảo hiểm theo kỳ.
- Người nhập/duyệt nghỉ; nghỉ nào hưởng lương, nghỉ nào trừ quỹ; định mức cấp phép, chuyển năm, hết hạn và cho âm.
- Số dư phép tại ngày chuyển, kỳ lịch sử cần chuyển và ngày hết hạn dịch vụ cũ.

Các mục trên là đầu vào để cấu hình và nghiệm thu; không mở rộng sang tích hợp dữ liệu kho hoặc lương sale.

## 9. Tiêu chí nghiệm thu

- Có đủ màn hình trong phạm vi mục 1, lương kho hoàn thiện trước.
- Nhập công việc/đóng thùng một lần sinh đúng chi tiết từng người; sửa/hủy không để lại chi tiết trùng hoặc mất tổng sản lượng.
- Bảng tháng/năm, phiếu lương và bản xuất khớp số tiền phiên tính; từng khoản truy được đầu vào.
- So sánh từng khoản với kỳ cũ; sai khác do quy tắc sửa được giải thích và duyệt.
- Nghỉ nửa ngày/cả ngày, qua tháng/năm, ngày lễ và vào/nghỉ việc giữa tháng tính đúng; nghỉ hưởng lương và trừ quỹ được xử lý độc lập.
- Duyệt/hủy nghỉ và cấp/hoàn/chuyển phép không cộng trừ hai lần; số dư đầu + biến động = cuối đối chiếu được.
- Một ca nhiều việc không nhân giờ tăng ca; thiếu đơn giá/nhân viên/thiết lập phải báo lỗi trước chốt.
- Chạy lại lương không mất phiên cũ; sửa đầu vào làm phiên nháp hết hiệu lực; kỳ chốt chỉ đổi theo quy trình có quyền và nhật ký.
- Chạy lương kho/văn phòng không thay đổi dữ liệu hoặc kết quả lương sale.
- Chuyển số dư phép không trừ lại lịch sử, bảng lương lịch sử giữ nguyên, backup khôi phục được.

## 10. File mã cũ làm căn cứ

Các đường dẫn tương đối với thư mục mã cũ ở mục 2.

| Nguồn cũ | Vai trò |
| --- | --- |
| `VLifeEngine/src/com/vlife/models/warehouse/WHSalaryModel.java` | Luồng tính lương kho |
| `VLifeEngine/src/com/vlife/utils/warehouse/WHSalaryUtils.java` | Công, tăng ca, thu nhập kho |
| `VLifeEngine/src/com/vlife/utils/warehouse/WHQuantityUtils.java` | Tiền công theo sản lượng và hệ số |
| `VLifeEngine/src/com/vlife/utils/warehouse/WHCratingUtils.java` | Tiền đóng thùng |
| `VLifeAPI/src/com/vlife/api/modules/warehouse/request/quantity/WhQuantityCreateRequest.java` | Chia sản lượng công việc cho nhân viên |
| `VLifeAPI/src/com/vlife/api/modules/warehouse/request/crating/WhCratingCreateRequest.java` | Chia số lượng đóng thùng |
| `VLifeEngine/src/com/vlife/entities/staff/onleave/OnLeave.java`, `OnLeaveRegister.java`, `OnLeaveType.java` | Quỹ phép, đăng ký nghỉ và loại phép |
| `VLifeEngine/src/com/vlife/utils/LeaveUtils.java` | Quy đổi buổi nghỉ và nghỉ hưởng lương |
| `vlife-cms/src/modules/staff/service/onleave.ts`, `onleave-register.ts` | API và dữ liệu giao diện ngày phép/nghỉ phép |
| `VLifeEngine/src/com/vlife/models/salary/SalaryOfficeModel.java` | Luồng tính lương văn phòng |
| `VLifeEngine/src/com/vlife/utils/salary/SalaryOfficeUtils.java` | Thành phần lương văn phòng |
| `vlife-cms/src/modules/slary/service/warehouse.ts` và `office.ts` | Hợp đồng dữ liệu/API giao diện cũ |
| `vlife-cms/src/modules/slary/views/warehouse` và `office` | Bảng lương, chi tiết và bản in |

