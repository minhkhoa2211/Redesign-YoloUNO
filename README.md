# Yolo UNO – Thiết kế lại board (ESP32-S3)

README nội bộ giữa hai thành viên, dùng để thống nhất cách chia sheet, tên net và linh kiện đã chốt. Chưa dành cho người ngoài đọc.

- **KiCad:** 10.0 (cả hai cùng dùng đúng bản này, không mở bằng bản major khác)
- **Module chính:** ESP32-S3-WROOM-1 (biến thể N16R8)
- **Spec và sơ đồ chân tham khảo:** xem thư mục `docs/`
- **Chỉ nộp bản mềm, không gia công mạch.**

## Cách mở project

1. Clone repo về máy.
2. Mở KiCad 10.0, chọn `File > Open Project`, trỏ tới `hardware/yolo_uno.kicad_pro`.
3. Thư viện riêng của dự án nằm trong `libs/` và được khai báo bằng đường dẫn tương đối (`${KIPRJMOD}`). Không cần cài thêm gì. Nếu KiCad báo thiếu symbol hoặc footprint, kiểm tra lại `Preferences > Manage Symbol/Footprint Libraries > Project Specific Libraries`.

## Cấu trúc thư mục

| Thư mục | Nội dung |
|---|---|
| `hardware/` | Project KiCad (sheet gốc, 3 sheet con, PCB) |
| `libs/` | Symbol, footprint, mô hình 3D riêng của dự án |
| `docs/` | Spec, bảng pin mapping, bảng ngân sách dòng, danh sách linh kiện, báo cáo |
| `datasheets/` | Datasheet linh kiện |

## Ba sheet và các khối thuộc về từng sheet

Sheet gốc (`yolo_uno.kicad_sch`) chỉ chứa 3 khối hộp và dây nối giữa chúng, không vẽ linh kiện.
![Sơ đồ khối của dự án](img\block_diagram_yoloUNO.png)

### `power` (`power.kicad_sch`) – người phụ trách: bạn cùng nhóm
- Jack DC 5521, bảo vệ ngược cực, TVS chống quá áp
- Buck 5 V từ VIN (6–12 V)
- Chọn nguồn (power-path) giữa VBUS của USB và 5 V từ jack DC
- LDO 3,3 V
- USB Type-C: điện trở CC, bảo vệ ESD đường D+/D-

### `mcu` (`mcu.kicad_sch`) – người phụ trách: Khoa
- ESP32-S3-WROOM-1 (N16R8) và tụ lọc nguồn theo khuyến nghị Espressif
- Nút RESET (chân EN) kèm mạch RC 10 kΩ / 1 µF, nút BOOT (GPIO0)
- Đường USB tới chân GPIO19/GPIO20 của module
- Test point cho `EN` (nếu đủ thời gian)

### `peripherals` (`peripherals.kicad_sch`) – người phụ trách: Khoa
- Header Arduino UNO R3 (header cái 2,54 mm)
- 12 cổng Grove (4 analog, 4 digital, 4 I2C), loại dán (SMD)
- 4 bộ chân servo GVS (4 header đực 1×3 riêng)
- Cổng QWIIC / STEMMA QT (JST-SH 1,0 mm, 4 chân, SMD vuông góc)
- LED nguồn, LED D13, LED RGB NeoPixel (có IC dịch mức, xem bên dưới)
- Điện trở kéo lên I2C

## Net dùng chung đã thống nhất
![Sơ đồ chân](img\pinout_yoloUNO.png)

### Giữa `power` và các sheet khác
| Net | Hướng | Ghi chú |
|---|---|---|
| `+3V3` | `power` → `mcu`, `peripherals` | Ray 3,3 V, dùng power symbol |
| `+5V` | `power` → `peripherals` | Cấp cho servo, Grove, LED RGB. Không đi vào `mcu` |
| `GND` | chung cho cả 3 sheet | Dùng power symbol |
| `USB_D+`, `USB_D-` | `power` ↔ `mcu` | Từ đầu nối Type-C tới GPIO20 và GPIO19 |

### Giữa `mcu` và `peripherals`
Đặt tên net theo tên Arduino, không theo GPIO (phép ánh xạ sang GPIO chỉ nằm ở chân module trong sheet `mcu`):
- `A0` … `A5`
- `D0` … `D13`
- `SDA`, `SCL`
- tín hiệu NeoPixel (xem mục "Chờ chốt")

### Quy ước chung
- Ngoài `+3V3`, `+5V`, `GND` (power symbol), mọi tín hiệu giữa các sheet đi qua hierarchical label và hierarchical pin có tên **giống hệt** nhau ở hai đầu.
- `EN` và `BOOT` nằm trong sheet `mcu`, không đi qua lại giữa các sheet.

## Linh kiện đã chốt (phần `mcu` và `peripherals`)

Chi tiết lợi/hại của từng lựa chọn xem file `docs/danh_sach_linh_kien.md`.

| Mục | Đã chốt | Ghi chú |
|---|---|---|
| Module | ESP32-S3-WROOM-1 **N16R8** | Octal PSRAM nên GPIO35, 36, 37 không dùng được |
| Header Arduino UNO R3 | Header cái 2,54 mm (socket) | Vị trí và kích thước theo UNO R3 (R10) |
| Cổng Grove (12) | SMD | Kiểm tra footprint và mô hình 3D trong KiCad |
| Cổng QWIIC / STEMMA QT | SMD, vuông góc (right-angle) | JST-SH 1,0 mm, 4 chân |
| Chân servo GVS (4 bộ) | 4 header đực 1×3 riêng | Thứ tự G–V–S |
| Nút nhấn RESET, BOOT | Tactile 3×4 mm | Cùng loại cho cả hai nút |
| LED nguồn (PWR) | Xanh lá, 1 kΩ, cỡ 0603 | Cấp từ 3,3 V; Vf khoảng 2,0 V nên dòng khoảng 1,3 mA |
| LED trạng thái D13 | Đỏ (hoặc cam/vàng), 1 kΩ, cỡ 0603 | Điều khiển từ GPIO48 (3,3 V); Vf khoảng 1,8–2,0 V nên dòng khoảng 1,3–1,5 mA, an toàn cho chân GPIO |
| Cỡ chân dán điện trở, tụ | **0603** cho cả hai sheet | Cần bạn cùng nhóm xác nhận dùng chung |
| Tụ lọc nguồn module | Theo khuyến nghị Espressif | Đối chiếu Hardware Design Guidelines |
| Test point | `+3V3`, `+5V`, `GND`, `EN` | Làm nếu đủ thời gian, không bắt buộc |

**Chốt tạm (sẽ điều chỉnh khi tìm hiểu kỹ hơn):**
- NeoPixel: dùng IC dịch mức (loại buffer HCT/AHCT) giữa GPIO45 (3,3 V) và chân dữ liệu của LED (nguồn 5 V). Chưa chọn mã IC và loại LED.
- Điện trở kéo lên I2C: 4,7 kΩ, chỉ một bộ cho cả bus, kéo lên 3,3 V.

## Quy ước làm việc với Git

- Mỗi người một nhánh. Chỉ sửa sheet của mình.
- Sheet gốc (`yolo_uno.kicad_sch`) chỉ sửa sau khi báo nhau.
- Thư viện: dùng thư viện trong `libs/`, không dùng đường dẫn tuyệt đối trên máy cá nhân.
- Dải số reference: `power` dùng 1xx (R101, C101…), `mcu` dùng 2xx, `peripherals` dùng 3xx.
- Không commit file sinh tự động (đã có trong `.gitignore`). **Phải commit** `sym-lib-table` và `fp-lib-table` trong thư mục `hardware/`, vì đó là khai báo thư viện riêng của dự án.

## Chờ chốt

- [ ] Ngân sách dòng 3,3 V và 5 V (cần bảng tổng hợp, đặc biệt servo và dòng đỉnh Wi-Fi của ESP32-S3)
- [ ] Đầu nối USB Type-C đặt ở sheet `power` (đề xuất)
- [ ] Cỡ 0603 cho cả `power`: bạn cùng nhóm đồng ý không
- [ ] NeoPixel: mã IC dịch mức, loại LED, tên chân (đề xuất `NEOPIXEL`)
- [ ] Giá trị pull-up I2C cuối cùng (đang tạm 4,7 kΩ)
- [ ] Cổng Grove cấp 5 V hay 3,3 V, QWIIC cấp 3,3 V
- [ ] `A4`/`SDA` và `A5`/`SCL` trùng GPIO11/GPIO12: chốt tên net dùng chung
- [ ] Test point `+3V3`, `+5V`, `GND`: đặt ở sheet nào (đề xuất mỗi người đặt ở sheet chứa net đó)