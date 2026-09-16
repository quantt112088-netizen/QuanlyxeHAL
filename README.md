# Đội xe HAL

Website tĩnh cho đội xe HAL, chạy trên GitHub Pages. Website hiển thị dữ liệu chuyến xe, bảng lương và chấm công theo từng tháng từ Google Sheets; đăng nhập dùng Firebase Authentication.

## Cấu trúc

- `index.html`: website chính.
- `doi-xe-hal.html`: chuyển hướng tương thích về `index.html`.
- `config/sheet-links.json`: liên kết Google Sheet dự phòng (fallback/seed); cấu hình chính được quản lý trong Firestore.
- `config/firebase-config.js`: Firebase Web configuration (cần điền trước khi dùng đăng nhập).
- `firestore.rules`: Security Rules cho Firestore (dán vào Firebase Console khi triển khai).

## 1. Cấu hình Firebase Authentication

1. Tạo project tại [Firebase Console](https://console.firebase.google.com/).
2. Vào **Authentication → Sign-in method**, bật **Email/Password**.
3. Vào **Project settings → Your apps**, tạo hoặc chọn ứng dụng Web rồi copy các trường `apiKey`, `authDomain`, `projectId`, `appId` vào `config/firebase-config.js`.
4. Vào **Authentication → Settings → Authorized domains**, thêm domain GitHub Pages của website (ví dụ `ten-tai-khoan.github.io`) và domain tùy chỉnh nếu có.
5. Trong **Authentication → Users**, quản trị viên tự tạo từng tài khoản email/mật khẩu. Không bật đăng ký công khai nếu chưa có quy trình gán tài xế.

Firebase Web configuration được phép xuất hiện trong client. Tuy vậy, **không bao giờ** đưa service-account JSON, Admin SDK key, mật khẩu, hash mật khẩu hay token quản trị vào repo public.

### Hồ sơ người dùng Firestore

Bật **Firestore Database**. Sau khi tạo user trong Authentication, lấy UID của user và tạo document `users/<UID>` như sau:

```json
{
  "role": "driver",
  "driverName": "Họ và tên đúng như trong Google Sheet"
}
```

Tài khoản quản lý dùng:

```json
{
  "role": "manager"
}
```

Tài khoản chủ (owner) dùng một trong hai dạng sau — cả hai đều có quyền quản lý dữ liệu tháng:

```json
{
  "isOwner": true
}
```

hoặc:

```json
{
  "role": "manager",
  "isOwner": true
}
```

`driverName` phải trùng hoàn toàn với cột **Họ và tên/Tên lái xe** trong dữ liệu tháng. Website không có form tự đăng ký; đây là chủ ý để người đăng ký không tự nhận quyền quản lý hoặc chọn dữ liệu của lái xe khác.

Quyền quản lý dữ liệu tháng được cấp khi profile có `isOwner: true` **hoặc** `role: "manager"`. Tài khoản có `isOwner: true` nhưng không khai `role` hợp lệ được coi là quản lý.

### Firestore Security Rules

Dán rules dưới đây vào **Firestore Database → Rules**, sau đó Publish. Mỗi người dùng chỉ có quyền đọc profile của chính họ; client không thể tự sửa quyền hoặc đổi tên lái xe.

```text
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    function isOwnerOrManager() {
      let profile = get(/databases/$(database)/documents/users/$(request.auth.uid)).data;
      return profile.get("isOwner", false) == true
        || profile.get("role", "") == "manager";
    }

    match /users/{userId} {
      allow get: if request.auth != null && request.auth.uid == userId;
      allow list, create, update, delete: if false;
    }

    match /sheetMonths/{month} {
      allow get, list: if request.auth != null;
      allow create, update: if request.auth != null && isOwnerOrManager();
      allow delete: if false;
    }

    match /planCatalog/{category}/items/{itemId} {
      allow read, create, update, delete: if request.auth != null
        && isOwnerOrManager();
    }

    match /planEntries/{entryId} {
      allow read, create, update, delete: if request.auth != null
        && isOwnerOrManager();
    }

    match /repairEntries/{entryId} {
      allow read, create, update, delete: if request.auth != null
        && isOwnerOrManager();
    }

    match /planSettings/{settingId} {
      allow read, create, update, delete: if request.auth != null
        && isOwnerOrManager();
    }

    match /planSalaryAdjustments/{adjustmentId} {
      allow read, create, update, delete: if request.auth != null
        && isOwnerOrManager();
    }
  }
}
```

Đoạn trên trùng với file `firestore.rules` trong repo. Sau khi thay đổi rules, phải nhấn **Publish** trong Firebase Console. Quản trị viên cần tạo/sửa profile qua Firebase Console, Firebase Admin SDK hoặc quy trình nội bộ đáng tin cậy — không thực hiện bằng website này. Website chỉ cho phép lưu/cập nhật cấu hình tháng, không cho xóa.

### Nhập liệu (Beta) — Firestore độc lập

Khu vực **Nhập liệu (Beta)** chỉ xuất hiện với tài khoản quản lý/chủ tài khoản. Đây là module độc lập hoàn toàn với ba tab Google Sheets đang dùng: không đọc hoặc ghi Google Sheets, không dùng `sheetMonths`, không thay đổi parser hay công thức của các tab hiện có. Toàn bộ dữ liệu Beta nằm trong Firestore; cột FCR không thuộc schema Beta.

#### Collections Beta

- `planCatalog/{customers|drivers|vehicles|cargos}/items/{itemId}`: các danh mục Beta.
  - Trang **Danh mục dùng chung** chỉ có Khách hàng, Lái xe và Biển kiểm soát; từng nhóm là accordion và đóng mặc định.
  - Khách hàng: `name`, tùy chọn `company_name`, `company_address`, `tax_code`, `company_email`.
  - Lái xe: `name`, tùy chọn `vehicle_received_date` (`YYYY-MM-DD`). Ngày nhận xe là nguồn duy nhất xác định trạng thái lương cơ bản.
  - Biển kiểm soát: `name`, tùy chọn `tonnage`.
  - Trang **Danh mục Cargo & Tuyến vận chuyển** quản lý Cargo: `name`, `routes` (mảng tối đa 7 tuyến), một Cargo trên mỗi dòng bảng. Nút **Nạp 26 Cargo mẫu** tạo Cargo chưa có và cập nhật `name`/`routes` của Cargo trùng tên đã chuẩn hóa theo bộ mẫu.
- `planEntries/{entryId}`: chuyến Plan, với các key tiếng Anh `snake_case`. Tất cả record mới ghi theo schema này; record camelCase cũ vẫn đọc được và chỉ được chuẩn hóa khi chính record đó được lưu lại.
- `planSettings/salary_parameters`: tham số lương chung Beta.
- `planSalaryAdjustments/{driverId}_{rangeStart}_{rangeEnd}`: điều chỉnh `leave_pay`, `housing_eligible` và `insurance_eligible` theo lái xe/kỳ.

Thông tin danh mục còn thiếu luôn hiển thị **Chưa cập nhật**; ứng dụng không tự tạo dữ liệu giả. Khi chọn biển số trong Plan, `tonnage` được lấy từ danh mục và snapshot vào chuyến đã lưu, vì vậy sửa danh mục sau đó không thay đổi lịch sử.

#### Nhập chuyến (Plan)

Các trường Plan theo đúng thứ tự xuất Excel là:

```text
stt, customer, pickup_date, pickup_time, loading_done_time, arrival_time,
return_done_time, attendance, driver, plate, tonnage, cargo, route,
km_pickup, km_return, km_trip, note, vehicle_revenue, warehouse_fee,
loading_fee, consolidation_fee, sunday_fee, wait_time_fee, second_meal_fee,
overtime_fee, turnaround_fee, rot_diem_cho_xe,
luong_lai_xe_theo_chuyen, fuel, epass, trip_extra_cost,
rot_diem_thu_khach, customer_waiting_fee, overnight_fee,
other_customer_charge, cuoc
```

- Bắt buộc nhập `customer`, `pickup_date` và `driver`. Chấm công có thể để trống, chọn **Có** (1 công) hoặc **Nửa ngày** (0,5 công); không có lựa chọn `Không`.
- `cargo` là dropdown danh mục Cargo; `route` là dropdown phụ thuộc tuyến của Cargo đã chọn. `route` chỉ tự chọn sẵn khi Cargo có đúng một tuyến và vẫn có thể đổi lại. Lựa chọn cuối **Khác (nhập tay)** mở ô nhập tuyến phát sinh; Cargo không có trong danh mục dùng chế độ nhập tay cả hai ô. Dữ liệu lưu vẫn là chuỗi `cargo`/`route`.
- `km_trip = km_return - km_pickup` và `luong_lai_xe_theo_chuyen = round(vehicle_revenue × 16%)` được tính tự động.
- **Phí trả lái xe** chỉ là tiêu đề nhóm, không phải field.
- Form dùng 4 khối trình bày: `Doanh thu trả cho xe` đứng riêng; đỏ cho chi phí lái xe được nhận; xanh lá cho chi phí khi xe hoạt động; xanh dương cho doanh thu thu khách hàng. Đây chỉ là thay đổi trình bày, không đổi key, thứ tự xuất Excel hay công thức.
- Nhóm A là chi phí trả cho xe/lái xe; Nhóm B là chi phí của chuyến; Nhóm C là cước tính khách. `rot_diem_cho_xe` và `rot_diem_thu_khach` là hai field độc lập.
- Bảng xem/sửa hỗ trợ lọc theo tháng hoặc khoảng `Từ ngày`/`Đến ngày` (khoảng ngày ưu tiên tháng), Biển kiểm soát và từ khóa Khách hàng/Lái xe/Tuyến theo điều kiện AND. Tổng số dòng và `vehicle_revenue` luôn tính từ đúng tập dữ liệu đã lọc.
- Xuất Plan dùng XLSX có định dạng: thông tin chuyến màu navy, Nhóm A đỏ, Nhóm B xanh lá, Nhóm C xanh dương; ngày dùng `dd/mm/yyyy`, tiền dùng `#,##0`.

#### Bảng kê SUB và Chấm công (Beta)

Bảng kê SUB và Chấm công (Beta) dùng cùng một kỳ mặc định, tính bao gồm từ ngày 26 tháng trước đến ngày 25 tháng hiện tại.

- Bảng kê SUB lọc theo đúng tên khách hàng và `pickup_date` trong kỳ; chỉ dùng Nhóm C. Mỗi dòng tính `total_fare = rot_diem_thu_khach + customer_waiting_fee + overnight_fee + other_customer_charge + cuoc`, `vat = round(total_fare × 8%)`, và `total_payment = total_fare + vat`. Báo cáo/XLSX hiển thị `company_name`, `company_address`, `tax_code`, `company_email` của khách hàng theo danh mục; thông tin thiếu hiển thị **Chưa cập nhật**.
- XLSX SUB có tiêu đề/metadata theo mẫu Hoàng Anh Logistics, header xanh lá, ngày `dd/mm/yyyy`, tiền `#,##0`. Sau dữ liệu là dòng **TỔNG CỘNG** in đậm, kẻ đôi phía trên, cộng Rớt điểm, Phí chờ giờ, Lưu ca, Phát sinh khác, Cước, Tổng cước, VAT và Tổng thanh toán. Khối ký gồm tên công ty khách hàng bên trái và **CÔNG TY TNHH TM & DV HOÀNG ANH LOGISTICS** bên phải, với dòng nghiêng `(Ký, đóng dấu)` dưới mỗi bên.
- Chấm công được suy ra trong bộ nhớ từ tên lái xe đã chuẩn hóa và `pickup_date`; không ghi document chấm công và không có thao tác sửa tay. Mọi lái xe trong danh mục đều hiển thị. Một hay nhiều chuyến cùng ngày dùng giá trị cao nhất: `Có` (1 công) ưu tiên hơn `Nửa ngày` (0,5 công). Chủ nhật có thể hiển thị dấu `X` hoặc `½` nhưng không cộng công. Bảng có tổng **Ngày công (tổng)**, **Ngày công thử việc** và **Ngày công chính thức**.

#### Chi phí vận hành (Beta)

Đây là trang báo cáo chỉ đọc, dùng duy nhất `planEntries` trong kỳ 26 tháng trước đến 25 tháng hiện tại. Trang không có form nhập liệu, không tạo collection và không ghi Firestore. Nút xuất XLSX tạo hai sheet có header màu, ngày `dd/mm/yyyy` và tiền `#,##0`: tổng hợp chi phí và doanh thu/chi phí/lợi nhuận theo khách hàng.

- **Phần A** tổng hợp 14 hạng mục: `vehicle_revenue`, `warehouse_fee`, `loading_fee`, `consolidation_fee`, `sunday_fee`, `wait_time_fee`, `second_meal_fee`, `overtime_fee`, `luong_lai_xe_theo_chuyen`, `turnaround_fee`, `rot_diem_cho_xe`, `fuel`, `epass`, `trip_extra_cost`.
- **TỔNG CHI PHÍ** chỉ cộng dòng 1–8 và 10–14. `luong_lai_xe_theo_chuyen` (dòng 9) chỉ để tham khảo, vì đã nằm trong `vehicle_revenue`; hệ thống tuyệt đối không cộng lại khoản này.
- **Phần B** lặp toàn bộ khách hàng trong danh mục, kể cả khách hàng không có chuyến. Với từng khách hàng, `customer_revenue` là tổng Nhóm C (`rot_diem_thu_khach + customer_waiting_fee + overnight_fee + other_customer_charge + cuoc`), `total_cost` là chi phí của chính các chuyến của khách đó theo công thức Phần A (trừ `luong_lai_xe_theo_chuyen`), và `profit = customer_revenue - total_cost`. Hàng tổng chỉ tổng hợp các dòng khách hàng đang hiển thị.

#### Sửa chữa (Beta)

Nhật ký sửa chữa/bảo dưỡng xe thực tế, lưu riêng trong `repairEntries/{entryId}`. Nguồn này độc lập hoàn toàn với `planEntries`, không suy ra từ chuyến Plan và không cộng qua lại với **Chi phí vận hành (Beta)**. Danh sách, tổng hợp theo xe và xuất Excel đều dùng tháng dương lịch (`YYYY-MM`) suy ra từ `ngay_gio`; không có trường tháng được lưu trong document.

- `ngay_gio` (bắt buộc, `YYYY-MM-DD`), `bien_kiem_soat` (dropdown danh mục Biển kiểm soát), `lai_xe` (dropdown danh mục Lái xe), `cong_viec` (bắt buộc: `Sửa chữa`, `Bảo dưỡng`).
- `giam_sat_sua_chua`, `don_vi_cung_cap`, `nhom_sua_chua` là text tự do có datalist từ các giá trị đã nhập trước đó trong `repairEntries`; không có collection danh mục riêng.
- `so_luong` mặc định `1`; `vat_percent` mặc định `8`.
- `thanh_tien_chua_vat` là số tiền nhập tay, không nhân tự động từ `so_luong × don_gia`. `tong_tien = round(thanh_tien_chua_vat × (1 + vat_percent / 100))`, hiển thị readonly và lưu snapshot.
- `thanh_toan` (`Ghi nợ`, `Tiền mặt`, `Chuyển khoản`) và `chung_tu` (`Hóa đơn VAT`, `Phiếu thu`, `Khác`) là dropdown tùy chọn.
- Thứ tự thô của form/bảng/xuất Excel là: `ngay_gio`, `km_hien_tai`, `bien_kiem_soat`, `lai_xe`, `giam_sat_sua_chua`, `don_vi_cung_cap`, `code`, `cong_viec`, `nhom_sua_chua`, `dien_giai`, `dvt`, `so_luong`, `don_gia`, `thanh_tien_chua_vat`, `vat_percent`, `tong_tien`, `don_vi_chiu_trach_nhiem`, `thanh_toan`, `chung_tu`, `so_dntt`, `xac_nhan_thanh_toan`, `ghi_chu`, `so_hoa_don`.
- Trang **Nhập sửa chữa** lưu/sửa qua `repairEntries`; trang **Sửa chữa (Beta)** lọc theo tháng dương lịch, tìm theo Biển kiểm soát/Đơn vị cung cấp, sửa/xóa từng dòng, tổng hợp `tong_tien` theo từng Biển kiểm soát trong tháng rồi thêm hàng **TỔNG CỘNG**, và xuất XLSX định dạng màu header, ngày `dd/mm/yyyy`, tiền `#,##0`.

#### Tham số và Bảng lương (Beta)

Tham số lương mặc định chỉ hiển thị khi chưa có `planSettings/salary_parameters`; chúng chỉ được ghi sau khi quản lý nhấn lưu. Bảng lương dùng ngày công Beta, toàn bộ lái xe trong danh mục (kèm lái xe ngoài danh mục nhưng có chuyến) và tổng Nhóm A của các chuyến cùng kỳ.

- Trạng thái lương cơ bản suy ra tự động từ `vehicle_received_date`; không còn `base_salary_status_override`. Thử việc tính từ ngày nhận xe đến hết cùng ngày của tháng kế tiếp; ví dụ nhận xe 15/01 thì thử việc hết 15/02 và chính thức từ 16/02.
- Không có `vehicle_received_date` (gồm cả lái xe ngoài danh mục): lương cơ bản và mọi khoản phụ cấp theo ngày công bằng 0, nhưng **Lương chuyến** và tổng Nhóm A vẫn được tính.
- `Tổng phụ cấp A` chỉ gồm `warehouse_fee`, `loading_fee`, `consolidation_fee`, `sunday_fee`, `wait_time_fee`, `second_meal_fee`, `overtime_fee`. `turnaround_fee` và `rot_diem_cho_xe` không nằm trong công thức.
- `Lương cơ bản = round(((công thử việc × 85% + công chính thức) / 26) × lương cơ bản/tháng × hệ số lương)`.
- `PC ăn trưa = ngày công × 40.000`. Các khoản theo tháng được chia theo ngày công chuẩn: `PC xăng dầu`, `PC công việc`, `PC điện thoại = round(mức tháng / 26 × ngày công)`.
- `PC chuyên cần` chỉ tính khi ngày công đủ 26. Hỗ trợ nhà ở chỉ áp dụng khi điều chỉnh kỳ chọn `housing_eligible: Có`, có ngày công chính thức và đủ dữ liệu thâm niên kể từ ngày bắt đầu chính thức (tháng 1–3: 350.000; từ tháng 4: 500.000).
- Cột **Bảo hiểm (NLĐ đóng)** chỉ tính khi điều chỉnh kỳ chọn **Hưởng BH? = Có** và người đó có ngày nhận xe: `round(lương cơ bản/tháng × hệ số lương × 10,5%)`; giá trị này bị trừ khỏi tổng thực lĩnh.
- `Tổng lương CB + PC` cộng `PC công việc` **hai lần** theo đúng công thức nguồn đã duyệt. `leave_pay` (Nghỉ phép, mặc định 0) hiển thị/sửa riêng nhưng **không cộng vào Tổng lương CB + PC hoặc Tổng thực lĩnh**.
- `Tổng thực lĩnh = Tổng lương CB + PC + Lương chuyến + Tổng phụ cấp A − Bảo hiểm`.
- Bảng lương và các báo cáo Beta đều có xuất XLSX trong trình duyệt bằng SheetJS CDN.

Sau khi triển khai code mới, dán nội dung [firestore.rules](firestore.rules) cập nhật vào **Firestore Database → Rules** và nhấn **Publish**. Trước bước này, Firestore sẽ từ chối đọc/ghi các collection Beta mới theo rules cũ.

## 2. Thêm Google Sheet cho tháng mới

Mỗi tháng sử dụng một file Google Sheet riêng và gồm ba tab:

- `Chi tiết chuyến`
- `Bảng lương`
- `Chấm công`

### Quản lý trực tiếp trong website

Đăng nhập bằng tài khoản có `isOwner: true` hoặc `role: "manager"`, rồi:

1. Chọn **Quản lý dữ liệu tháng** ở thanh bên.
2. Chọn tháng/năm cần thêm hoặc cập nhật.
3. Dán URL Google Sheet, ví dụ `https://docs.google.com/spreadsheets/d/.../edit`.
4. Nhấn **Lưu tháng**.

Website lấy Sheet ID từ URL, lưu document `sheetMonths/<YYYY-MM>` vào Firestore và cập nhật cấu hình đang chạy qua listener thời gian thực. Sau khi lưu, tháng đó có thể được sử dụng ngay; **không cần sửa file JSON, sửa code, commit hoặc push GitHub**.

Danh sách trong cửa sổ quản lý cho biết cấu hình nào đến từ **Firestore** và cấu hình nào chỉ đến từ **File tĩnh**. Chọn **Chỉnh sửa** để đưa một tháng file tĩnh lên Firestore hoặc cập nhật URL/GID của tháng đã lưu. Giao diện chỉ hỗ trợ lưu/cập nhật, không có chức năng xóa.

### GID từng tab (không bắt buộc)

Nếu ba tab giữ đúng tên chuẩn ở trên, có thể để trống toàn bộ GID; website tải CSV theo tên tab. Để cấu hình chắc chắn hơn hoặc khi tên tab khác, mở phần **GID từng tab (không bắt buộc)** và nhập GID cho từng tab. GID là dãy số xuất hiện trong URL khi mở tab, ví dụ `.../edit#gid=123456789`.

Nếu URL chính được dán có `gid=...` và ô GID **Chi tiết chuyến** đang trống, website dùng GID đó cho tab Chi tiết chuyến. Khi chỉnh sửa một cấu hình Firestore, để trống một ô GID rồi lưu sẽ xóa GID cũ của tab đó và quay về tải theo tên tab chuẩn.

### File cấu hình dự phòng

`config/sheet-links.json` chỉ còn là fallback/seed để website vẫn có các tháng mặc định khi Firestore chưa có dữ liệu hoặc tạm thời không đọc được. Khi cùng một tháng tồn tại ở cả hai nơi, cấu hình Firestore được ưu tiên. Quy trình thêm tháng thông thường không chỉnh file này.

Website tự chọn tháng hiện tại theo ngày hệ thống của thiết bị; khi tháng đó chưa được cấu hình, website hiển thị “Chưa có dữ liệu tháng này” và cho phép chọn tháng cũ.

### Quyền chia sẻ Google Sheet

Để URL export CSV hoạt động, mở từng file sheet → **Share** → **General access** → chọn **Anyone with the link** và quyền **Viewer**.

> Lưu ý bảo mật quan trọng: cách lấy CSV này yêu cầu Sheet công khai với bất kỳ ai có link. Firebase bảo vệ việc truy cập qua giao diện website nhưng **không thể ngăn** một người đã biết URL export CSV đọc sheet trực tiếp. Nếu dữ liệu phải hoàn toàn riêng tư, cần backend có xác thực thay cho Google Sheets CSV public; GitHub Pages tĩnh đơn thuần không giải quyết được yêu cầu đó.

## 3. Đổi mật khẩu

Người dùng đăng nhập bằng email/mật khẩu Firebase. Trong website, mở tùy chọn **Đổi mật khẩu**, nhập mật khẩu hiện tại và mật khẩu mới. Firebase sẽ xác thực lại chính tài khoản đang đăng nhập trước khi cập nhật; người dùng không thể đổi mật khẩu của người khác.

## 4. Chẩn đoán đăng nhập

Trang đăng nhập kiểm tra bốn giá trị bắt buộc trong `config/firebase-config.js` (`apiKey`, `authDomain`, `projectId`, `appId`), có timeout 15 giây cho Firebase Authentication và cho lần đọc profile Firestore. Khi gặp lỗi, nút sẽ trở về trạng thái **Đăng nhập** và hiển thị thông báo thay vì chờ vô hạn.

Mở trang web, nhấn **F12** (hoặc `Ctrl` + `Shift` + `I`), chọn tab **Console**, sau đó đăng nhập. Các log có tiền tố `[Hoàng Anh Auth]` cho biết tiến trình dừng ở đâu:

- `Bắt đầu đăng nhập Firebase` → yêu cầu Auth đã được gửi.
- `Firebase Auth đăng nhập thành công` → email/mật khẩu hợp lệ.
- `bắt đầu đọc hồ sơ Firestore` → đang đọc `users/<UID>` để lấy role.
- `Đọc hồ sơ Firestore thành công/thất bại` → xác định lỗi profile hoặc Firestore Security Rules.

Trước khi triển khai thật, kiểm tra **Firebase Console → Authentication → Settings → Authorized domains** có đúng domain GitHub Pages đang chạy website (ví dụ `ten-tai-khoan.github.io`) và domain tùy chỉnh (nếu có). Domain chưa được cấp phép là nguyên nhân phổ biến khiến đăng nhập thất bại.

## 5. Triển khai GitHub Pages

1. Commit và push toàn bộ các file, gồm cả thư mục `config/`.
2. Trong GitHub repo, vào **Settings → Pages** và chọn branch/folder đang chứa `index.html`.
3. Sau khi deploy, thêm domain GitHub Pages vào Firebase **Authorized domains**.
4. Mở site bằng HTTP server hoặc GitHub Pages; không mở trực tiếp bằng `file://` vì browser có thể chặn tải JSON/CSV module.
