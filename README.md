# Phòng tự học thông minh: giám sát chất lượng không khí và phát hiện đồ bỏ quên

> IoT Apps Dev · ESP32 · MQTT · Node-RED Dashboard · PlatformIO

- **Sinh viên thực hiện:** Nguyễn Doãn Bình – 23001582
- **Giảng viên hướng dẫn:** Nguyễn Anh Tuấn

---

## 1. Giới thiệu

Đề tài xây dựng hệ thống IoT trên mô hình phòng tự học thu nhỏ gồm 4 bàn, thực hiện đồng thời ba chức năng:

1. **Giám sát môi trường:** đo nhiệt độ, độ ẩm và chỉ số chất lượng không khí; hiển thị luân phiên cùng ngày giờ trên màn hình OLED; tự động bật quạt thông gió khi chất lượng không khí vượt ngưỡng cho phép.
2. **Phát hiện đồ bỏ quên:** mỗi bàn được trang bị hai cảm biến độc lập xác định "có người" và "có đồ". Khi bàn có đồ nhưng không có người vượt quá thời gian chờ cấu hình, hệ thống chuyển sang trạng thái **"Quên đồ"** và hiển thị cảnh báo trên bản đồ phòng tại dashboard.
3. **Thông báo qua email cho sinh viên:** mỗi bàn được dán một mã QR. Sinh viên ngồi vào bàn quét mã, nhập mã sinh viên; hệ thống suy ra địa chỉ email `<mã sinh viên>@hus.edu.vn` và tự động gửi thư nhắc khi bàn của sinh viên đó chuyển sang trạng thái "Quên đồ".

Dữ liệu cảm biến được trao đổi qua giao thức **MQTT** (broker công khai) và được giám sát, điều khiển từ xa bằng **Node-RED Dashboard** trên máy tính và điện thoại. Việc đăng ký bàn và gửi email do **Node-RED** đảm nhiệm, phần mềm trên ESP32 không phải thay đổi.

## 2. Đặt vấn đề

**Chất lượng không khí trong không gian kín.** Phòng tự học thường đóng kín, sử dụng điều hòa và có mật độ người cao, khiến nồng độ khí CO₂ và các chất gây ô nhiễm tăng nhanh, làm giảm sự tỉnh táo và hiệu quả học tập. Người sử dụng thường chỉ nhận biết khi đã xuất hiện triệu chứng mệt mỏi.

**Đồ bỏ quên.** Người học thường để lại sách vở, tư trang trên bàn khi rời phòng. Đồ để lại không có người trông coi trong thời gian dài có nguy cơ thất lạc hoặc mất cắp, đồng thời người quản lý khó phát hiện kịp thời do không có thông tin khách quan về tình trạng từng bàn.

**Khó liên lạc với chủ đồ.** Ngay cả khi hệ thống phát hiện được bàn bị bỏ quên đồ, cảnh báo trên dashboard chỉ đến được người quản lý, còn người cần được nhắc là sinh viên đã rời đi thì không nhận được thông tin. Phòng tự học không có sổ đăng ký chỗ ngồi nên không biết ai đang ngồi ở bàn nào.

**Hướng giải quyết.** Sử dụng cảm biến để thu thập hai nhóm thông tin khách quan: điều kiện môi trường của phòng và tình trạng từng bàn (có người, có đồ). Thông tin được cung cấp từ xa cho người quản lý và sinh viên, trong khi việc thông gió được thiết bị tự động xử lý. Để liên lạc với chủ đồ, mỗi bàn có mã QR dẫn đến biểu mẫu nhập mã sinh viên, từ đó hệ thống biết bàn đang thuộc về ai và gửi email nhắc khi cần. Cách này tận dụng quy ước email của trường (`<mã sinh viên>@hus.edu.vn`) nên sinh viên không phải tạo tài khoản hay cài thêm ứng dụng.

## 3. Mục tiêu và phạm vi

**Mục tiêu**
- Đo và hiển thị nhiệt độ, độ ẩm, chất lượng không khí theo thời gian thực.
- Điều khiển quạt theo ngưỡng, hỗ trợ hai chế độ Auto và Manual.
- Xác định trạng thái từng bàn (Trống, Có người, Tạm vắng, Quên đồ) và hiển thị trên bản đồ phòng.
- Điều khiển và cấu hình thông số (ngưỡng không khí, thời gian chờ) từ xa qua Node-RED.
- Cho phép sinh viên đăng ký chỗ ngồi bằng cách quét mã QR tại bàn và nhập mã sinh viên.
- Tự động gửi email nhắc đến sinh viên khi bàn của họ chuyển sang trạng thái "Quên đồ".

**Phạm vi**
- Mô hình thu nhỏ gồm 4 bàn; cấu trúc topic MQTT theo từng bàn và mã QR theo từng bàn cho phép mở rộng số lượng bàn.
- Chỉ số chất lượng không khí mang tính tương đối, không thay thế thiết bị đo chuẩn.
- Hệ thống đánh giá dựa trên thời gian đồ bị để lại mà không có người; các trường hợp người dùng chủ ý để đồ tạm thời được xử lý bằng cách cấu hình thời gian chờ phù hợp.
- Mã sinh viên chỉ được kiểm tra về định dạng, chưa xác thực chủ sở hữu (xem mục 6.4 về các biện pháp hạn chế lạm dụng).
- Điện thoại của sinh viên cần truy cập được trang đăng ký do Node-RED cung cấp: dùng chung mạng WiFi với máy chạy Node-RED khi trình diễn, hoặc công khai qua đường hầm (tunnel) khi triển khai thực tế.

## 4. Kiến trúc hệ thống

```mermaid
flowchart LR
    subgraph ROOM["Mô hình phòng học thu nhỏ"]
        DESKS["4 bàn × 2 vị trí (ngồi, để đồ)<br/>= 8 công tắc hành trình"]
        LEDS["LED trạng thái từng bàn"]
        QR["Mã QR dán tại mỗi bàn"]
    end
    subgraph ESP["ESP32"]
        MCU["Đọc cảm biến, máy trạng thái bàn,<br/>điều khiển quạt"]
    end
    DHT["DHT22<br/>nhiệt độ, độ ẩm"] --> MCU
    MQ["MQ-135<br/>chất lượng không khí"] --> MCU
    DESKS --> MCU
    MCU --> LEDS
    MCU --> OLED["OLED SSD1306<br/>giờ, ngày, thông số"]
    MCU --> FAN["Quạt thông gió<br/>(qua MOSFET)"]
    NTP["NTP server"] --> MCU
    MCU <-->|WiFi| BROKER[("MQTT Broker")]
    BROKER <--> NR["Node-RED + Dashboard"]
    NR --> UI["Máy tính / điện thoại"]
    QR -.->|sinh viên quét| PHONE["Điện thoại sinh viên"]
    PHONE <-->|"HTTP: biểu mẫu nhập mã SV"| NR
    NR -->|SMTP| MAIL["Email sinh viên<br/>mã SV@hus.edu.vn"]
```

## 5. Phần cứng

| Linh kiện | Vai trò |
|---|---|
| ESP32 DevKit | Xử lý, kết nối WiFi, MQTT |
| DHT22 | Đo nhiệt độ, độ ẩm |
| MQ-135 | Đo chỉ số chất lượng không khí tương đối |
| OLED SSD1306 128×64 (I2C) | Hiển thị giờ, ngày, thông số |
| Quạt 5V + MOSFET + diode | Thông gió |
| 8 công tắc hành trình | Mỗi bàn: 1 vị trí ngồi, 1 vị trí để đồ |
| 4 LED (hoặc LED RGB/WS2812B) | Hiển thị trạng thái từng bàn |
| 4 mã QR in giấy (có thể ép plastic) | Dán tại mỗi bàn, dẫn đến biểu mẫu đăng ký của bàn đó |

**Chi phí dự kiến:** khoảng 300–400 nghìn đồng (chưa gồm ESP32). Mã QR chỉ tốn chi phí in ấn không đáng kể.

**Quy ước trong mô hình thu nhỏ**
- "Người" được mô phỏng bằng nhân vật mô hình có thêm tải trọng để đủ lực nhấn công tắc tại vị trí ngồi.
- "Đồ" được mô phỏng bằng vật nhỏ (khối giấy cứng, sách mô hình) đặt tại vị trí để đồ.
- Khi triển khai thực tế, thay bằng cảm biến áp lực ở ghế và cảm biến phù hợp tại vị trí để đồ; phần mềm giữ nguyên.
- Mã QR mã hóa một đường dẫn có dạng `http://<địa_chỉ_node_red>:1880/checkin?desk=<id>`, trong đó `<id>` là số thứ tự bàn (1–4).

## 6. Nguyên lý hoạt động

### 6.1. Giám sát môi trường, điều khiển quạt và hiển thị

- DHT22 và MQ-135 được đọc theo chu kỳ 2–5 giây, giá trị được làm mượt bằng trung bình trượt.
- Giá trị MQ-135 được quy đổi thành chỉ số 0–100 và phân loại thành ba mức: Tốt, Trung bình, Kém.
- **Điều khiển quạt có trễ (hysteresis):** quạt bật khi chỉ số vượt ngưỡng trên và chỉ tắt khi xuống dưới ngưỡng dưới, tránh đóng ngắt liên tục. Các ngưỡng ban đầu là giá trị dự kiến, sẽ được hiệu chỉnh qua thực nghiệm.
- **OLED hiển thị luân phiên** bằng `millis()` (không dùng `delay()`), mỗi trang 4–5 giây:
  1. Giờ và ngày (đồng bộ NTP, múi giờ UTC+7)
  2. Nhiệt độ và độ ẩm
  3. Chất lượng không khí và trạng thái quạt
  4. Tóm tắt trạng thái các bàn

### 6.2. Máy trạng thái của bàn

Mỗi bàn có hai đầu vào số: **có người** (công tắc tại vị trí ngồi) và **có đồ** (công tắc tại vị trí để đồ). Tín hiệu được lọc nhiễu tiếp điểm (debounce) và có độ trễ ngắn để việc đứng dậy trong thời gian ngắn không bị coi là rời đi.

| Trạng thái | Điều kiện | Màu LED |
|---|---|---|
| Trống | Không đồ, không người | Xanh lá |
| Có người | Có người ngồi | Xanh dương |
| Tạm vắng | Có đồ, không người, chưa quá thời gian chờ | Vàng |
| Quên đồ | Có đồ, không người, đã quá thời gian chờ | Đỏ |

```mermaid
stateDiagram-v2
    [*] --> TRONG
    TRONG --> CO_NGUOI: có người ngồi
    TRONG --> TAM_VANG: có đồ, không người
    CO_NGUOI --> TRONG: người đi, không đồ
    CO_NGUOI --> TAM_VANG: người đi, đồ còn
    TAM_VANG --> CO_NGUOI: người quay lại
    TAM_VANG --> QUEN_DO: quá thời gian chờ
    TAM_VANG --> TRONG: lấy đồ đi
    QUEN_DO --> CO_NGUOI: người quay lại
    QUEN_DO --> TRONG: lấy đồ đi
```

**Thời gian chờ** được cấu hình từ Node-RED: 30 giây khi trình diễn, khoảng 20 phút khi sử dụng thực tế.

### 6.3. Đăng ký bàn bằng mã QR và gửi email nhắc quên đồ

**Đăng ký bàn (check-in)**

1. Sinh viên ngồi vào bàn và dùng điện thoại quét mã QR dán tại bàn.
2. Điện thoại mở trang `/checkin?desk=<id>` do Node-RED cung cấp, hiển thị biểu mẫu nhập mã sinh viên và cho biết đang đăng ký cho bàn số mấy.
3. Sinh viên nhập mã sinh viên (ví dụ `23001582`) và gửi.
4. Node-RED kiểm tra dữ liệu (xem bên dưới). Nếu hợp lệ, hệ thống lưu liên kết **bàn → mã sinh viên → email** (`23001582@hus.edu.vn`) và trả về trang xác nhận.

**Kiểm tra khi đăng ký**
- Mã sinh viên đúng định dạng (8 chữ số, biểu thức chính quy `^\d{8}$`; có thể điều chỉnh theo quy định của trường).
- Mã bàn hợp lệ (1–4).
- Bàn đang ở trạng thái **Có người** tại thời điểm đăng ký, để hạn chế việc đăng ký từ xa hoặc đăng ký khi không ngồi tại bàn.
- Giới hạn tần suất đăng ký theo địa chỉ IP và theo bàn.

**Vòng đời liên kết bàn – sinh viên**
- Được tạo khi sinh viên đăng ký thành công; nếu có người khác đăng ký lại cho cùng bàn khi bàn đang "Có người", liên kết cũ bị thay thế.
- Được giữ nguyên khi bàn chuyển qua các trạng thái Có người, Tạm vắng, Quên đồ, vì đây chính là lúc cần biết chủ đồ.
- Bị xóa khi bàn trở về **Trống** (người đã lấy đồ và rời đi) hoặc khi quá thời hạn tối đa (ví dụ 12 giờ), để không gửi nhầm cho người ngồi trước.

**Gửi email nhắc quên đồ**

```mermaid
sequenceDiagram
    participant SV as Sinh viên
    participant NR as Node-RED
    participant ESP as ESP32
    participant MAIL as Máy chủ SMTP
    SV->>NR: Quét QR, mở /checkin?desk=2
    NR-->>SV: Biểu mẫu nhập mã sinh viên
    SV->>NR: Gửi mã sinh viên
    NR->>NR: Kiểm tra định dạng, bàn đang có người
    NR-->>SV: Xác nhận đăng ký
    ESP->>NR: MQTT desk/2/state = FORGOTTEN
    NR->>NR: Tra liên kết bàn 2, kiểm tra chưa gửi mail
    NR->>MAIL: Gửi thư tới mã SV@hus.edu.vn
    MAIL-->>SV: Thư nhắc quên đồ
```

- Node-RED theo dõi topic `desk/<id>/state`. Khi trạng thái chuyển sang `FORGOTTEN` (Quên đồ), Node-RED tra liên kết của bàn đó.
- Nếu có liên kết, Node-RED gửi email nhắc qua node `email` (SMTP) và ghi vào nhật ký sự kiện. Nếu chưa có ai đăng ký, chỉ ghi nhật ký và hiển thị cảnh báo trên dashboard.
- **Mỗi lần quên đồ chỉ gửi một email.** Cờ "đã gửi" được đặt lại khi bàn rời khỏi trạng thái Quên đồ (người quay lại hoặc lấy đồ), tránh gửi lặp liên tục.
- Nội dung thư (dự kiến):

> **Tiêu đề:** [Phòng tự học] Bạn có đồ để quên tại bàn 2
>
> Chào bạn (MSSV 23001582),
> Hệ thống ghi nhận bàn số 2 có đồ nhưng không có người từ 14:32 (quá 20 phút). Vui lòng quay lại lấy đồ hoặc liên hệ người quản lý phòng nếu bạn đã rời đi.
> *Đây là thư tự động, vui lòng không trả lời.*

### 6.4. Bảo mật và quyền riêng tư

- **Xác thực chủ sở hữu:** vì chỉ nhập mã sinh viên nên có thể có người nhập mã của người khác. Các biện pháp hạn chế: chỉ nhận đăng ký khi bàn đang có người, giới hạn tần suất, mỗi lần quên đồ chỉ gửi một thư nên nguy cơ gây phiền (spam) thấp. Hướng nâng cấp: gửi mã OTP đến email trường để xác nhận trước khi liên kết.
- **Dữ liệu cá nhân tối thiểu:** chỉ lưu mã sinh viên (từ đó suy ra email) và chỉ trong thời gian bàn đang được sử dụng; xóa liên kết khi bàn trống hoặc hết hạn.
- **Thông tin đăng nhập SMTP** (tài khoản, mật khẩu ứng dụng) lưu trong kho thông tin xác thực của Node-RED, không đưa vào mã nguồn hay kho Git.
- **Dashboard** chỉ hiển thị mã sinh viên ở dạng che một phần (ví dụ `2300xxxx`) nếu dashboard được xem công khai.
- **Truy cập trang đăng ký:** trang `/checkin` có thể truy cập qua mạng nội bộ; khi công khai ra Internet nên đặt sau đường hầm có HTTPS.

## 7. Giao tiếp MQTT

Hệ thống sử dụng broker công khai (HiveMQ hoặc EMQX). Do broker công khai không giới hạn truy cập, tất cả topic dùng tiền tố riêng có dạng `<tiền_tố>/studyroom/...`.

| Topic | Hướng | Nội dung |
|---|---|---|
| `.../room/env` | ESP32 → Node-RED | JSON: nhiệt độ, độ ẩm, chỉ số không khí |
| `.../room/fan/state` | ESP32 → Node-RED | Trạng thái và chế độ của quạt |
| `.../room/fan/cmd` | Node-RED → ESP32 | `auto` / `on` / `off` |
| `.../desk/<id>/state` | ESP32 → Node-RED | JSON: có đồ, có người, trạng thái, thời gian chờ |
| `.../config/timeout` | Node-RED → ESP32 | Thời gian chờ (giây) |
| `.../config/air_threshold` | Node-RED → ESP32 | Ngưỡng bật/tắt quạt |
| `.../status` | ESP32 → Node-RED | `online` / `offline` (Last Will and Testament) |

Ví dụ bản tin (dự kiến):

```json
// .../room/env
{ "temp": 29.4, "hum": 68, "air": 52, "air_level": "medium" }

// .../desk/2/state
{ "item": true, "person": false, "state": "AWAY", "wait_s": 840 }

// .../desk/2/state (khi quá thời gian chờ, kích hoạt gửi email)
{ "item": true, "person": false, "state": "FORGOTTEN", "wait_s": 1200 }
```

Chức năng đăng ký bàn và gửi email **không thêm topic MQTT mới**: liên kết bàn – sinh viên được quản lý hoàn toàn trong Node-RED, nên không đưa thông tin cá nhân lên broker công khai.

**Giao diện HTTP do Node-RED cung cấp (không qua MQTT)**

| Đường dẫn | Phương thức | Chức năng |
|---|---|---|
| `/checkin?desk=<id>` | GET | Trả về biểu mẫu nhập mã sinh viên cho bàn `<id>` |
| `/checkin` | POST | Nhận mã sinh viên và mã bàn, kiểm tra, lưu liên kết bàn – sinh viên |

## 8. Dashboard Node-RED

- **Bản đồ phòng:** lưới 4 bàn xây dựng bằng `ui_template` (HTML/CSS), đổi màu theo bốn trạng thái ở mục 6.2; mỗi bàn hiển thị thêm mã sinh viên đã đăng ký (che một phần) hoặc "Chưa đăng ký".
- **Môi trường:** biểu đồ nhiệt độ, độ ẩm và chỉ số không khí theo thời gian.
- **Điều khiển:** công tắc Auto/Manual cho quạt; thanh trượt chỉnh ngưỡng không khí và thời gian chờ.
- **Nhật ký sự kiện:** ghi nhận các chuyển trạng thái (ví dụ: "Bàn 2 chuyển sang Quên đồ lúc 14:32"), cùng các sự kiện đăng ký bàn và gửi email (ví dụ: "Đã gửi email nhắc quên đồ cho 2300xxxx, bàn 2, lúc 14:32" hoặc "Bàn 3 quên đồ nhưng chưa có người đăng ký").
- **Trạng thái thiết bị:** online/offline.
- **Điều khiển thông báo:** bật/tắt chức năng gửi email, nút gửi thư thử để kiểm tra cấu hình SMTP.
- *Mở rộng:* thống kê số lần phát hiện quên đồ theo bàn và theo thời gian; thông báo qua Telegram; xác nhận mã sinh viên bằng OTP; gửi thư nhắc lần hai nếu đồ vẫn chưa được lấy.

## 9. Đóng góp và tính ứng dụng

- Tích hợp giám sát môi trường và phát hiện đồ bỏ quên trong một hệ thống, đáp ứng nhu cầu thực tế của phòng tự học và thư viện.
- Dùng hai cảm biến độc lập (người, đồ) thay vì cảm biến chuyển động đơn thuần, tránh báo nhầm khi người học ngồi yên.
- Gắn danh tính người dùng với bàn bằng mã QR và mã sinh viên, không cần tài khoản hay ứng dụng riêng, tận dụng quy ước email của trường để nhắc đúng người.
- Thông báo trực tiếp đến sinh viên thay vì chỉ cảnh báo cho người quản lý, giúp chủ đồ kịp quay lại lấy.
- Tăng độ ổn định hệ thống bằng điều khiển quạt có trễ và lọc nhiễu tiếp điểm.
- Trao đổi dữ liệu hai chiều qua MQTT: dữ liệu cảm biến lên dashboard, lệnh điều khiển và cấu hình xuống thiết bị.
- Tách bạch dữ liệu cá nhân (xử lý trong Node-RED) khỏi dữ liệu cảm biến (truyền qua broker công khai).

## 10. Công cụ và môi trường phát triển

- **Lập trình:** PlatformIO, Arduino framework (ESP32)
- **Thư viện:** PubSubClient (MQTT), DHT sensor library, Adafruit SSD1306 và GFX, ArduinoJson
- **Mô phỏng:** Wokwi
- **Giám sát và điều khiển:** Node-RED, node-red-dashboard
- **Đăng ký bàn và gửi email:** node `http in` / `http response` (trang đăng ký), `node-red-node-email` (gửi thư qua SMTP), context lưu trữ của Node-RED (liên kết bàn – sinh viên)
- **Tạo mã QR:** công cụ tạo QR bất kỳ (ví dụ thư viện `qrcode` hoặc trang web tạo QR) từ đường dẫn `/checkin?desk=<id>`
- **Đồng bộ thời gian:** NTP, múi giờ UTC+7

## 11. Kế hoạch thực hiện

| Giai đoạn | Nội dung |
|---|---|
| 1 | Mô phỏng trên Wokwi: DHT22, MQ-135 (hoặc biến trở), OLED, quạt, nút nhấn thay công tắc |
| 2 | Kết nối WiFi, MQTT, NTP; OLED luân phiên; quạt Auto/Manual |
| 3 | Máy trạng thái 4 bàn, debounce, LED trạng thái, gửi dữ liệu MQTT |
| 4 | Node-RED: bản đồ phòng, biểu đồ, điều khiển, nhật ký |
| 5 | Node-RED: trang đăng ký `/checkin`, lưu liên kết bàn – sinh viên, gửi email khi Quên đồ; tạo và in mã QR |
| 6 | Lắp mô hình thực, hiệu chỉnh ngưỡng, kiểm thử toàn hệ thống (kể cả đăng ký bằng điện thoại và nhận email), quay video trình diễn |

## 12. Hình ảnh linh kiện

<table>
  <tr>
    <td align="center"><img src="images/esp32c6.png" width="200"><br><sub><b>Hình 1.</b> Vi điều khiển ESP32</sub></td>
    <td align="center"><img src="images/dht22.webp" width="200"><br><sub><b>Hình 2.</b> Cảm biến DHT22</sub></td>
    <td align="center"><img src="images/mq135.webp" width="200"><br><sub><b>Hình 3.</b> Cảm biến MQ-135</sub></td>
    <td align="center"><img src="images/oled.jpg" width="200"><br><sub><b>Hình 4.</b> Màn hình OLED SSD1306</sub></td>
  </tr>
  <tr>
    <td align="center"><img src="images/fan.webp" width="200"><br><sub><b>Hình 5.</b> Quạt thông gió</sub></td>
    <td align="center"><img src="images/switch.webp" width="200"><br><sub><b>Hình 6.</b> Công tắc hành trình</sub></td>
    <td align="center"><img src="images/led.webp" width="200"><br><sub><b>Hình 7.</b> LED trạng thái</sub></td>
    <td></td>
  </tr>
</table>

## 13. Tài liệu tham khảo

1. Random Nerd Tutorials, [ESP32 MQTT Publish/Subscribe with Node-RED](https://randomnerdtutorials.com/esp32-mqtt-publish-subscribe-arduino-ide/)
2. Hackaday.io, [Project Aura: Open-Source ESP32 Air Quality](https://hackaday.io/project/204993-project-aura-open-source-esp32-air-quality)
3. [Wokwi – ESP32 Simulator](https://wokwi.com)
4. [Node-RED](https://nodered.org)
5. [node-red-node-email](https://flows.nodered.org/node/node-red-node-email) – node gửi và nhận email cho Node-RED
