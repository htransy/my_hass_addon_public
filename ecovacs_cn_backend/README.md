# Ecovacs China Backend cho Home Assistant

Ecovacs China Backend `1.2.0` là hệ thống điều khiển robot Ecovacs tài khoản
Trung Quốc theo kiến trúc tách rời:

- **Add-on** xử lý đăng nhập Ecovacs, cloud REST, MQTT, trạng thái, bản đồ và
  lệnh điều khiển trên cổng nội bộ `4545`.
- **Custom integration** nhận thay đổi qua WebSocket nội bộ và tạo entity chuẩn
  Home Assistant theo capability an toàn mà add-on quảng bá.

Tài khoản Ecovacs không được nhập hoặc lưu trong config entry của custom
integration. Đây không phải phần mềm chính thức của Ecovacs.

## Kiến trúc

```text
Ecovacs China cloud + MQTT
            │
            ▼
Ecovacs Backend add-on :4545
  ├─ đăng nhập và lưu credential mã hóa
  ├─ nhận trạng thái robot
  ├─ dựng/cache bản đồ SVG
  └─ kiểm tra và thực thi lệnh
            │ WebSocket push + API token nội bộ
            ▼
Custom integration
  ├─ vacuum / lawn_mower
  ├─ sensor / setting
  ├─ image bản đồ
  └─ button/action
            │
            ▼
Home Assistant
```

## Tính năng

### Đăng nhập Ecovacs trong add-on

- Ecovacs ID và mật khẩu.
- Số điện thoại và mật khẩu.
- Số điện thoại và mã SMS.
- Số điện thoại Trung Quốc nhập nhầm ở tab Ecovacs ID được tự chuyển sang
  luồng điện thoại, không làm thay đổi cách xử lý ID thông thường.
- Lưu phiên đăng nhập để tự kết nối lại sau khi khởi động add-on.
- Xóa toàn bộ tài khoản Ecovacs khỏi add-on bằng giao diện quản trị.

### Sensor trạng thái

Tùy capability robot thực tế cung cấp, integration có thể tạo:

- trạng thái online/offline, pin và trạng thái làm việc;
- mã lỗi và mô tả lỗi;
- diện tích, thời gian và tổng số lượt dọn;
- cường độ Wi-Fi và địa chỉ IP;
- lực hút, mức nước, tình trạng gắn giẻ và chế độ lau;
- trạng thái trạm sạc/trạm Omni;
- phiên bản, trạng thái và tự động cập nhật firmware;
- phần trăm và thời gian còn lại của chổi, lọc, chổi cạnh và vật tư khác;
- trạng thái không làm phiền trên profile T9 POWER tương thích.

### Bản đồ

- Hiển thị bản đồ robot dưới dạng ảnh SVG.
- Entity bản đồ luôn được tạo khi robot quảng bá capability map, kể cả lúc SVG
  đầu tiên chưa tải xong.
- Cache bản đồ trong add-on để custom integration không xử lý dữ liệu cloud.
- Theo dõi revision và tên bản đồ.
- Khi robot di chuyển, event map từ MQTT dựng revision SVG mới tối đa khoảng
  mỗi giây; custom chỉ tải một lần cho mỗi revision rồi dùng cache nội bộ.
- Chu kỳ map 30 giây chỉ là fallback khi event bị bỏ lỡ, không phải độ trễ
  realtime bình thường.

### Realtime và tải Home Assistant

- Add-on chỉ phát WebSocket khi trạng thái, control hoặc revision bản đồ thật sự
  thay đổi; lúc robot nghỉ không gửi lại snapshot lặp.
- SVG không đi qua WebSocket. Socket chỉ gửi metadata/revision nhỏ, còn ảnh được
  tải từ API local khi revision đổi.
- Mỗi entity so sánh phần DTO riêng trước khi ghi state, vì vậy map di chuyển
  không ép toàn bộ sensor/control ghi lại mỗi giây.
- Poll local 60 giây và refresh cloud 120 giây là đường dự phòng. Dữ liệu giống
  hệt không tạo update mới trong Home Assistant.
- Sau login mật khẩu/SMS, add-on chuyển thẳng authenticator đã xác thực sang
  controller thay vì đăng nhập portal lần thứ hai.

### Nút điều khiển

Tùy capability thiết bị, add-on có thể cung cấp:

- bắt đầu, tạm dừng và dừng dọn;
- về trạm và tìm robot;
- hút rác, giặt giẻ, sấy giẻ và vệ sinh đế trạm;
- đặt lại tuổi thọ vật tư;
- chọn lực hút, mức nước và số lượt dọn;
- bật/tắt tự động cập nhật firmware;
- bật/tắt không làm phiền;
- chọn lau tiêu chuẩn/lau sâu trên T9 POWER;
- yêu cầu làm mới trạng thái ngay lập tức.

Các nút chỉ được tạo khi add-on xác nhận robot hỗ trợ capability tương ứng.

### Thực thể vacuum và thiết lập

- Mỗi robot hút bụi có một thực thể `vacuum` chuẩn để bắt đầu, tạm dừng, dừng,
  về trạm, tìm robot và đặt lực hút.
- Các lựa chọn như lực hút, mức nước, work mode, auto-empty và bản đồ đang dùng
  xuất hiện dưới dạng `select`.
- Các thiết lập bật/tắt như khóa trẻ em, nhiều bản đồ, tự động cập nhật firmware
  và các capability tương thích xuất hiện dưới dạng `switch`.
- Số lượt dọn, âm lượng, tần suất tự giặt giẻ hoặc mức nước tùy chỉnh xuất hiện
  dưới dạng `number` khi profile thiết bị có hỗ trợ.
- Trạng thái boolean được đưa vào `binary_sensor`; sensor số/chữ, ảnh bản đồ và
  button trạm/vật tư tiếp tục được tạo đầy đủ từ DTO của add-on.
- Thiết bị GOAT có profile tương thích được tạo thành thực thể `lawn_mower`
  chuẩn với bắt đầu cắt, tạm dừng và về trạm theo capability thực tế.

### Hỗ trợ thiết bị nội địa

- DEEBOT T10 OMNI được nhận diện tự động từ định danh sản phẩm bất biến của
  Ecovacs, gồm các biến thể OMNI, OMNIWHITE và CURIEOMNI đã xác nhận trong APK.
- Class `hxm494` của T10 OMNI nội địa được nhận diện trực tiếp ngay cả khi tên
  sản phẩm trong phản hồi cloud bị thiếu hoặc thay đổi.
- T10 OMNI dùng profile DT10/X1 OMNI bảo thủ: giữ lệnh dọn legacy để tránh ép
  thiết bị sang giao thức live V2, đồng thời bật capability trạm OMNI khi robot
  thực sự quảng bá hỗ trợ.
- T10 Turbo/Newton và model OMNI không cùng dòng không bị nhận nhầm thành T10
  OMNI.
- Với class nội địa chưa có trực tiếp trong `deebot-client`, add-on chuẩn hóa
  vùng `cn`/`ww` trong `UILogicId` và tìm profile quốc tế tương đương từ catalog
  218 profile đã đóng gói.
- DEEBOT và GOAT chưa có cặp khớp chính xác dùng profile nền cùng thế hệ một
  cách bảo thủ để ưu tiên trạng thái realtime và các lệnh cơ bản.
- WINBOT, AIRBOT và thiết bị legacy chưa có protocol tương ứng vẫn xuất hiện
  dưới dạng sensor metadata; add-on không quảng bá lệnh hoặc bản đồ chưa được
  xác minh, và một thiết bị kiểu này không còn làm lỗi đăng nhập cả tài khoản.

## Yêu cầu

- Home Assistant OS hoặc Supervised có hỗ trợ local add-on/app.
- Kiến trúc `amd64` hoặc `aarch64`.
- Robot đã được thêm vào Ecovacs Home vùng Trung Quốc.
- Home Assistant có kết nối Internet tới Ecovacs China cloud.
- Python của add-on là `3.14`; Home Assistant Core không cần cài thư viện cloud
  Ecovacs cho custom integration.

## Cài add-on local

1. Giải nén `ecovacs_cn_addon-repository-v1.2.0.zip`.
2. Chép nguyên thư mục `ecovacs_cn_backend` vào `/addons/`.
3. Mở Add-on Store và chọn **Reload/Check for updates**.
4. Chọn **Ecovacs China Backend** và nhấn **Install/Rebuild**.
5. Kiểm tra trang thông tin phải hiển thị phiên bản `1.2.0`.
6. Khởi động add-on và bật **Show in sidebar** nếu muốn.

Không chép riêng `addon_app` hoặc `protocol_components`. Docker build cần toàn bộ
thư mục `ecovacs_cn_backend`.

## Khởi tạo lần đầu

1. Mở giao diện **Ecovacs Backend** từ thanh bên Home Assistant.
2. Nhập mã khởi tạo một lần do người cài đặt cung cấp.
3. Đặt PIN quản trị 4 số hoặc mật khẩu dài hơn.
4. Lưu API token đầu tiên ở nơi an toàn.

Mã khởi tạo chỉ dùng cho installation chưa được thiết lập và bị vô hiệu sau khi
đặt mật khẩu quản trị. Đây không phải mật khẩu khôi phục hoặc master password.

## Đăng nhập tài khoản Ecovacs

1. Đăng nhập quản trị add-on bằng mật khẩu đã đặt.
2. Chọn **Ecovacs ID**, **Điện thoại + mật khẩu** hoặc **SMS**.
3. Nhập thông tin trực tiếp trong giao diện add-on.
4. Đợi trạng thái chuyển sang `connected`.
5. Nếu cloud/MQTT lỗi tạm thời, dùng nút **Kết nối lại**.

Mật khẩu và mã SMS được xóa khỏi form sau khi gửi. Giao diện/API không có chức
năng đọc lại credential Ecovacs đã lưu.

## Tạo API token cho Home Assistant

1. Trong add-on, mở phần **API cho Home Assistant**.
2. Nhập mật khẩu quản trị và đặt tên máy khách.
3. Tạo token và sao chép ngay; token đầy đủ bắt đầu bằng `ecv1_` và chỉ hiển
   thị một lần.
4. Nên tạo token riêng cho từng máy Home Assistant.

Giá trị 16 ký tự hiển thị trong danh sách token là **ID quản lý** chỉ dùng để
thu hồi, không phải API token. Sau khi nâng cấp lên `1.0.10`, hãy tạo một token
mới và nhập toàn bộ chuỗi bắt đầu bằng `ecv1_` vào custom integration.

Đổi mật khẩu quản trị sẽ thu hồi toàn bộ token cũ và cấp token thay thế. Không
có master password hoặc backdoor cho người cài đặt.

## Cài custom integration

1. Giải nén `ecovacs_cn_hass-v1.2.0.zip`.
2. Chép thư mục `custom_components/ecovacs_cn` vào thư mục config Home
   Assistant.
3. Khởi động lại Home Assistant Core.
4. Vào **Settings → Devices & services → Add integration**.
5. Chọn **Ecovacs China Backend Client**.
6. Custom sẽ tự tìm hostname qua Supervisor. Với add-on local, URL đúng là
   `http://local-ecovacs-cn-backend:4545`; nhập API token vừa tạo và tắt kiểm
   tra TLS vì kết nối nội bộ này dùng HTTP.

Nếu URL đã lưu không còn hoạt động, custom tự hỏi lại Supervisor, thử hostname
mới và cập nhật config entry. Khi Home Assistant khởi động trước add-on,
integration chuyển sang trạng thái retry thay vì dừng setup vĩnh viễn.

Config entry của custom chỉ chứa:

- `addon_url`;
- `api_token` của add-on;
- `verify_ssl`.

Không nhập Ecovacs ID, số điện thoại, mật khẩu hoặc cloud token vào custom
integration.

## Cấu hình hiệu năng

Add-on mặc định ưu tiên tải nhẹ:

- một tiến trình Python;
- một WebSocket nội bộ chỉ phát khi DTO thật sự thay đổi;
- map event MQTT dựng SVG revision mới khoảng mỗi giây khi robot di chuyển;
- custom poll local 60 giây chỉ để dự phòng kết nối;
- map fallback 30 giây và full state fallback 120 giây;
- bản đồ và trạng thái được cache, không tạo tiến trình phụ;
- không dùng `host_network`, không privileged và không mở cổng host mặc định.

Options:

| Option | Mặc định | Phạm vi | Công dụng |
| --- | ---: | ---: | --- |
| `log_level` | `info` | debug–error | Mức log của backend |
| `map_refresh_interval` | `30` giây | 5–60 | Fallback dựng lại map nếu event bị lỡ |
| `state_refresh_interval` | `120` giây | 15–300 | Fallback yêu cầu toàn bộ trạng thái |

Máy yếu nên dùng map `15–20` giây và state `60` giây. Không bật debug lâu dài vì
log nhiều hơn và có thể tăng I/O.

## Bảo mật và dữ liệu

- Mật khẩu quản trị được băm bằng scrypt.
- API token ngẫu nhiên, chỉ lưu HMAC hash và có thể thu hồi.
- Credential Ecovacs được mã hóa AES-256-GCM trong `/data/secrets.enc`.
- Khóa mã hóa chỉ nằm trong `/data` của installation.
- Form quản trị dùng `sessionStorage`, không dùng `localStorage`.
- API giới hạn body 64 KiB và rate-limit các lần xác thực thất bại.
- Audit log không ghi mật khẩu, SMS code hoặc token đầy đủ.
- AppArmor giới hạn filesystem; `/home` và `/root` bị chặn.
- Cổng `4545/tcp` không ánh xạ ra host theo mặc định.

Không thể chống sao chép tuyệt đối nếu một người có toàn quyền filesystem,
Docker image hoặc backup `/data`. Không mở trực tiếp cổng `4545` ra Internet.

## Sao lưu và cập nhật

- Dữ liệu add-on nằm trong `/data` và được Home Assistant backup cùng add-on.
- Khi cập nhật local add-on, thay toàn bộ thư mục `ecovacs_cn_backend`, chọn
  **Reload** rồi **Rebuild**.
- Không chọn xóa dữ liệu nếu muốn giữ tài khoản, mật khẩu quản trị và token.
- Kiểm tra phiên bản trên trang add-on và dòng launcher trong log sau cập nhật.

## Xử lý lỗi

### Docker báo thiếu protocol file

Phải dùng bản `1.2.0` và chép nguyên thư mục add-on. Trong thư mục phải có:

```text
ecovacs_cn_backend/
├── addon_app/
├── protocol_components/ecovacs_cn/
├── Dockerfile
├── config.yaml
└── run.sh
```

### Log không có launcher đúng phiên bản

Home Assistant vẫn đang dùng image cache cũ. Dừng add-on, Reload Add-on Store,
kiểm tra version rồi chọn **Rebuild**.

### `ModuleNotFoundError: addon_app`

Kiểm tra đang dùng `1.2.0`. Bản này cài package trực tiếp vào Python
`site-packages`; Docker build sẽ tự import-test package và không còn phụ thuộc
`PYTHONPATH` hoặc quyền đọc `/app`.

### Add-on kết nối nhưng không có robot

- Kiểm tra robot xuất hiện trong Ecovacs Home vùng Trung Quốc.
- Kiểm tra phương thức đăng nhập và mã vùng.
- Chọn **Kết nối lại** và xem log cloud/MQTT.
- Không chia sẻ log debug trước khi tự kiểm tra dữ liệu nhạy cảm.

### Custom integration không kết nối

- Kiểm tra add-on đang chạy và `/health` hoạt động.
- Add-on local dùng `http://local-ecovacs-cn-backend:4545`; không dùng URL
  Ingress hoặc `localhost`.
- Reload integration một lần để bản `1.2.0` tự rediscover và lưu hostname mới.
- Tạo token mới trong add-on và sao chép toàn bộ chuỗi bắt đầu bằng `ecv1_`.
- Không nhập ID quản lý 16 ký tự hiển thị trong danh sách token.
- Không sử dụng Ingress URL làm API URL cho custom integration.

## Gói phát hành

- `ecovacs_cn_addon-repository-v1.2.0.zip`: add-on repository/local build.
- `ecovacs_cn_hass-v1.2.0.zip`: custom integration mỏng.
- `ecovacs_cn_hass-full-archive-v1.2.0.zip`: add-on và custom trong một gói.
- `SHA256SUMS-v1.2.0.txt`: checksum kiểm tra file phát hành.

Xem thêm hướng dẫn vận hành ngắn trong `DOCS.md` và lịch sử thay đổi trong
`CHANGELOG.md`.
