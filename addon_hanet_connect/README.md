# HANET Connect Gateway 0.9.0

Add-on quản lý hệ sinh thái HANET trực tiếp trong Home Assistant với giao diện
Ingress tiếng Việt, khóa truy cập riêng và API cục bộ có xác thực.

## Điểm mới trong 0.9.0

- Giao diện Aurora/Glass mới, tối ưu desktop, tablet và điện thoại.
- Màn hình mở khóa riêng trước khi tải dữ liệu camera và nhận diện.
- Mật khẩu ban đầu chỉ được đóng gói dưới dạng PBKDF2 hash, không hiển thị trong
  giao diện, log hoặc tài liệu phát hành.
- Cho phép đổi mật khẩu ngay trong **Cài đặt > Bảo mật giao diện**; mật khẩu mới
  cần tối thiểu 4 ký tự.
- Session `HttpOnly`, tự hết hạn sau 12 giờ và bị thu hồi toàn bộ khi đổi mật
  khẩu.
- Tạm khóa đăng nhập khi nhập sai nhiều lần liên tiếp.
- Webhook chỉ nhận secret qua header, không nhận secret trong URL.

## Tính năng

- Tự đăng nhập HANET Mobile API và HANET Connect Web để gộp camera sở hữu/chia sẻ.
- Tổng quan camera online, offline, RTSP và thời điểm đồng bộ gần nhất.
- Ảnh camera, ảnh sự kiện, lịch sử nhận diện và clip cloud theo ngày.
- Điều khiển PTZ, preset, zoom, còi báo động, cửa và các command được hỗ trợ.
- Quản lý Face ID thành viên/khách, thêm từng ảnh hoặc nhập nhiều ảnh.
- Quản lý nhân viên, phòng ban, biển số và dữ liệu chấm công.
- Bật/tắt RTSP, LED, IR, WDR, ghi hình, thông báo và setting theo model camera.
- Đồng bộ sự kiện qua SSE; tự chuyển sang cloud polling khi SSE không khả dụng.
- WebSocket nội bộ cập nhật dashboard mà không phải tải lại trang.
- API nâng cao cho toàn bộ catalog endpoint đã reverse engineering.
- API cục bộ bằng `X-HANET-Gateway-Key` và webhook bằng secret riêng.

## Cài đặt nhanh

1. Chép thư mục `addon_hanet_connect` vào `/addons/hanet_connect`.
2. Trong Home Assistant, mở **Settings > Apps > App store**.
3. Chọn **Check for updates**, sau đó cài **HANET Connect Gateway**.
4. Trong tab **Configuration**, nhập tài khoản và mật khẩu HANET.
5. Khởi động add-on và bật **Show in sidebar**.
6. Mở **HANET Connect**, nhập mật khẩu truy cập ban đầu do quản trị viên hoặc bộ
   cài cung cấp.
7. Vào **Cài đặt > Bảo mật giao diện** và đổi sang mật khẩu riêng.

Không cần ánh xạ cổng `9091` khi chỉ dùng giao diện Ingress. Chỉ bật cổng này khi
một hệ thống trong mạng nội bộ cần gọi API hoặc webhook; không mở cổng trên router
ra Internet.

## Hai lớp đăng nhập độc lập

- **Tài khoản HANET** nằm trong cấu hình add-on và dùng để gọi HANET Cloud.
- **Mật khẩu giao diện** chỉ khóa dashboard add-on, không thay đổi tài khoản HANET.
- Custom integration đăng nhập HANET bằng config entry riêng, không dùng session,
  mật khẩu giao diện hay API key của add-on.

## Sử dụng giao diện

- **Camera**: xem trạng thái, ảnh mới nhất, điều khiển và mở phần setting từng máy.
- **Sự kiện**: lọc theo ngày, camera, loại nhận diện, nguồn và từ khóa.
- **Ghi hình**: tìm clip cloud theo ngày/camera và xem trực tiếp trong dialog.
- **Danh tính**: quản lý Face ID, nhân viên, phòng ban, biển số và chấm công.
- **Cài đặt**: kiểm tra kết nối, đổi mật khẩu, khóa giao diện, bật RTSP và dùng API
  nâng cao.

## Bảo mật

- Hash mật khẩu giao diện được lưu tại `/data/ui_auth.json` với quyền owner-only.
- Token HANET được lưu riêng tại `/data/session.json` và không gửi tới trình duyệt.
- Credential P2P chỉ tồn tại ngắn hạn trong bộ nhớ và được truyền tới worker qua
  `stdin`.
- Session giao diện chỉ nằm trong bộ nhớ add-on; khởi động lại add-on sẽ yêu cầu
  mở khóa lại.
- Không ghi mật khẩu truy cập ban đầu vào README, giao diện hoặc API status.

Xem `DOCS.md` để biết cấu hình chi tiết, API và xử lý sự cố.
