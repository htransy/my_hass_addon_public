# Ecovacs Private Gateway

Add-on tập trung toàn bộ kết nối Ecovacs vào một dịch vụ riêng chạy trên cổng
nội bộ `7890`. Custom integration Home Assistant chỉ nhận dữ liệu đã chuẩn hóa,
hiển thị thực thể và gửi lệnh ngược về add-on.

## Tính năng

- đăng nhập tài khoản Ecovacs trực tiếp trong giao diện ingress;
- mã hóa credential cloud bằng AES-GCM trong `/data`;
- discovery đồng thời danh sách robot local/global của tài khoản;
- tự nạp profile theo class do `deebot-client` hỗ trợ;
- suy luận profile theo `deviceName` và `UILogicId` bất biến khi cloud trả class
  mới nhưng cùng family robot;
- hỗ trợ REST, MQTT P2P command, telemetry và bản đồ SVG;
- ưu tiên broker NGIOT `service.jmq` do cloud gán cho từng robot và hỗ trợ nhiều
  broker trong cùng tài khoản;
- tự thử kết nối lại MQTT theo chu kỳ nếu lần khởi tạo đầu bị timeout;
- giữ robot chưa có profile ở chế độ `raw` thay vì loại khỏi danh sách;
- MQTT lỗi sẽ chuyển gateway sang `degraded`, không làm mất toàn bộ discovery;
- bearer token riêng cho bridge Home Assistant và có thể đổi bất kỳ lúc nào;
- mật khẩu quản trị PBKDF2-SHA256, CSRF, rate-limit và session HttpOnly;
- cho phép đổi mật khẩu quản trị lại bất kỳ lúc nào.

## Model và profile

Gateway ưu tiên profile đúng class. Nếu class chưa có trong dependency, gateway
chỉ dùng các trường sản phẩm không do người dùng sửa để nhận family tương thích.
Ví dụ X1 Turbo và X1 Omni có thể dùng profile family đã có dù cloud cấp class
mới. Family X1 nội địa tự dùng map-set V2 và bỏ lệnh clean-log không được cloud
mod hỗ trợ. Trong mode `d248_full`, các nút start, pause, stop, dọn theo phòng
và control typed dùng MQTT P2P theo contract APK thay vì REST command. Panel
hiển thị `profile_status`, `profile_class` và broker MQTT đã chọn để kiểm tra.

Gateway mô tả capability bật/tắt, lựa chọn, số và hành động dưới dạng metadata
`switch`, `select`, `number` và `button`; bridge Home Assistant tạo entity đúng
loại và đọc state từ field event tương ứng.

Mọi event trong profile được đưa vào telemetry. Bridge tạo các sensor cốt lõi
**Mức pin**, **Trạng thái**, **Lỗi**, **Dữ liệu robot** và tự thêm sensor tiếng
Việt riêng khi xuất hiện event mới. Robot có map capability sẽ có image entity
**Bản đồ** SVG.

## Cài đặt

1. Chép thư mục `ecovacs_gateway` vào thư mục local add-ons của Home Assistant.
2. Refresh Add-on Store, cài add-on và giữ port `7890/tcp` ở trạng thái không
   publish nếu chỉ dùng trong mạng nội bộ add-on.
3. Start add-on và mở **Web UI**.
4. Đăng nhập bằng thông tin khởi tạo do người cài đặt cung cấp, sau đó thiết lập
   mật khẩu quản trị mới dài ít nhất 4 ký tự.
5. Chọn chế độ cloud, nhập tài khoản Ecovacs và bấm lưu/kết nối.
6. Sao chép bearer token trong mục **API nội bộ**.
7. Cài custom integration từ `custom_components/ecovacs_cn_mod`.
8. Nhập URL mặc định `http://local-ecovacs-private-gateway:7890` và bearer
   token. Nếu Supervisor tạo alias khác, dùng hostname nội bộ thực tế.

Không cần `install_id`, license hoặc activation key.

## Đổi mật khẩu sau này

1. Mở Web UI và đăng nhập bằng mật khẩu quản trị hiện tại.
2. Trong mục **Đổi mật khẩu quản trị**, nhập mật khẩu hiện tại và mật khẩu mới.
3. Sau khi đổi thành công, mọi session quản trị bị đăng xuất.
4. Đăng nhập lại bằng mật khẩu mới. Bearer token của bridge không đổi.

Nếu nhập sai quá nhiều lần, chờ 5 phút hoặc restart add-on để xóa rate-limit
đang lưu trong bộ nhớ.

## Trạng thái gateway

- `ready`: cloud và MQTT đã sẵn sàng;
- `degraded`: discovery vẫn hoạt động nhưng MQTT hoặc một subscription bị lỗi;
- `no_devices`: cloud không trả robot;
- `error`: đăng nhập hoặc khởi tạo cloud thất bại;
- `raw`: robot được discovery nhưng chưa tìm được profile tương thích;
- `inferred`: profile được suy luận từ metadata sản phẩm bất biến.

## Robot không có thực thể hoặc bản đồ

1. Kiểm tra bảng robot trong Web UI.
2. Nếu `profile_status=inferred`, kiểm tra `profile_class` và lỗi MQTT.
3. Broker của X1 nội địa cần là host `jmq-ngiot-...` lấy từ `service.jmq`; nếu
   lần đầu timeout, gateway sẽ tự thử lại mỗi 2 phút.
4. Nếu `profile_status=raw`, gửi phần metadata đã khử DID/resource/homeId để bổ
   sung family profile; không gửi token hoặc credential.
5. Bản đồ chỉ xuất hiện khi profile có map capability và robot đã trả map data.
6. Restart add-on sau khi nâng version để image mới được rebuild.

Dashboard `1.1.0` nhận SSE realtime, hiển thị riêng từng robot, map và quick
controls. Có thể chỉnh chu kỳ fallback sensor/map và quét thiết bị ngay trong
ingress; robot mới thêm vào tài khoản được nhận mà không cần restart.

## Bảo mật

- panel ingress chỉ hiển thị cho quản trị viên Home Assistant;
- API dữ liệu và lệnh yêu cầu bearer token;
- không bật CORS;
- port `7890` không được publish ra host theo cấu hình mặc định;
- không forward cổng này trên router hoặc public ra Internet;
- không ghi credential cloud vào log hoặc trả qua bridge API.
