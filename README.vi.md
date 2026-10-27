# SmartSolar MPPT cho Home Assistant

<!-- readme-axi type=ha -->

[![GitHub release](https://img.shields.io/github/release/ngoviet/smartsolar_ha.svg)](https://github.com/ngoviet/smartsolar_ha/releases)
[![CI](https://github.com/ngoviet/smartsolar_ha/actions/workflows/ci.yml/badge.svg)](https://github.com/ngoviet/smartsolar_ha/actions/workflows/ci.yml)
[![HACS Validation](https://github.com/ngoviet/smartsolar_ha/actions/workflows/hacs-validation.yml/badge.svg)](https://github.com/ngoviet/smartsolar_ha/actions/workflows/hacs-validation.yml)
[![Home Assistant](https://img.shields.io/badge/Home%20Assistant-2026.9%2B-41BDF5)](https://www.home-assistant.io)
[![Python](https://img.shields.io/badge/python-3.14.2%2B-blue)](https://www.python.org)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

[English](README.md) · **Tiếng Việt**

Integration tuỳ chỉnh cho **bộ điều khiển sạc năng lượng mặt trời SmartSolar MPPT**. Integration
đọc số liệu trực tiếp từ cloud API của SmartSolar và từ MQTT broker của SmartSolar, đưa chúng
thành các sensor trong Home Assistant, và giữ cho giá trị luôn được cập nhật từ nguồn nào đang
hoạt động.

[![Mở Home Assistant của bạn và thêm kho này vào HACS.](https://my.home-assistant.io/badges/hacs_repository.svg)](https://my.home-assistant.io/redirect/hacs_repository/?owner=ngoviet&repository=smartsolar_ha&category=integration)

> 🇻🇳 Dự án do một người Việt tự xây dựng và duy trì — nếu nó giúp ích cho bạn, hãy
> [ủng hộ dự án](#ủng-hộ-dự-án).

## Mục lục

- [Tính năng](#tính-năng)
- [Thiết bị hỗ trợ](#thiết-bị-hỗ-trợ)
- [Yêu cầu](#yêu-cầu)
- [Cài đặt](#cài-đặt)
- [Cấu hình](#cấu-hình)
- [Thực thể](#thực-thể)
- [Dịch vụ](#dịch-vụ)
- [Xử lý sự cố](#xử-lý-sự-cố)
- [Cách hoạt động](#cách-hoạt-động)
- [Phát triển](#phát-triển)

## Tính năng

| | |
|---|---|
| **Thời gian thực qua MQTT** | Giá trị trực tiếp đến qua WebSocket Secure; sensor cập nhật ngay, không phải chờ lần poll kế tiếp |
| **REST polling làm nền** | Cloud API bổ sung những gì MQTT không có, ví dụ tổng cấp dự án |
| **Chịu được sự cố ở một trong hai nguồn** | Nếu poll lỗi, các entity được MQTT cấp dữ liệu vẫn available — và ngược lại |
| **Thống kê theo ngày** | Công suất đỉnh, công suất trung bình và số giờ phát điện trong ngày, theo dõi riêng từng thiết bị |
| **Chế độ dự án hoặc thiết bị** | Gộp nhiều bộ điều khiển vào một dashboard, hoặc theo dõi riêng một bộ |
| **Cố định tên thiết bị và entity** | `PV1`/`PV2` theo thứ tự số của GUID thiết bị, nên entity ID không bị xáo trộn giữa các lần khởi động lại |
| **Chu kỳ poll điều chỉnh được** | Number entity `Update Frequency`, 1–30 giây, không phải sửa YAML |
| **Xác thực tự phục hồi** | Token được làm mới trước khi hết hạn, và token bị máy chủ từ chối sẽ bị bỏ để lần poll sau đăng nhập lại |
| **Tự tìm thông tin đăng nhập MQTT** | Đọc từ cloud API; không cần cấu hình tay |
| **Config flow trên giao diện** | Thiết lập, xác thực lại và cấu hình lại đều trong giao diện Home Assistant (tiếng Anh + tiếng Việt) |
| **Tải diagnostics** | Một cú nhấp trong giao diện, mật khẩu và token đã được che |

## Thiết bị hỗ trợ

| Model | Điện áp PV | Dòng sạc | Điện áp pin | Kết nối |
|-------|-----------|----------------|-----------------|--------------|
| 40A WiFi | 18–100 V | 1–40 A | 6–120 V | WiFi + cloud SmartSolar + MQTT |
| 45A WiFi | 18–100 V | 1–45 A | 6–120 V | WiFi + cloud SmartSolar + MQTT |
| 60A WiFi | 18–100 V | 1–60 A | 6–120 V | WiFi + cloud SmartSolar + MQTT |

Các bộ điều khiển SmartSolar khác gửi dữ liệu lên cùng cloud API và broker cũng dùng được —
integration không gán cứng tên model.

## Yêu cầu

- Home Assistant **2026.9** trở lên (`hacs.json` khai báo `2026.9.0`)
- Python **3.14.2+** (Home Assistant 2026.x yêu cầu)
- `aiohttp >= 3.8.0` và `aiomqtt >= 2.0` — khai báo trong manifest, Home Assistant tự cài.
  Nếu không import được `aiomqtt`, integration vẫn chạy bằng REST polling.
- Tài khoản SmartSolar đăng ký tại [smartsolar.io.vn](https://smartsolar.io.vn)

**Phiên bản hiện tại:** v2.0.7 — đã kiểm tra trực tiếp trên **HA 2026.10.1** (2026-10-12); mỗi bản phát hành đều được deploy lên hệ thống tham chiếu và kiểm tra bằng `verify_live.py` ngay sau khi gắn tag.

## Cài đặt

### HACS (khuyến nghị)

Kho này **không** có trong danh sách mặc định của HACS, nên cần thêm dưới dạng kho *tuỳ chỉnh*
(custom repository):

1. HACS → **Integrations** → ⋮ (góc trên bên phải) → **Custom repositories**
2. Repository: `https://github.com/ngoviet/smartsolar_ha`, category **Integration**
3. Tìm **SmartSolar MPPT** trong HACS và bấm **Download**
4. Khởi động lại Home Assistant

Nút ở trên mở đúng hộp thoại này trên Home Assistant của bạn.

### Cài thủ công

```bash
cd /config/custom_components
# tên thư mục phải trùng với domain của integration
git clone https://github.com/ngoviet/smartsolar_ha.git smartsolar_ha
# sau đó khởi động lại Home Assistant
```

> **Nâng cấp từ v1.x?** Thư mục và domain đổi từ `smartsolar_mppt` sang `smartsolar_ha` ở
> v2.0.0. Home Assistant nhận diện integration theo tên thư mục, nên entry cũ không thể nạp
> dưới tên mới. Hãy chuyển (migrate) entry đã lưu thay vì xoá nó — xem
> [Chuyển từ v1.x](#chuyển-từ-v1x).

### Chuyển từ v1.x

v2.0.0 đổi tên integration và domain từ `smartsolar_mppt` sang `smartsolar_ha`, vì Home
Assistant yêu cầu thư mục integration phải trùng với `manifest["domain"]` và domain chỉ được
chứa chữ thường và dấu gạch dưới. (Tên kho GitHub không liên quan — Home Assistant không bao
giờ đọc nó.)

Entry hiện có không thể nạp dưới domain mới. Có hai cách:

- **Chuyển entry tại chỗ (khuyến nghị).** Sửa domain trong `core.config_entries`,
  `core.entity_registry` và `core.device_registry` khi Home Assistant đang dừng. `entry_id`
  giữ nguyên, nên mọi unique ID, entity ID, dashboard và thống kê dài hạn đều còn nguyên. Quy
  trình chi tiết nằm trong `CLAUDE.md` → "Deploying to HA".
- **Xoá rồi thêm lại.** Đơn giản hơn, nhưng tạo ra `entry_id` mới, nên entity ID bị thêm hậu
  tố `_2`, dashboard phải sửa lại và thống kê dài hạn bị mồ côi.

## Cấu hình

1. **Settings** → **Devices & Services** → **Add Integration**
2. Tìm **SmartSolar MPPT**
3. Nhập tên đăng nhập và mật khẩu tài khoản SmartSolar
4. Chọn chế độ:
   - **Device** — một bộ điều khiển, xác định bằng **Chipset ID**
   - **Project** — mọi bộ điều khiển của một dự án, theo **Project ID**, hoặc theo danh sách
     **Device ID**
5. Nhập các ID và gửi

Thông tin đăng nhập được kiểm tra trước khi tạo entry, và việc kiểm tra này cũng chạy khi xác
thực lại và cấu hình lại — mật khẩu trống hoặc sai sẽ bị từ chối thay vì được ghi vào và làm
hỏng một entry đang chạy tốt.

### Các khoá của config entry

| Khoá | Ý nghĩa |
|-----|---------|
| `username`, `password` | Thông tin đăng nhập tài khoản SmartSolar (mật khẩu được che trong diagnostics) |
| `mode` | `device` hoặc `project` |
| `device_type` | Loại thiết bị dùng cho lệnh gọi API dự án-theo-thiết-bị |
| `chipset_ids` | Chế độ thiết bị: đúng một Chipset ID. Dự án theo thiết bị: các GUID được theo dõi. Dự án theo ID: không có — danh sách thiết bị được lấy từ phản hồi API lúc chạy (entry cũ có thể mang danh sách rỗng) |
| `project_id` | Chỉ dùng cho chế độ dự án theo ID |
| `update_interval` | Chu kỳ poll tính bằng giây, 1–30 (bị kẹp vào khoảng này khi nạp, sửa sau được bằng number entity) |

### Chu kỳ cập nhật

Number entity `Update Frequency` đổi chu kỳ poll trong khoảng 1 đến 30 giây. Giá trị được lưu
trong config entry nên giữ được qua các lần khởi động lại, và entity này vẫn available khi
cloud gặp sự cố — đây là thiết lập cục bộ, và giảm tần suất poll chính là điều bạn có thể muốn
làm lúc đó.

## Thực thể

### Sensor (13 cho mỗi thiết bị)

| Sensor | Đơn vị | Nguồn |
|--------|------|--------|
| PV Voltage | V | MQTT + REST |
| PV Current | A | MQTT + REST |
| Battery Voltage | V | MQTT + REST |
| Battery Current | A | MQTT + REST |
| Charge Power | W | MQTT + REST |
| Today Energy | kWh | MQTT + REST (`total_increasing`, về 0 lúc nửa đêm) |
| Total Energy | kWh | MQTT + REST (`total_increasing`) |
| Temperature | °C | MQTT + REST |
| Status | — | MQTT + REST, ánh xạ từ mã số (Online / Charging / Idle (No Sun) / Fault) |
| WiFi Signal | % | MQTT, hoặc trường `signalQuality` của REST |
| Peak Power Today | W | Coordinator tính từ các mẫu trong ngày |
| Avg Power Today | W | Trung bình các mẫu trong ngày (không có trọng số thời gian — xem phần giới hạn) |
| Production Hours Today | h | Số giờ trong ngày có công suất trên ~5 W |

Ở chế độ dự án bạn có 13 sensor cho mỗi thiết bị (`…pv1…`, `…pv2…`) cộng thêm 10 sensor số
liệu trực tiếp tổng của dự án; thống kê theo ngày chỉ có theo từng thiết bị, không có bản gộp
cấp dự án. Ở chế độ thiết bị bạn có một bộ 13 sensor.

Entity ID do Home Assistant tạo từ tên thiết bị, nên có dạng
`sensor.<device_slug>_pv1_pv_voltage`. Unique ID có dạng `{entry_id}_{prefix}_{sensor_type}`,
vì thế chuyển entry tại chỗ (thay vì xoá rồi thêm lại) giữ nguyên mọi entity ID và toàn bộ
thống kê dài hạn.

### Gộp số liệu nhiều thiết bị

Khi tổng của dự án được tính từ từng bộ điều khiển — xảy ra mỗi khi `synthesisStreams` của
chính cloud bị thiếu hoặc cũ — mỗi loại sensor có một cách gộp rõ ràng:

| Cách gộp | Sensor | Lý do |
|----------|---------|-----|
| **sum** (cộng) | Dòng PV, dòng pin, công suất sạc, điện năng hôm nay/tổng | Các giá trị này cộng dồn được giữa các bộ điều khiển |
| **average** (trung bình) | Điện áp PV, điện áp pin, nhiệt độ, sóng WiFi | Cả hai bộ sạc cùng nằm trên một bus 24 V, cộng lại sẽ ra ~53 V |
| **max** (lớn nhất) | Status | Trạng thái xấu nhất được chọn |

Công suất sạc, các dòng điện và chất lượng sóng luôn được gộp cục bộ, vì giá trị tổng hợp của
cloud cho các mục này thường xuyên bị thiếu.

### Number

| Entity | Khoảng | Ghi chú |
|--------|-------|-------|
| Update Frequency | 1–30 giây | Tự khôi phục giá trị sau khi khởi động lại; luôn available |

## Dịch vụ

| Dịch vụ | Trường | Mô tả |
|---------|--------|-------------|
| `smartsolar_ha.refresh_token` | `entry_id` (bắt buộc) | Buộc làm mới token ngay cho một config entry |

## Xử lý sự cố

| Triệu chứng | Cần kiểm tra |
|---------|---------------|
| Không có dữ liệu sensor | Thông tin đăng nhập hợp lệ và thiết bị đang online trong app SmartSolar; xem log INFO của integration |
| Home Assistant yêu cầu xác thực lại | Mật khẩu tài khoản bị từ chối (đã đổi hoặc bị thu hồi). Nhập mật khẩu mới vào thông báo — các entity ở trạng thái `unavailable` cho đến khi bạn nhập, và không cần cấu hình lại gì khác |
| Integration không nạp được | `manifest.json` cần `aiohttp` và `aiomqtt`; xem lỗi loader trong log Home Assistant |
| Lỗi cloud (502/503/hết thời gian chờ) | Cloud SmartSolar tạm thời không phản hồi — poll được thử lại có backoff, và các sensor do MQTT cấp vẫn cập nhật |
| Lỗi token | Token tự làm mới; dịch vụ `smartsolar_ha.refresh_token` buộc làm mới cho một entry |
| MQTT không bao giờ kết nối | Phải cho phép WSS đi ra tới `mqttx.smartsolar.io.vn:8084`; dù vậy integration vẫn tiếp tục poll qua REST |
| Entity `unavailable` khi cloud gặp sự cố | Không còn xảy ra nữa: sensor available khi poll **hoặc** dữ liệu MQTT trực tiếp cấp được cho nó |
| WiFi Signal `unknown` | Bình thường với firmware không gửi `signalQuality` (xem phần giới hạn) |
| Đã thêm thiết bị vào dự án nhưng không thấy | Khởi động lại Home Assistant: bộ entity được cố định lúc thiết lập |

### Giới hạn đã biết và hành vi có chủ đích

Đây là lựa chọn, không phải lỗi — mỗi điểm đều có test:

- **Thiết bị xuất hiện lần đầu sau khi thiết lập sẽ không có entity.** Tập subscription MQTT và
  tập entity được cố định lúc thiết lập. Sensor theo thiết bị được gắn nhãn `PV1`, `PV2`, … theo
  thứ tự số của GUID, nên chèn một thiết bị có GUID nhỏ hơn sẽ đánh số lại entity ID hiện có và
  làm hỏng dashboard. Hãy khởi động lại Home Assistant sau khi thêm bộ điều khiển.
- **`Avg Power Today` là trung bình mẫu**, không phải trung bình có trọng số thời gian: MQTT gửi
  khoảng 2 Hz và mỗi lần poll thêm một mẫu.
- **`WiFi Signal` luôn `unknown` với một số firmware.** Bộ 40A ở hệ thống tham chiếu không bao
  giờ báo `signalQuality` (API trả về `null` và payload MQTT của nó dùng định dạng cũ không có
  trường này).
- **Sensor không khôi phục giá trị cũ** sau khi khởi động lại. Chúng kế thừa `RestoreEntity`,
  nhưng sensor hiển thị `unknown` cho đến lần poll hoặc tin MQTT đầu tiên, vì một số đo được
  khôi phục có thể đã cũ bao lâu cũng không biết. Chỉ `Update Frequency` khôi phục giá trị của
  chính nó.
- **Khi SmartSolar gặp sự cố, các entity chuyển sang `unavailable` và log vẫn yên lặng.** Sensor
  available khi poll **hoặc** MQTT trực tiếp cấp được dữ liệu, nên khi không kết nối được nhà
  cung cấp thì cả hai đường đều ngừng và không có gì để hiển thị. Integration chỉ báo một lần
  (một `WARNING` + một `INFO` khi phục hồi); không lặp lại lỗi ở mỗi lần poll. **Mật khẩu bị từ
  chối** được xử lý khác một cách có chủ đích: integration yêu cầu Home Assistant mở luồng xác
  thực lại thay vì thử lại như thể đang có sự cố. Xem ghi chú phát hành tại
  [Releases](https://github.com/ngoviet/smartsolar_ha/releases).
- **Diagnostics có chứa tên đăng nhập tài khoản** (mật khẩu và token đã được che), để có thể
  lần ra bản diagnostics thuộc tài khoản nào.

## Cách hoạt động

```
        SmartSolar cloud (api.smartsolar.io.vn)      SmartSolar broker
        POST /Auth/Login, GET /Metric/*,             mqttx.smartsolar.io.vn:8084
        GET /Device/Status                                  │ WSS /mqtt
                 │                                          │
        SmartSolarAPI                                SmartSolarMQTTClient
        (auth, token refresh,                        (subscribe per device,
         retry with backoff)                          parse payloads, reconnect)
                 └────────────────┬─────────────────────────┘
                                  ▼
                 SmartSolarDataUpdateCoordinator
                 (poll + merge MQTT into the polled data,
                  daily statistics, 1 Hz listener notification cap)
                                  │
                 sensors (×13 per device) · Update Frequency · diagnostics
```

**REST API**

- `POST /Auth/Login` trả về token kèm thời hạn; token được làm mới trước khi hết hạn, và lỗi
  `401` sẽ bỏ token đang lưu để lần gọi sau đăng nhập lại.
- Lỗi tạm thời (lỗi mạng, 5xx, 408, 429) được thử lại với exponential backoff — với
  `RETRY_MAX_ATTEMPTS = 3` thì thời gian chờ là 1 giây rồi 2 giây. Các phản hồi 4xx khác báo
  lỗi ngay.
- Chế độ thiết bị đọc `GET /Device/Status?deviceGuid=…`; chế độ dự án đọc
  `GET /Metric/ProjectMetrics?projectId=…` hoặc
  `GET /Metric/SynthesisMetrics?deviceType=…&deviceGuids=…`.

**MQTT**

- WebSocket Secure ở cổng 8084, thông tin đăng nhập lấy từ phản hồi API.
- Hiểu được ba dạng payload: định dạng `update_device_metrics` hiện tại (`signalQuality` ở cấp
  ngoài cùng), định dạng `updateDeviceLog` cũ hơn (không có chất lượng sóng — sensor WiFi của
  thiết bị đó giữ `unknown`), và dạng dictionary phẳng đời cũ với các khoá được ánh xạ qua
  `MQTT_FIELD_MAPPING`.
- Broker dùng chung cho mọi khách hàng SmartSolar, nên integration chỉ subscribe các topic theo
  thiết bị của config entry này và từ chối mọi GUID thiết bị khác trong callback.
- Kết nối bị rớt sẽ nối lại sau 5 giây; trong lúc đó REST polling vẫn tiếp tục.

## Phát triển

```bash
py -3.14 -m venv .venv                       # Home Assistant 2026.x cần Python >= 3.14.2
.venv/Scripts/python -m pip install -e ".[test,dev,deploy]"

.venv/Scripts/python -m pytest tests/ --cov=custom_components/
.venv/Scripts/ruff check custom_components/ tests/ upload_to_ha.py deploy_to_ha.py verify_live.py
.venv/Scripts/ruff format --check custom_components/ tests/ upload_to_ha.py deploy_to_ha.py verify_live.py
.venv/Scripts/python -m mypy custom_components/
```

Extra `[test]` ghim `homeassistant==2026.9.4` — phiên bản mà integration được kiểm tra và cũng
là phiên bản hệ thống tham chiếu đang chạy. Extra `[deploy]` (`paramiko`) chỉ cần cho các script
hỗ trợ qua SSH.

**Bộ test:** badge CI ở trên luôn phản ánh trạng thái hiện tại — ruff, format, mypy và pytest là các cổng bắt buộc, chạy với extra `[test]` trên phiên bản Home Assistant đã ghim.

### Cùng một cổng kiểm tra chạy ở máy và trên CI

`deploy_to_ha.py` và `.github/workflows/ci.yml` lint cùng một danh sách tệp, và CI chạy thêm toàn
bộ test kèm coverage cùng HACS validation.

### Deploy lên hệ thống đang chạy

```bash
.venv/Scripts/python deploy_to_ha.py               # deploy có cổng: lint + types + tests → backup → upload → restart → chờ
.venv/Scripts/python deploy_to_ha.py --skip-checks # đẩy lại khẩn cấp
.venv/Scripts/python verify_live.py                # kiểm tra các entity đã deploy là đúng
```

`deploy_to_ha.py` lưu bản đang deploy vào `_local_archive/deployed/`, xoá các tệp không còn trong
kho, rồi chờ hai điều: REST API trả lời, **và** config entry của integration này báo `loaded`. Nó
từ chối deploy khi entry bị kẹt ở `setup_retry`/`setup_error`, và không nhầm một danh sách không
đọc được thành entry không tồn tại.

`verify_live.py` có thể chạy nối ngay sau đó: vì Home Assistant ghi trạng thái entity bất đồng bộ
sau khi entry nạp xong, trình kiểm tra sẽ chờ (có giới hạn, 12 × 5 giây) cho đến khi mọi entity nó
cần kiểm tra đều tồn tại rồi mới đánh giá giá trị.

### Cấu trúc dự án

```
custom_components/smartsolar_ha/
├── __init__.py        Setup/unload, đăng ký service, migrate entry
├── manifest.json      Domain, phiên bản, requirements
├── const.py           Metadata sensor, quy tắc gộp, cấu hình MQTT và log
├── helpers.py         Hàm thuần (coerce_float, as_list, device_logs, …)
├── config_flow.py     Auth → mode → device/project, reauth, reconfigure
├── api.py             REST client: đăng nhập, làm mới token, retry/backoff
├── mqtt_client.py     MQTT client: kết nối WSS, phân tích payload, nối lại
├── coordinator.py     Polling + gộp MQTT + thống kê theo ngày
├── sensor.py          Sensor entity (thiết bị, tổng hợp dự án, thiết bị trong dự án, thống kê)
├── number.py          Entity Update Frequency
├── diagnostics.py     Diagnostics của config entry (đã che thông tin bí mật)
├── brand/             Icon/logo cục bộ do Home Assistant phục vụ
└── translations/      en.json, vi.json
tests/                 Test đơn vị, test script và test end-to-end (xem badge CI để biết số lượng hiện tại)
```

`CLAUDE.md` ghi chi tiết các quyết định kiến trúc và mọi bản sửa sau audit; `KNOWLEDGE.md` là tài
liệu tiếng Việt đi sâu về cloud API và MQTT broker.

### Đóng góp

Hoan nghênh issue và pull request.

- **Báo lỗi**: [mở issue](https://github.com/ngoviet/smartsolar_ha/issues/new)
- **Đề xuất tính năng**: [mở issue](https://github.com/ngoviet/smartsolar_ha/issues/new)
- **Code**: [tạo pull request](https://github.com/ngoviet/smartsolar_ha/compare)

Hãy chạy cổng kiểm tra cục bộ ở trên trước khi mở pull request; CI chạy cùng các bước kiểm tra
đó và thêm HACS validation.

<!-- support:start -->
## Ủng hộ dự án

<p align="center">
  <img src="https://raw.githubusercontent.com/ngoviet/.github/main/assets/support/banner.svg" width="720" alt="Built and maintained by one person in Việt Nam">
</p>

<p align="center">
  <a href="https://buymeacoffee.com/ngoviet"><img src="https://raw.githubusercontent.com/ngoviet/.github/main/assets/support/btn-coffee.svg" height="56" alt="Buy me a coffee"></a>
  &nbsp;
  <a href="https://github.com/sponsors/ngoviet"><img src="https://raw.githubusercontent.com/ngoviet/.github/main/assets/support/btn-sponsors.svg" height="56" alt="GitHub Sponsors"></a>
</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/ngoviet/.github/main/assets/support/vietqr-card.svg" width="300" alt="VietQR — MB Bank — NGO DUC VIET — 911051212">
</p>

<p align="center"><sub>🇻🇳 VietQR: quét bằng app ngân hàng bất kỳ, nội dung <code>Ung ho smartsolar_ha</code>. Cảm ơn bạn đã giúp dự án tiếp tục phát triển!</sub></p>
<p align="center"><sub>Tiền mã hoá (mạng BEP20): <code>0xcbf0402b232cb24b4d998f09a4884f87b5f74750</code> · USDT · USDC · BNB · ETH · BTC</sub></p>
<!-- support:end -->

## Giấy phép

[MIT](LICENSE) © Ngô Đức Việt. Không liên kết hay được Mạnh Quân / SmartSolar bảo trợ.
