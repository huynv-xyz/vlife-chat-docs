# Phân tích chuyển lương kho và văn phòng sang hệ thống VLife mới

Ngày lập: 16/10/2026. Trạng thái: đề xuất nghiệp vụ và kỹ thuật để trao đổi với kế toán/Nghĩa; chưa triển khai.

## 1. Hiểu yêu cầu

Chuyển chức năng tính lương kho và văn phòng từ trang Kế Toán VLife cũ sang trang mới để người dùng làm việc trên một hệ thống và có thể ngừng gia hạn hệ thống cũ. Ưu tiên lương kho để kế toán xem, đối chiếu và trao đổi với Nghĩa trước khi hoàn thiện.

**Phạm vi bổ sung đã xác nhận:** chuyển đầy đủ hai chức năng **Nghỉ phép** và **Ngày phép** cho nhân viên kho/văn phòng. Đây là phần bắt buộc của đợt chuyển, có màn hình và dữ liệu riêng, không chỉ là số ngày nhập vào bảng lương.

Thay đổi nghiệp vụ quan trọng là lấy sản lượng trực tiếp từ sổ chi tiết kho để tính tiền công, giảm việc nhập lại số lượng ở màn hình chấm công kho. Đây là chuyển cả dữ liệu, quy tắc tính và quy trình vận hành, không chỉ chuyển menu hoặc giao diện bảng lương.

Ảnh cung cấp thể hiện các chức năng liên quan: nhân viên, nghỉ phép, ngày phép, chi phí phát sinh, tăng ca, chấm công; công việc, chi tiết công việc, đóng thùng, chi tiết đóng thùng. Những đầu vào này cần được chuyển hoặc có chức năng thay thế để bảng lương hoạt động độc lập với trang cũ.

**Hướng đề xuất:** xây phần lương kho trên hệ thống mới, dùng sổ kho/chứng từ kho làm nguồn sản lượng, bổ sung phân công nhân viên và duyệt sản lượng tính công. Sau khi lương kho được nghiệm thu, hoàn thiện lương văn phòng, chuyển lịch sử và ngừng hệ thống cũ theo kế hoạch.

## 2. Phạm vi khảo sát và giới hạn

- Mã cũ: `/Users/huynguyen/Documents/projects/vlife`, gồm `vlife-cms`, `VLifeAPI`, `VLifeEngine`.
- Mã mới: `/Users/huynguyen/Documents/vlife/git`, tập trung `vl-cms`, `vl-api`, `vl-shared`.
- Giả định “trang mới” là hệ thống trong workspace trên; cần xác nhận môi trường triển khai khi bắt đầu thực hiện.
- Kết luận hiện trạng dưới đây dựa trên đọc mã nguồn, chưa chạy tính lương hoặc kiểm tra cơ sở dữ liệu production.
- Chưa có bảng lương mẫu đã duyệt, dữ liệu đơn giá thực tế, quy định chia công mới và ngày hết hạn dịch vụ cũ. Các thông số từ code cũ là bằng chứng về cách hệ thống cũ xử lý, chưa phải quy tắc được duyệt cho hệ thống mới.
- Tài liệu không xác nhận tính phù hợp pháp lý hiện hành của công thức thuế, bảo hiểm hoặc tăng ca cũ. Khi triển khai cần kế toán duyệt cấu hình áp dụng theo kỳ.

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

## 4. Hệ thống mới có gì và còn thiếu gì

### 4.1. Phân biệt module lương sale với nguồn dữ liệu kho

**Theo xác nhận của người phụ trách, `salary-setup`, `payroll-config`, `payroll-run`, `payroll-result`, `salary-adjustment` là các chức năng lương sale. Không đưa các màn hình, API, cấu hình hoặc bảng kết quả này vào phương án tái sử dụng cho lương kho/văn phòng.**

- `PayrollService` cùng các service trong `vl-shared/.../service/salary` thuộc luồng lương hiện có với scope và lương B2B. Đây không phải bộ máy tính lương chung đã được xác minh phù hợp với kho/văn phòng.
- Lương kho/văn phòng cần màn hình, API, cấu hình, phiên tính, kết quả và điều chỉnh riêng. Phần dùng chung giữa kho và văn phòng sẽ thiết kế từ nghiệp vụ của hai nhóm này.
- Hồ sơ định danh nhân viên và hạ tầng đăng nhập/phân quyền chỉ xem xét dùng chung sau khi kiểm tra mô hình dữ liệu. Các trường thiết lập lương sale không mặc nhiên áp dụng cho kho/văn phòng.
- Module sổ chi tiết kho ở `vl-cms/src/features/inventory/ledger` và chứng từ ở `inventory/voucher`.
- `InventoryLedger` có chứng từ/dòng chứng từ, ngày giờ hạch toán, sản phẩm, kho, lô, số lượng, đơn vị, loại chứng từ và tham chiếu.
- `InventoryVoucher` có loại nghiệp vụ, ngày chứng từ/ngày hạch toán, trạng thái, kho vật lý, nguồn chứng từ và liên kết sản xuất. `InventoryVoucherService` có ghi sổ, sửa chứng từ đã ghi sổ và bỏ ghi sổ.

### 4.2. Khoảng trống cần bổ sung

Entity `InventoryLedger` hiện chưa có người thực hiện, tổ/ca làm, công việc tính lương, tỷ lệ chia công hoặc đơn giá tiền công. Người lập/ghi sổ chứng từ không mặc nhiên là người làm việc hưởng lương.

Qua phạm vi rà soát chưa thấy bộ chức năng lương kho/văn phòng tương đương mã cũ, bao gồm phân công sản lượng, đóng thùng và chấm công/nghỉ phép tương ứng. Cần xác minh nguồn định danh nhân viên để liên kết đúng; các dữ liệu công, cấu hình lương và kết quả kho/văn phòng được thiết kế riêng với lương sale.

**Kết luận kỹ thuật:** có thể tự động lấy số lượng từ kho, nhưng chưa đủ dữ liệu để tự động tính lương theo nhân viên hoàn toàn. Cần một lớp ghi nhận công việc và phân bổ sản lượng nối giữa kho và bảng lương.

## 5. Luồng nghiệp vụ đề xuất cho lương kho

```mermaid
flowchart LR
    A[Chứng từ kho đã ghi sổ] --> B[Lọc nghiệp vụ được tính công]
    B --> C[Sản lượng và quy đổi đơn vị]
    C --> D[Gắn công việc, ca và người làm]
    D --> E[Duyệt phân bổ sản lượng]
    E --> F[Tính lương kho theo kỳ]
    G[Công, phép, tăng ca, phụ cấp] --> F
    F --> H[Đối chiếu và chốt lương]
    H --> I[Phiếu lương và lịch sử]
```

### 5.1. Xác định nguồn sản lượng

1. Chọn kỳ, kho vật lý và danh sách nghiệp vụ được phép tính công.
2. Lấy dòng chứng từ hợp lệ đã ghi sổ; liên kết về sổ kho để người dùng kiểm tra.
3. Ánh xạ loại nghiệp vụ/sản phẩm sang nhóm việc và đơn vị tính công theo cấu hình.
4. Hiển thị danh sách đủ điều kiện, thiếu cấu hình, thiếu người làm và bị loại kèm lý do.
5. Người phụ trách bổ sung ca, tổ và nhân viên; số lượng đã có trên kho không cần nhập lại.
6. Duyệt dữ liệu công trước khi chạy lương.

Không cộng toàn bộ số nhập/xuất thành sản lượng tính lương. Một quá trình sản xuất có thể sinh nhiều dòng nguyên liệu, thành phẩm và hạch toán; chuyển kho có hai phía; hàng trả lại và điều chỉnh cũng tạo chuyển động kho. Phải chọn sự kiện nghiệp vụ và công đoạn được trả công, tránh trả nhiều lần cho cùng một việc.

Đề xuất MVP chọn một số nghiệp vụ cụ thể cùng Nghĩa và kế toán. Việc trộn, đóng gói hoặc đóng thùng chỉ được tự sinh khi chứng từ nguồn thực sự chứng minh công việc đó đã xảy ra. Không suy ra đã đóng thùng chỉ từ lượng hàng xuất bán.

### 5.2. Phân công và chia sản lượng

- Mặc định có thể kế thừa cách chia đều của hệ thống cũ, nhưng cần xác nhận trước khi áp dụng.
- Hỗ trợ gán tổ theo ngày/ca để giảm thao tác; kế toán hoặc quản lý kho duyệt người thực tế tham gia.
- Trường hợp chia theo tỷ trọng/giờ công chỉ bổ sung nếu nghiệp vụ yêu cầu. Tổng tỷ trọng phải bằng 100% trong từng công việc được phân bổ.
- Mỗi công đoạn có phạm vi sản lượng riêng: cùng một lô có thể có công trộn và công đóng gói hợp lệ, nhưng không được tự động nhân đôi trong cùng công đoạn.
- Thiếu người làm, đơn giá hoặc quy đổi đơn vị phải báo lỗi dữ liệu; không lặng lẽ tính tiền bằng 0.
- Công việc ngoài sổ kho được nhập bổ sung có lý do, người duyệt và nguồn riêng, để không quay lại nhập tay toàn bộ sản lượng.

Ví dụ minh họa, chưa phải đơn giá áp dụng: một công việc có 1.000 kg, hai người chia đều, đơn giá 200.000 đồng/tấn, hệ số 1. Mỗi người nhận 500 kg và 100.000 đồng tiền công. Cần kiểm tra mẫu thực tế để xác nhận phép chia 1.000 trong code cũ có đúng là quy đổi kg sang tấn hay mang ý nghĩa khác.

### 5.3. Công, phép và tăng ca

Sổ kho không thay thế chấm công. Người có mặt nhưng chưa có chứng từ vẫn có thể được hưởng lương thời gian; thời điểm ghi sổ cũng không đủ xác định ca làm thực tế.

- Lưu công/nghỉ phép theo ngày, tăng ca theo ca hoặc khoảng thời gian và người thực hiện.
- Không lấy giờ hạch toán làm giờ tăng ca mặc định.
- Khi nhiều công việc thuộc cùng ca, tổng giờ tăng ca được kiểm soát độc lập với số dòng sản lượng.
- Cấu hình lịch làm việc, ngày lễ, phép hưởng lương và ngoại lệ nhân viên theo ngày hiệu lực.

### 5.4. Nghỉ phép — quản lý từng lần nghỉ

Phạm vi áp dụng: nhân viên kho và văn phòng; dữ liệu độc lập với nghiệp vụ lương sale.

- Màn hình danh sách nghỉ có lọc nhân viên, bộ phận, ngày/kỳ, loại nghỉ và trạng thái.
- Tạo đơn/ghi nhận nghỉ: nhân viên, ngày hoặc khoảng ngày, buổi sáng/chiều/cả ngày, loại nghỉ, lý do, hưởng lương hay không và có trừ quỹ phép hay không theo chính sách loại nghỉ.
- Khoảng nghỉ nhiều ngày được tách thành chi tiết từng ngày để xét lịch làm việc, ngày lễ, qua tháng và qua năm. MVP hỗ trợ đơn vị 0,5 ngày như cũ; nghỉ theo giờ chỉ bổ sung khi có yêu cầu.
- Đề xuất trạng thái: nháp → chờ duyệt → đã duyệt/từ chối; đơn đã duyệt có thao tác hủy theo quyền. Chốt với kế toán việc nhân viên tự đăng ký hay người phụ trách nhập hộ.
- Kiểm tra trùng buổi nghỉ, thời gian ngoài thời gian làm việc và số dư phép; quy tắc cho phép âm/vượt phép phải cấu hình và được duyệt.
- Chỉ nghỉ đã duyệt được đưa vào công/lương và trừ quỹ phép. Đơn chờ duyệt hiển thị riêng, không tự trừ số dư chính thức.
- Sửa đơn đã duyệt phải duyệt lại phần thay đổi; hủy phải hoàn phép đúng một lần và giữ lịch sử, không xóa dấu vết.
- Nghỉ và công/ca cùng thời gian phải được đối soát. Không tự suy luận có đi làm chỉ vì nhân viên được gán trên chứng từ kho.
- Quyền đề xuất: người nhập tạo/sửa nháp; quản lý duyệt nghỉ; nhân sự/kế toán quản lý chính sách và điều chỉnh. Phân quyền cụ thể chốt với người vận hành.

### 5.5. Ngày phép — quản lý quỹ và số dư

Màn hình **Ngày phép** theo năm, nhân viên và loại phép, hiển thị: số dư đầu kỳ, phép được cấp, phép chuyển năm trước, điều chỉnh, phép đã dùng, phép hết hạn, đơn đang chờ duyệt và số còn lại. Chi tiết nhân viên truy ra từng giao dịch cấp/trừ/hoàn/điều chỉnh.

Công thức đề xuất:

`Phép còn lại = số dư đầu kỳ + cấp mới + chuyển vào + điều chỉnh tăng - điều chỉnh giảm - đã dùng + hoàn phép - hết hạn`

- Số dư đầu kỳ phải quy định rõ đã bao gồm phép chuyển năm trước hay chưa; không cộng hai lần.
- Cấu hình cấp phép theo năm hoặc tích lũy theo tháng, ngày vào làm, thời gian hiệu lực; không tự gán một định mức chung khi chưa được xác nhận.
- Chính sách loại nghỉ có hai thuộc tính độc lập: tỷ lệ hưởng lương và có trừ quỹ phép. Ví dụ nghỉ hưởng lương không trừ phép và nghỉ phép năm cần được biểu diễn riêng.
- Chốt quy tắc chuyển phép qua năm, hạn sử dụng, thứ tự trừ phép cũ/mới và xử lý nhân viên nghỉ việc. Chỉ tự động cấp/chuyển/hết hạn sau khi quy tắc được duyệt.
- Cấp hoặc điều chỉnh hàng loạt có xem trước, lý do, người duyệt và nhật ký. Chạy lại tác vụ cấp/chuyển phép không được cộng thêm lần nữa.
- Ghi sổ biến động phép theo giao dịch và liên kết đơn nghỉ; kiểm soát đồng thời khi duyệt nhiều đơn để không vượt số dư ngoài chính sách.

### 5.6. Liên kết nghỉ phép với lương và khóa kỳ

- Tách rõ tổng ngày nghỉ, ngày nghỉ hưởng lương, ngày nghỉ không lương và ngày trừ quỹ phép trong bảng công.
- Với chính sách tương đương mã cũ: công tính lương = công theo lịch trong thời gian làm việc − ngày nghỉ thuộc lịch làm việc + ngày nghỉ hưởng lương. Loại nghỉ hưởng một phần lương, nếu có, cần công thức riêng được duyệt.
- Ngày lễ/ngày không làm việc trong khoảng nghỉ không tự động trừ phép hoặc trừ công; dựa trên lịch áp dụng cho nhân viên.
- Nghỉ có lương không tự sinh tiền sản lượng/đóng thùng; các khoản này vẫn dựa trên công việc thực tế đã duyệt.
- Phiên tính lương lưu snapshot đơn nghỉ và chính sách đã dùng. Kỳ đã chốt không thay đổi khi sửa đơn nghỉ; xử lý qua điều chỉnh hoặc mở kỳ có quyền.
- Việc hoàn/trừ lại quỹ phép do sửa đơn và việc điều chỉnh tiền lương phải có liên kết để đối soát, tránh hoàn phép nhưng bỏ sót điều chỉnh công.

### 5.7. Tính, duyệt và điều chỉnh

- Bảng nháp cho thấy từng khoản: lương công, tiền công việc, đóng thùng, tăng ca, phụ cấp, phát sinh, bảo hiểm, thuế, thực nhận.
- Từ tiền công mở được chi tiết sản lượng → người làm → đơn giá/hệ số → chứng từ nguồn.
- Chốt lương phải lưu snapshot nguồn, hồ sơ, quy tắc, đơn giá và kết quả; thay đổi cấu hình sau này không thay đổi kỳ đã chốt.
- Nếu nguồn kho thay đổi trước khi chốt: đánh dấu dữ liệu cần cập nhật và duyệt lại phần bị ảnh hưởng.
- Nếu nguồn thay đổi sau khi chốt: tạo điều chỉnh kỳ sau hoặc mở lại có quyền và lý do; lưu phiên cũ để đối chiếu.

## 6. Hướng triển khai kỹ thuật

### 6.1. Tổ chức chức năng

Đề xuất tạo nhóm chức năng **Lương kho và văn phòng**, gồm **Lương kho**, **Lương văn phòng**, **Chấm công**, **Nghỉ phép**, **Ngày phép**, **Sản lượng tính công**, **Đơn giá và hệ số**, **Phát sinh và điều chỉnh**. Nhóm này có route, API và quyền nghiệp vụ riêng. Chỉ liên kết hồ sơ định danh nhân viên và dùng hạ tầng phân quyền hiện có sau khi xác minh; phát sinh/điều chỉnh lương kho và văn phòng được quản lý riêng với sale.

Màn hình ưu tiên để kế toán xem sớm:

1. Sản lượng tính công: lọc ngày/kho/việc, xem nguồn chứng từ, gán tổ/người, hiển thị lỗi và duyệt.
2. Cấu hình công việc và đơn giá có hiệu lực theo ngày.
3. Bảng lương kho tháng, chi tiết nhân viên và truy ngược nguồn.
4. Nghỉ phép: đăng ký, duyệt, lịch nghỉ và đối soát công.
5. Ngày phép: cấp phép, số dư, chuyển phép và lịch sử biến động.
6. In/xuất bảng lương, phiếu lương và lịch sử phiên tính.

### 6.2. Mô hình dữ liệu đề xuất

Tên bảng dưới đây là thiết kế dự kiến; cần đối chiếu schema thật trước khi migration.

| Nhóm dữ liệu | Nội dung cần lưu |
| --- | --- |
| Nhóm lương nhân viên | Kho/văn phòng và thời gian hiệu lực; ánh xạ mã cũ sang nhân viên mới |
| Công việc tính lương | Mã việc, công đoạn, đơn vị tính và phương pháp tính |
| Quy tắc nguồn kho | Loại chứng từ/nghiệp vụ, chiều nhập/xuất, sản phẩm/kho áp dụng, quy đổi |
| Đơn giá/hệ số | Việc, loại ca/ngày, đơn giá, hệ số, ngày hiệu lực, phiên bản |
| Sự kiện sản lượng | Khóa nghiệp vụ nguồn, dòng chứng từ, công đoạn, ngày làm, số lượng gốc/quy đổi và snapshot |
| Phân bổ nhân viên | Sự kiện, nhân viên, ca/tổ, tỷ trọng, sản lượng hưởng và trạng thái duyệt |
| Chính sách phép | Loại nghỉ, hưởng lương, trừ quỹ, định mức/cách cấp, chuyển năm/hết hạn, hiệu lực |
| Đơn nghỉ và chi tiết ngày | Nhân viên, ngày/buổi, loại nghỉ, số ngày, lý do, trạng thái, người duyệt và phiên sửa |
| Sổ biến động phép | Nhân viên, năm/loại phép, cấp/trừ/hoàn/chuyển/hết hạn/điều chỉnh, số ngày, đơn nguồn, khóa chống trùng |
| Công/tăng ca | Nhân viên, ngày/ca, công hưởng, giờ tăng ca, loại ngày và phê duyệt; liên kết đơn nghỉ |
| Phiên lương và chi tiết | Kỳ, nhóm lương, trạng thái, phiên quy tắc, từng khoản thu nhập/khấu trừ, nguồn và lịch sử |

Số tiền và số lượng dùng `BigDecimal`/`DECIMAL`; chốt cách làm tròn ở cấp khoản và cấp tổng. Không dùng phép cộng int cho sản lượng có phần lẻ.

### 6.3. Chống trùng và thay đổi nguồn

- Khóa duy nhất phải dựa trên sự kiện/dòng nguồn ổn định + công đoạn + phạm vi phân bổ, không chỉ dựa trên `inventory_ledger.id`.
- Mã mới có các luồng xóa/sửa dòng sổ kho; cần kiểm tra tính ổn định của cả dòng chứng từ khi rewrite. Nếu không ổn định, bổ sung định danh nghiệp vụ bền vững và lưu mapping phiên nguồn.
- Nhiều dòng sổ của một dòng nghiệp vụ chỉ tạo một sự kiện tính công tương ứng, trừ trường hợp có nhiều công đoạn được duyệt.
- Đồng bộ lặp lại không tăng số lượng hoặc tiền công. Bỏ ghi sổ/hủy phải loại nguồn chưa chốt và báo tác động đến kỳ đã chốt.
- Có fingerprint/version dữ liệu nguồn để phát hiện chỉnh sửa; lưu snapshot để vẫn giải thích được bảng lương khi bản ghi nguồn bị thay đổi.
- Khi chốt phải kiểm tra lại phiên nguồn và thực hiện giao dịch để tránh nguồn thay đổi giữa lúc tính và lúc duyệt.

### 6.4. Xây luồng lương kho/văn phòng độc lập với lương sale

Tạo service tính lương kho và service tính lương văn phòng riêng, dựa trên quy tắc đã đối chiếu từ hệ thống cũ. Các phần tương đồng giữa hai nhóm như công/phép, cấu hình khấu trừ và lưu phiên kết quả có thể tổ chức trong module kho/văn phòng sau khi chốt nghiệp vụ.

- Tạo API, cấu hình, bảng phiên tính, kết quả, chi tiết và điều chỉnh riêng cho kho/văn phòng; tên cụ thể chốt ở thiết kế chi tiết.
- Không gọi pipeline sale, không ghi kết quả kho/văn phòng vào `payrolls` của sale và không mở rộng các màn hình sale để xử lý hai nhóm mới.
- `PayrollService.run(period)` hiện xóa `payrolls` theo kỳ. Đây là lý do phải giữ dữ liệu kho/văn phòng ngoài phạm vi chạy của service này; không đề xuất sửa pipeline sale trong phạm vi chuyển lương kho/văn phòng.
- Cấu hình bảo hiểm/thuế của sale cũng không mặc nhiên dùng chung. Xây cấu hình theo kỳ cho kho/văn phòng và để kế toán xác nhận. Nếu về sau muốn trích xuất thư viện tính toán dùng chung, đó là thay đổi riêng cần đánh giá và kiểm thử.
- Kiểm thử sự độc lập: chạy, chạy lại, chốt và điều chỉnh lương của mỗi module không thay đổi dữ liệu của module còn lại.

Đề xuất API cho các thao tác: xem trước nguồn, đồng bộ nguồn, cập nhật phân bổ, duyệt công, tính thử, lấy chi tiết, chốt và điều chỉnh. Tất cả thao tác ghi phải kiểm tra quyền backend, trạng thái kỳ và ghi nhật ký.

## 7. Lộ trình thực hiện

| Giai đoạn | Công việc | Kết quả bàn giao |
| --- | --- | --- |
| 0. Chốt mẫu nghiệp vụ | Cùng Nghĩa/kế toán chọn một kỳ lương, vài chứng từ đại diện; xác định đơn vị, công đoạn, cách chia, ngày tính công | Bộ mẫu đầu vào → kết quả và danh sách quy tắc được duyệt |
| 1. Demo lương kho | Nối nguồn kho cho nghiệp vụ ưu tiên, gán nhân viên, cấu hình đơn giá, tính thử và xem chi tiết | Kế toán kiểm tra được một tháng mẫu, truy ngược được số liệu |
| 2. Hoàn thiện kho | Hoàn thiện Nghỉ phép và Ngày phép (cấp, duyệt, số dư, hoàn/chuyển phép), công/tăng ca, phát sinh, khóa kỳ, sửa nguồn, phân quyền, in/xuất | Lương kho chạy độc lập và nghiệm thu |
| 3. Lương văn phòng | Chuyển công thức, dữ liệu đầu vào, bảng tháng/năm, phiếu lương; dùng cấu hình thuộc module kho/văn phòng | Lương văn phòng đối chiếu đạt |
| 4. Chuyển dữ liệu và ngừng cũ | Chuyển lịch sử, đối soát, vận hành song song trong kỳ được chọn, chuyển chính thức | Có dữ liệu lưu trữ và có thể ngừng gia hạn dịch vụ cũ |

Chưa chốt số ngày thực hiện trước khi có mẫu dữ liệu và danh sách nghiệp vụ nguồn. Giai đoạn 1 nên đủ nhỏ để kế toán xem sớm; các yêu cầu phát sinh từ Nghĩa được ghi thành danh sách thay đổi có mẫu kiểm chứng.

## 8. Chuyển dữ liệu và ngừng trang cũ

### 8.1. Dữ liệu cần kiểm kê và chuyển

- Nhân viên kho/văn phòng, mã nhân viên, nhóm lương và các tham số trên hồ sơ.
- Công việc, nhóm việc, đơn giá/hệ số, loại đóng thùng.
- Chi tiết công việc/đóng thùng, tăng ca, phát sinh.
- Ngày phép: danh mục loại phép, số ngày được cấp theo nhân viên/năm, số đã dùng và số dư tại ngày chuyển.
- Nghỉ phép: từng ngày/buổi nghỉ, loại nghỉ, cờ hưởng lương, liên kết nhân viên và lịch sử có thể lấy được. Không tự gán trạng thái “đã duyệt” cho dữ liệu cũ khi chưa có quy tắc chuyển được kế toán xác nhận.
- Chọn một phương án số dư: dựng lại từ lịch sử đầy đủ hoặc nhập số dư mở đầu tại ngày chuyển. Nếu nhập số dư mở đầu, các đơn trước ngày chuyển chỉ để tra cứu, không trừ lại vào số dư; đơn nghỉ vắt qua ngày chuyển phải tách phần trước/sau để đối soát.
- Bảng lương tháng/năm, chi tiết nhân viên và snapshot lịch sử; lưu mẫu phiếu lương nếu cần đối chiếu.
- Mapping ID nguồn sang ID mới; không dùng trực tiếp ID nhân viên giữa hai hệ thống.

Giữ số tiền lương lịch sử đã duyệt theo bản gốc. Không tính lại lịch sử bằng đơn giá hoặc quy tắc mới. Những kỳ muốn tính lại phải tạo phiên đối chiếu riêng.

### 8.2. Điều kiện ngừng gia hạn

1. Xác định ngày hết hạn và danh sách dịch vụ phải ngừng: hosting/server, database, tên miền, backup hoặc dịch vụ liên quan. Không suy luận tất cả đều có thể hủy cùng lúc.
2. Kiểm kê chức năng khác còn dùng trên trang cũ; hai module lương hoàn tất chưa đủ để tắt toàn trang nếu còn nghiệp vụ phụ thuộc.
3. Sao lưu database, mã nguồn và tệp đính kèm cần giữ; thử khôi phục bản sao lưu.
4. Nghiệm thu lương kho và văn phòng; đối chiếu tổng theo kỳ, từng nhân viên và các khoản cấu thành.
5. Chốt thời điểm ngừng nhập cũ, chuyển phần phát sinh cuối cùng, xác nhận số liệu mới và hướng dẫn người dùng.
6. Lưu lịch sử trong hệ thống mới hoặc kho lưu trữ có thể tra cứu; xác định người phụ trách và quyền truy cập.
7. Nếu kiểm tra cuối thất bại, dùng phương án quay lại đã thống nhất; tránh nhập đồng thời ở hai hệ thống mà không có nguồn chuẩn.
8. Chỉ ngừng dịch vụ khi người phụ trách xác nhận hoàn tất và đã kiểm tra các phụ thuộc. Mục tiêu là tránh kỳ gia hạn tiếp theo nhưng không đánh đổi việc mất dữ liệu hoặc gián đoạn trả lương.

## 9. Các nội dung cần chốt với Nghĩa và kế toán

| Câu hỏi | Vì sao cần chốt |
| --- | --- |
| “Sổ chi tiết kho” cụ thể là màn hình nào, loại chứng từ nào? | Chọn đúng nguồn, không tính lương trên mọi biến động kho |
| Những công việc nào hưởng tiền theo sản lượng? | Xác định trộn/đóng gói/bốc xếp/đóng thùng và tránh trùng công đoạn |
| Đơn vị gốc và đơn vị tiền công là kg, tấn, bao, hộp hay thùng? | Giải thích hệ số `/1000` và quy đổi theo từng sản phẩm |
| Chia đều như cũ hay theo người/giờ/tỷ trọng? Ai xác nhận tổ làm? | Dữ liệu kho không có đủ thông tin chia công |
| Có dữ liệu nào ghi nhận đóng thùng thực tế? | Quyết định tự động lấy nguồn hay cần ghi nhận bổ sung |
| Lấy ngày làm thực tế, ngày chứng từ hay ngày hạch toán làm kỳ lương? | Xử lý chứng từ nhập muộn và khác tháng |
| Đơn giá/hệ số cũ còn áp dụng? Ngoại lệ thủ kho còn tồn tại? | Chuyển thành cấu hình thay vì gắn với ID cũ |
| Công, phép và tăng ca lấy ở đâu; ai nhập, ai duyệt nghỉ? | Hoàn thiện phần không thể suy ra từ sản lượng |
| Ngày phép cấp theo năm hay tháng, có chuyển năm/hết hạn/cho âm không? | Chốt cách quản lý quỹ phép |
| Loại nghỉ nào hưởng lương, loại nào trừ quỹ phép? | Tránh đồng nhất nghỉ có lương với phép năm |
| Số dư phép tại ngày chuyển và lịch sử nghỉ đã được đối chiếu chưa? | Không trừ phép hai lần khi chuyển dữ liệu |
| Thuế, bảo hiểm và phụ cấp dùng cấu hình nào theo kỳ? | Tránh sao chép hằng số cũ hoặc tính trùng khoản |
| Cần chuyển bao nhiêu năm lịch sử? Kỳ nào làm chuẩn đối chiếu? | Xác định khối lượng migration và tiêu chí nghiệm thu |
| Ngày hết hạn trang cũ và các chức năng còn sử dụng? | Lập lịch chuyển và ngừng gia hạn khả thi |

## 10. Tiêu chí nghiệm thu

- Kế toán lấy được sản lượng của nghiệp vụ đã thống nhất từ nguồn kho, không nhập lại số lượng.
- Mỗi khoản tiền công truy ra được chứng từ, công đoạn, số lượng, nhân viên, đơn giá và hệ số.
- Đồng bộ/tính thử nhiều lần cho cùng nguồn không tạo công hoặc tiền trùng.
- Nhập nguyên liệu và nhập thành phẩm, hai chiều chuyển kho, chứng từ hủy/trả hàng/điều chỉnh được xử lý đúng bộ mẫu đã duyệt.
- Thiếu quy đổi, đơn giá hoặc người làm xuất hiện trong danh sách lỗi; không cho chốt phần dữ liệu chưa hợp lệ.
- Hai người chia sản lượng có phần lẻ vẫn bảo toàn tổng sản lượng và xử lý làm tròn đúng quy định.
- Một nhân viên làm nhiều việc trong một ca không bị nhân số giờ tăng ca.
- Kiểm tra ngày lễ trùng Chủ nhật, tháng ngắn, vào/nghỉ việc giữa tháng và dữ liệu qua năm.
- Sửa/bỏ ghi sổ trước chốt cập nhật đúng; sau chốt không âm thầm làm thay đổi lương đã duyệt.
- Chạy lương kho/văn phòng không xóa hoặc đổi kết quả nhóm lương hiện có ngoài phạm vi.
- Bảng lương mẫu mới khớp từng khoản với kỳ đối chiếu; sai khác do sửa quy tắc/làm tròn được giải thích và duyệt, không chỉ đối chiếu tổng thực nhận.
- Nghỉ nửa ngày/cả ngày, nhiều ngày, qua tháng/năm tính đúng theo lịch; không trùng buổi và không trừ phép vào ngày nghỉ theo lịch ngoài quy tắc đã duyệt.
- Nghỉ hưởng lương và nghỉ trừ quỹ được xử lý độc lập; ví dụ nghỉ không lương giảm công, nghỉ phép hưởng lương giữ công nhưng trừ quỹ đúng số ngày.
- Duyệt/hủy đơn và chạy lại cấp/chuyển phép nhiều lần không trừ/cộng trùng; hai đơn duyệt đồng thời không làm vượt số dư trái chính sách.
- Số dư đầu kỳ, giao dịch cấp/trừ/hoàn/hết hạn và số dư cuối kỳ đối soát được từng nhân viên. Dữ liệu nghỉ lịch sử không làm trừ lại số dư nhập khi chuyển.
- Sửa nghỉ sau chốt lương có lịch sử và điều chỉnh rõ ràng; kết quả đã chốt giữ nguyên cho đến khi có thao tác được phép.
- Quyền xem, sửa công, sửa đơn giá, chốt và mở kỳ được kiểm tra ở backend; có nhật ký thay đổi.
- Dữ liệu lịch sử đã chuyển tra cứu được; bản sao lưu cũ khôi phục được trước khi ngừng dịch vụ.

## 11. Các file làm căn cứ để triển khai tiếp

Đường dẫn trong hai cột dưới đây tương đối với thư mục nguồn tương ứng ở mục 2.

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

| Nguồn mới | Điểm tích hợp |
| --- | --- |
| `vl-shared/src/main/java/com/vlife/shared/jdbc/entity/inventory/InventoryLedger.java` | Nguồn sổ chi tiết kho |
| `vl-shared/src/main/java/com/vlife/shared/jdbc/entity/inventory/InventoryVoucher.java` | Chứng từ, trạng thái và liên kết nghiệp vụ |
| `vl-shared/src/main/java/com/vlife/shared/service/inventory/InventoryVoucherService.java` | Ghi sổ, rewrite và bỏ ghi sổ |
| `vl-shared/src/main/java/com/vlife/shared/service/salary/PayrollService.java` | Luồng lương sale cần giữ độc lập; không dùng để chạy lương kho/văn phòng |
| `vl-cms/src/features/inventory/ledger` | Giao diện nguồn sản lượng |
| `vl-cms/src/features/salary-setup`, `payroll-config`, `payroll-run`, `payroll-result`, `salary-adjustment` | Các màn hình lương sale; không thuộc phương án tái sử dụng |

**Bước tiếp theo đề xuất:** lấy một kỳ lương kho đã duyệt cùng các chứng từ kho tương ứng để lập bảng đối chiếu nguồn → công việc → người làm → tiền công, chốt với Nghĩa rồi triển khai demo lương kho.
