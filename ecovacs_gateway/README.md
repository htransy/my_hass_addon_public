# Ecovacs Private Gateway

Add-on đa thiết bị cho Ecovacs, xuất trực tiếp Home Assistant MQTT Discovery.
Không cần cài custom component.

## Tính năng

- đăng nhập Ecovacs và lưu credential/app token bằng AES-GCM;
- tự discovery profile, event và control cho nhiều model;
- nhận telemetry Ecovacs MQTT realtime, REST chỉ làm fallback;
- publish riêng từng entity và bỏ mọi payload không đổi;
- publish bản đồ SVG chỉ khi digest thay đổi;
- MQTT LWT, retained discovery/state và reconnect exponential nền;
- kết nối broker ngay khi add-on chạy, không chờ Ecovacs cloud;
- QoS 0 retained và tối đa 4 publish đồng thời để giảm độ trễ có kiểm soát;
- sensor/binary_sensor lồng nhau, button thao tác, room select và room button;
- nút Giặt giẻ/Sấy giẻ/Vệ sinh trạm cho profile X1 đã xác nhận;
- kéo chọn vùng trực tiếp trên bản đồ ingress để gửi lệnh `customArea`;
- queue có giới hạn, event burst coalesce và command concurrency có giới hạn;
- Web UI nhập broker/port/username/password/TLS/prefix;
- dashboard ingress, API chẩn đoán và quick command vẫn chạy trên port `7890`.

## Cài đặt nhanh

1. Chép thư mục này vào local add-ons và cài add-on.
2. Start, mở Web UI và đổi mật khẩu quản trị.
3. Nhập tài khoản Ecovacs.
4. Nhập broker MQTT Home Assistant trong mục MQTT trực tiếp.
5. Bật MQTT Discovery trong Home Assistant; entity tự xuất hiện.

Mật khẩu MQTT để trống khi sửa cấu hình sẽ giữ giá trị đã mã hóa. Để tắt bridge,
xóa hostname broker rồi lưu. Port `7890` không cần map ra LAN.

Xem hướng dẫn đầy đủ tại `DOCS.md` và kiến trúc tại
`../../docs/ADDON_ARCHITECTURE.md`.
