# Ecovacs China Backend cho Home Assistant

Ecovacs China Backend `1.2.5` là add-on điều khiển robot Ecovacs tài khoản Trung
Quốc và đưa entity vào Home Assistant trực tiếp bằng MQTT Discovery. Không cần
cài custom component hoặc nhập cloud credential vào Home Assistant Core.

Đây không phải phần mềm chính thức của Ecovacs.

## Kiến trúc

```text
Ecovacs China cloud + MQTT
            │
            ▼
Ecovacs Backend add-on :4545
  ├─ đăng nhập và lưu credential mã hóa
  ├─ nhận trạng thái robot từ Ecovacs MQTT
  ├─ dựng/cache bản đồ SVG
  ├─ kiểm tra và thực thi lệnh
  └─ cache từng MQTT topic, chỉ publish khi giá trị đổi
            │ MQTT Discovery + retained state
            ▼
Home Assistant MQTT integration
  ├─ vacuum / lawn_mower
  ├─ sensor / setting
  ├─ image bản đồ
  └─ button/action
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

Tùy capability robot thực tế cung cấp, add-on có thể tạo qua MQTT Discovery:

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
- Cache bản đồ trong add-on; Home Assistant chỉ nhận SVG đã dựng xong.
- Theo dõi revision và tên bản đồ.
- Khi robot di chuyển, event map từ Ecovacs dựng revision SVG mới. MQTT bridge
  chỉ phát bản mới nhất theo giới hạn `mqtt_map_min_interval`.
- Chu kỳ map 30 giây chỉ là fallback khi event bị bỏ lỡ, không phải độ trễ
  realtime bình thường.

### Realtime và tải Home Assistant

- Mỗi sensor/control có topic riêng và cache payload riêng. Giá trị giống hệt
  lần trước không được publish lại.
- Discovery, state và availability được retain để Home Assistant phục hồi nhanh
  sau khi Core hoặc broker khởi động lại.
- Map dùng topic riêng nên robot di chuyển không ép toàn bộ sensor/control cập
  nhật; kích thước và tần suất map đều có giới hạn.
- Lỗi broker, DNS hoặc credential MQTT chỉ làm bridge reconnect theo backoff;
  không được ném ngược vào web server, watchdog hoặc Home Assistant Core.
- Command MQTT được xử lý tuần tự, giới hạn 4 KiB, bỏ qua retained command và
  tiếp tục đi qua allow-list/type validation của backend.
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

- DEEBOT X9 PRO OMNI hỗ trợ trực tiếp class quốc tế `ilt3k8` và hai class nội
  địa `lwmdoj`, `0jv4ti`; class X9 mới có model/tên sản phẩm khớp sẽ dùng profile
  `ilt3k8` đã xác minh.
- X9 PRO có bản đồ/phòng/vị trí, dọn khu vực, bốn mức lực hút, ba work mode, mức
  nước tùy chỉnh `0–50`, tần suất tự giặt giẻ `0–60`, auto-empty, khóa trẻ em,
  firmware, chế độ hiệu suất và các cảm biến tuổi thọ vật tư.
- Nút trạm X9 hiện chỉ gồm hút rác theo capability upstream. Add-on không tự
  quảng bá giặt giẻ, sấy giẻ hoặc vệ sinh đế khi lệnh chưa được xác minh.
- DEEBOT T10 OMNI được nhận diện tự động từ định danh sản phẩm bất biến của
  Ecovacs, gồm các biến thể OMNI, OMNIWHITE và CURIEOMNI đã xác nhận trong APK.
- Class `hxm494` của T10 OMNI nội địa được đăng ký trực tiếp trong
  `deebot-client`, không còn cảnh báo class chưa được nhận diện.
- T10 OMNI dùng profile DT10/X1 OMNI bảo thủ: giữ lệnh dọn legacy để tránh ép
  thiết bị sang giao thức live V2, không gọi `getWorkMode` và không tiếp tục gọi
  `getMapSubSet` sau map metadata vì firmware `hxm494` không phản hồi hai lệnh
  này; capability trạm OMNI và bản đồ nền vẫn được giữ.
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
- Một MQTT broker có thể truy cập từ container add-on; có thể dùng broker bên
  ngoài hoặc MQTT service do Supervisor cung cấp.
- Kiến trúc `amd64` hoặc `aarch64`.
- Robot đã được thêm vào Ecovacs Home vùng Trung Quốc.
- Home Assistant có kết nối Internet tới Ecovacs China cloud.
- Python của add-on là `3.14`; Home Assistant Core không cần cài thư viện cloud
  Ecovacs hoặc custom component.

## Cài add-on local

1. Giải nén `ecovacs_cn_addon-repository-v1.2.5.zip`.
2. Chép nguyên thư mục `ecovacs_cn_backend` vào `/addons/`.
3. Mở Add-on Store và chọn **Reload/Check for updates**.
4. Chọn **Ecovacs China Backend** và nhấn **Install/Rebuild**.
5. Kiểm tra trang thông tin phải hiển thị phiên bản `1.2.5`.
6. Khởi động add-on và bật **Show in sidebar** nếu muốn.

Không chép riêng `addon_app` hoặc `protocol_components`. Docker build cần toàn bộ
thư mục `ecovacs_cn_backend`.

## Khởi tạo lần đầu

1. Mở giao diện **Ecovacs Backend** từ thanh bên Home Assistant.
2. Nhập mã khởi tạo một lần do người cài đặt cung cấp.
3. Đặt PIN quản trị 4 số hoặc mật khẩu dài hơn.
4. Kiểm tra ô **MQTT Home Assistant** chuyển sang **Đã kết nối**.

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

## Kết nối Home Assistant bằng MQTT

1. Mở giao diện **Ingress** của add-on.
2. Trong thẻ **Kết nối MQTT**, điền địa chỉ, cổng, tài khoản và mật khẩu broker.
3. Bật **Dùng TLS** nếu broker yêu cầu TLS, thường là cổng `8883`.
4. Bấm **Lưu và kết nối MQTT**; bridge sẽ kết nối lại ngay.
5. Nếu để trống địa chỉ, add-on thử MQTT service `mqtt:want` của Supervisor.
6. Sau khi đăng nhập Ecovacs, device/entity tự xuất hiện qua MQTT Discovery.

Không cần API token để Home Assistant nhận entity. Nếu trước đây đã cài
`custom_components/ecovacs_cn`, hãy xóa integration cũ, xóa thư mục đó rồi
restart Home Assistant để tránh entity trùng.

## Cấu hình hiệu năng

Add-on mặc định ưu tiên tải nhẹ:

- một tiến trình Python;
- một MQTT bridge nền tách lỗi khỏi web server và cloud controller;
- mỗi topic chỉ publish khi payload thực sự thay đổi;
- availability kép cho trạng thái bridge và từng robot;
- reconnect broker theo exponential backoff tối đa 300 giây;
- map chỉ publish bản mới nhất theo interval và giới hạn byte;
- map fallback 30 giây và full state fallback 120 giây;
- bản đồ và trạng thái được cache, không tạo tiến trình phụ;
- không dùng `host_network`, không privileged và không mở cổng host mặc định.

Options:

| Option | Mặc định | Phạm vi | Công dụng |
| --- | ---: | ---: | --- |
| `log_level` | `info` | debug–error | Mức log của backend |
| `map_refresh_interval` | `30` giây | 5–60 | Fallback dựng lại map nếu event bị lỡ |
| `state_refresh_interval` | `120` giây | 15–300 | Fallback yêu cầu toàn bộ trạng thái |
| `mqtt_enabled` | `true` | true/false | Bật MQTT Discovery bridge |
| `mqtt_map_enabled` | `true` | true/false | Publish ảnh SVG qua MQTT Image |
| `mqtt_map_min_interval` | `5` giây | 1–60 | Khoảng cách tối thiểu giữa hai map publish |
| `mqtt_map_max_bytes` | `2000000` | 64000–10000000 | Bỏ qua map quá lớn để bảo vệ broker/Core |

Máy yếu nên dùng map `15–20` giây và state `60` giây. Không bật debug lâu dài vì
log nhiều hơn và có thể tăng I/O.

## Bảo mật và dữ liệu

- Mật khẩu quản trị được băm bằng scrypt.
- Phiên bearer quản trị ngẫu nhiên, chỉ lưu HMAC hash và có thể thu hồi.
- Credential Ecovacs được mã hóa AES-256-GCM trong `/data/secrets.enc`.
- MQTT broker credential nhập trong Ingress được mã hóa riêng trong `/data/mqtt`.
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

Phải dùng bản `1.2.5` và chép nguyên thư mục add-on. Trong thư mục phải có:

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

Kiểm tra đang dùng `1.2.5`. Bản này cài package trực tiếp vào Python
`site-packages`; Docker build sẽ tự import-test package và không còn phụ thuộc
`PYTHONPATH` hoặc quyền đọc `/app`.

### Add-on kết nối nhưng không có robot

- Kiểm tra robot xuất hiện trong Ecovacs Home vùng Trung Quốc.
- Kiểm tra phương thức đăng nhập và mã vùng.
- Chọn **Kết nối lại** và xem log cloud/MQTT.
- Không chia sẻ log debug trước khi tự kiểm tra dữ liệu nhạy cảm.

### MQTT không tạo entity

- Kiểm tra broker và MQTT integration của Home Assistant đang hoạt động.
- Kiểm tra ô **MQTT Home Assistant** trong giao diện add-on.
- Xem log có `MQTT service is unavailable` hoặc lỗi xác thực broker hay không.
- Restart add-on sau khi broker được cài để Supervisor cấp service credential.
- Không cấu hình retained command thủ công; add-on chủ động bỏ qua loại message
  này để tránh lặp lại lệnh cũ sau reconnect.

## Gói phát hành

- Mọi bản build mới được lưu trong thư mục `ket_qua` ở root workspace.
- `ecovacs_cn_addon-repository-v1.2.5.zip`: add-on repository/local build.
- `SHA256SUMS-v1.2.5.txt`: checksum của archive add-on.

Xem thêm hướng dẫn vận hành ngắn trong `DOCS.md` và lịch sử thay đổi trong
`CHANGELOG.md`.
