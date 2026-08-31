# Đội xe HAL

Website tĩnh cho đội xe HAL, chạy trên GitHub Pages. Website hiển thị dữ liệu chuyến xe, bảng lương và chấm công theo từng tháng từ Google Sheets; đăng nhập dùng Firebase Authentication.

## Cấu trúc

- `index.html`: website chính.
- `doi-xe-hal.html`: chuyển hướng tương thích về `index.html`.
- `config/sheet-links.json`: liên kết Google Sheet cho từng tháng.
- `config/firebase-config.js`: Firebase Web configuration (cần điền trước khi dùng đăng nhập).

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

`driverName` phải trùng hoàn toàn với cột **Họ và tên/Tên lái xe** trong dữ liệu tháng. Website không có form tự đăng ký; đây là chủ ý để người đăng ký không tự nhận quyền quản lý hoặc chọn dữ liệu của lái xe khác.

### Firestore Security Rules

Dán rules dưới đây vào **Firestore Database → Rules**, sau đó Publish. Mỗi người dùng chỉ có quyền đọc profile của chính họ; client không thể tự sửa quyền hoặc đổi tên lái xe.

```text
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId} {
      allow get: if request.auth != null && request.auth.uid == userId;
      allow list, create, update, delete: if false;
    }

    match /sheetMonths/{month} {
      allow get, list: if request.auth != null;
      allow create, update, delete: if request.auth != null
        && get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role == "manager";
    }
  }
}
```

Quản trị viên cần tạo/sửa profile qua Firebase Console, Firebase Admin SDK hoặc quy trình nội bộ đáng tin cậy — không thực hiện bằng website này.

## 2. Thêm Google Sheet cho tháng mới

Mỗi tháng sử dụng một file Google Sheet riêng và gồm ba tab:

- `Chi tiết chuyến`
- `Bảng lương`
- `Chấm công`

Mở từng tab trong Google Sheet; URL sẽ có dạng `.../edit?gid=123456789#gid=123456789`. Lấy giá trị `gid` của từng tab, sau đó thêm một object vào `config/sheet-links.json`:

```json
{
  "2026-10": {
    "sheetId": "ID_FILE_GOOGLE_SHEET_THANG_10",
    "tabs": {
      "trips": { "gid": "GID_CHI_TIET_CHUYEN" },
      "salary": { "gid": "GID_BANG_LUONG" },
      "attendance": { "gid": "GID_CHAM_CONG" }
    }
  }
}
```

Chỉ cần thêm một object theo tháng (`YYYY-MM`), **không cần sửa logic trong `index.html`**. Website tự chọn tháng hiện tại theo ngày hệ thống của thiết bị; khi tháng đó chưa được cấu hình, website hiển thị “Chưa có dữ liệu tháng này” và cho phép chọn tháng cũ.

Cấu hình tháng 9/2026 đang có sẵn:

- Sheet ID: `1p_it8Q0KJECq6F8vOL7tWbMOnKSEM4kPVgjL57LTyjE`
- Chi tiết chuyến: GID `767579772`
- Bảng lương: GID `774547220`
- Chấm công: GID `1350159733`

### Thêm tháng ngay trong app (quản lý)

Sau khi đăng nhập bằng tài khoản có `role: "manager"`, nút **Thêm tháng** xuất hiện ở thanh bên. Chọn tháng và dán URL file Google Sheet; app tự lấy Sheet ID, lưu vào Firestore collection `sheetMonths`, sau đó cập nhật danh sách tháng cho tất cả tài khoản đang mở website.

Ba tab trong file mới hiện cần giữ chính xác các tên: `Chi tiết chuyến`, `Bảng lương`, `Chấm công`. App tải CSV theo tên tab nên quản lý không phải tìm GID. Khi form hoặc tên tab thay đổi, cần cập nhật parser/cấu hình tương ứng.

### Quyền chia sẻ Google Sheet

Để URL export CSV hoạt động, mở từng file sheet → **Share** → **General access** → chọn **Anyone with the link** và quyền **Viewer**.

> Lưu ý bảo mật quan trọng: cách lấy CSV này yêu cầu Sheet công khai với bất kỳ ai có link. Firebase bảo vệ việc truy cập qua giao diện website nhưng **không thể ngăn** một người đã biết URL export CSV đọc sheet trực tiếp. Nếu dữ liệu phải hoàn toàn riêng tư, cần backend có xác thực thay cho Google Sheets CSV public; GitHub Pages tĩnh đơn thuần không giải quyết được yêu cầu đó.

## 3. Đổi mật khẩu

Người dùng đăng nhập bằng email/mật khẩu Firebase. Trong website, mở tùy chọn **Đổi mật khẩu**, nhập mật khẩu hiện tại và mật khẩu mới. Firebase sẽ xác thực lại chính tài khoản đang đăng nhập trước khi cập nhật; người dùng không thể đổi mật khẩu của người khác.

## 4. Chẩn đoán đăng nhập

Trang đăng nhập kiểm tra bốn giá trị bắt buộc trong `config/firebase-config.js` (`apiKey`, `authDomain`, `projectId`, `appId`), có timeout 15 giây cho Firebase Authentication và cho lần đọc profile Firestore. Khi gặp lỗi, nút sẽ trở về trạng thái **Đăng nhập** và hiển thị thông báo thay vì chờ vô hạn.

Mở trang web, nhấn **F12** (hoặc `Ctrl` + `Shift` + `I`), chọn tab **Console**, sau đó đăng nhập. Các log có tiền tố `[HAL Auth]` cho biết tiến trình dừng ở đâu:

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
