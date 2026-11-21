# Yolo UNO – Danh sách linh kiện cần chốt (sheet `mcu` và `peripherals`)

Mỗi mục gồm: câu hỏi cần chốt, các option kèm lợi/hại, và một gợi ý nghiêng về (chỉ là gợi ý, quyết định là của nhóm).
Các thông số kỹ thuật phải đối chiếu lại với datasheet của nhà sản xuất trước khi chốt mã linh kiện.

## Bối cảnh: chỉ nộp bản mềm, không gia công

Vì không làm mạch thật, các yếu tố như hàn tay có dễ không, mua ở đâu, còn hàng không, giá bao nhiêu **không còn là tiêu chí chính**. Thay vào đó, khi chọn linh kiện ta ưu tiên:

1. **Khớp spec và hình dạng UNO** (R1–R11), đặc biệt kích thước, vị trí header và 4 lỗ bắt vít (R10).
2. **Có sẵn symbol, footprint và mô hình 3D trong thư viện chuẩn của KiCad**, để tiết kiệm thời gian và có hình 3D đẹp khi đưa vào báo cáo.
3. **Có datasheet rõ ràng**, để trích dẫn và chứng minh bằng tính toán (giảng viên yêu cầu phải chứng minh phương án đáp ứng Bảng 1).
4. **Hợp lý về kỹ thuật và thực tế** (mức logic, dòng, điện áp). Nên đặt rule DRC theo năng lực của nhà sản xuất PCB phổ thông, để thiết kế vẫn đáng tin nếu sau này muốn làm thật.
5. **Layout gọn và sạch lỗi** (ERC và DRC không còn lỗi), dễ trình bày.

Mã linh kiện vẫn nên là mã thật, có tồn tại, vì bản thiết kế sẽ được đánh giá như một sản phẩm có thể sản xuất.

## Đã chốt theo spec (không cần quyết)

- Module: ESP32-S3-WROOM-1 (anten PCB, không dùng bản `U` có đầu nối anten ngoài). Khi layout phải để trống vùng anten.
- Nạp chương trình qua USB native của ESP32-S3 (R4), nên **không cần chip cầu UART** (CP2102, CH340...).
- Nút RESET (chân EN) và nút BOOT (GPIO0); mạch EN dùng RC 10 kΩ / 1 µF theo sơ đồ giải pháp.
- Số lượng đầu nối theo R6–R9: 12 Grove, 4 bộ servo GVS, 1 QWIIC, LED nguồn, LED D13, 1 NeoPixel.

---

## Nhóm A. Vi điều khiển

### A1. Biến thể module (bộ nhớ)
Spec yêu cầu PSRAM 8 MB, Flash 4 MB hoặc 16 MB, nên có hai hướng: **N4R8** hoặc **N16R8**.

| Option | Lợi | Hại |
|---|---|---|
| N4R8 | Đủ cho firmware cỡ vừa; thể hiện chọn cấu hình tiết kiệm | Flash 4 MB dễ chật nếu cần OTA, lưu dữ liệu, mô hình |
| N16R8 | Dư chỗ cho OTA, dữ liệu, mở rộng; biện luận "dư tài nguyên" dễ | Đắt hơn (ít quan trọng vì không gia công) |

Ghi chú chung: cả hai dùng PSRAM kiểu octal nên các chân GPIO35, 36, 37 bị chiếm nội bộ, không dùng được. Spec hiện không dùng các chân này nên không ảnh hưởng.
Cần kiểm tra: mã module có trong trang thông tin đặt hàng (ordering information) của Espressif, để chắc là mã thật.
Nghiêng về: N16R8, vì giá và hàng không còn là vấn đề.

---

## Nhóm B. Đầu nối và cơ khí

### B1. Header Arduino UNO R3
| Option | Lợi | Hại |
|---|---|---|
| Header cái 2,54 mm (socket) | Đúng kiểu UNO, cắm shield vào được | Cao hơn, mô hình 3D lớn hơn |
| Header đực 2,54 mm (pin) | Gọn | Không cắm shield UNO được; lệch mục tiêu "tương thích UNO" |

Cần kiểm tra: các hàng 8/10/6/8 chân và vị trí chính xác theo kích thước UNO R3 (R10). Sai vị trí là lỗi spec chứ không chỉ lỗi layout.
Nghiêng về: header cái, theo đúng mục tiêu tương thích UNO.

### B2. Cổng Grove (12 cổng)
| Option | Lợi | Hại |
|---|---|---|
| Xuyên lỗ (THT) | Chắc về cơ khí; layout xuyên hai mặt | Chiếm chỗ ở cả hai mặt, khó xếp 12 cổng gọn |
| Dán (SMD) | Gọn, tất cả nằm trên một mặt, dễ route | Phải có footprint và 3D đúng |

Cần kiểm tra: chuẩn Grove thường dùng bước chân 2,0 mm, 4 chân; xác nhận lại với datasheet đầu nối và xem KiCad đã có footprint chưa.
Nghiêng về: loại nào có footprint và mô hình 3D sẵn; nếu cả hai đều có thì SMD cho layout gọn.

### B3. Cổng QWIIC / STEMMA QT (JST-SH 1,0 mm, 4 chân)
| Option | Lợi | Hại |
|---|---|---|
| SMD, đứng (vertical) | Dễ bố trí trong vùng board | Cáp cắm dễ bị đầu nối khác che |
| SMD, vuông góc (right-angle) | Cáp đi sát mặt board, giống đa số board QWIIC | Cần đặt sát cạnh board |

Nghiêng về: right-angle đặt sát cạnh board.

### B4. Chân servo GVS (4 bộ)
| Option | Lợi | Hại |
|---|---|---|
| 4 header đực 1×3 riêng | Bố trí linh hoạt theo vị trí D2/D11/D12/D13 | Nhiều linh kiện hơn trong layout |
| 1 khối header 1×12 | Ít linh kiện | Khó khớp vị trí tín hiệu và khoảng cách |

Nghiêng về: 4 header 1×3 riêng (theo thứ tự G–V–S đúng chuẩn servo).

### B5. Nút nhấn RESET và BOOT
| Option | Lợi | Hại |
|---|---|---|
| Tactile 6×6 mm | Footprint phổ biến, mô hình 3D có sẵn | Chiếm chỗ hơn |
| Tactile nhỏ (khoảng 3×4 mm) | Tiết kiệm chỗ | Cần kiểm tra footprint/3D có sẵn không |

Nghiêng về: loại có footprint và 3D sẵn trong KiCad; RESET và BOOT dùng cùng loại.

---

## Nhóm C. Hiển thị

### C1. LED nguồn và LED D13
| Option | Lợi | Hại |
|---|---|---|
| LED SMD 0805 | Footprint phổ biến, nhìn rõ trên layout | To hơn |
| LED SMD 0603 | Gọn | Cần chú ý khoảng cách DRC nếu xếp sát |

Cần chốt: màu LED (ví dụ xanh lá cho nguồn, màu khác cho D13) và giá trị điện trở hạn dòng (tính theo điện áp rơi LED và độ sáng mong muốn; nên có phép tính trong báo cáo).
Nghiêng về: cùng cỡ với các linh kiện thụ động.

### C2. NeoPixel (LED RGB) và mức logic
Điểm cần chú ý: LED cấp 5 V (theo R3) nhưng tín hiệu từ GPIO45 chỉ 3,3 V. Với nhiều loại WS2812, ngưỡng mức cao đầu vào tính theo 0,7 × điện áp nguồn của LED, nên 3,3 V có thể không đủ ổn định. Cần đối chiếu datasheet loại LED cụ thể. Đây là điểm mà giảng viên đã đặt dấu hỏi trong sơ đồ, nên nên có biện luận rõ trong báo cáo.

| Option | Lợi | Hại |
|---|---|---|
| Cấp 5 V, nối thẳng GPIO45 | Đơn giản nhất, ít linh kiện | Có thể không nhận đúng mức logic; khó biện luận |
| Thêm IC dịch mức (ví dụ loại buffer HCT/AHCT) | Đúng cách làm chuẩn, dễ chứng minh | Thêm linh kiện và chỗ trên board |
| Hạ áp cấp cho LED bằng một diode nối tiếp | Ít linh kiện, hạ ngưỡng đầu vào | Cần tính toán để chứng minh đủ ngưỡng; màu/độ sáng có thể đổi nhẹ |

Cần chốt thêm: kiểu LED (WS2812B 5050 hay loại nhỏ hơn). GPIO45 là chân strapping, nên đừng để mạch ngoài kéo mức cố định khi boot.
Nghiêng về: IC dịch mức, hoặc chứng minh bằng datasheet rằng loại LED chọn nhận được 3,3 V.

---

## Nhóm D. Linh kiện phụ trợ

### D1. Cỡ chân dán cho điện trở và tụ (thống nhất với bạn cùng nhóm)
| Option | Lợi | Hại |
|---|---|---|
| 0805 | Dễ đọc trên layout, khoảng cách dễ thỏa DRC | Chiếm chỗ hơn |
| 0603 | Cân bằng kích thước và độ gọn | Cần kiểm tra khoảng cách khi xếp sát |
| 0402 | Rất gọn, tiết kiệm diện tích board | Dễ vướng rule DRC khi dày đặc; khó đọc ký hiệu |

Nghiêng về: 0603 hoặc 0805 cho toàn bộ thụ động; hai sheet dùng chung một cỡ cho linh kiện cùng loại.

### D2. Điện trở kéo lên I2C
Chỉ đặt **một bộ** cho cả bus (GPIO11/12), kéo lên 3,3 V, vì bus được dùng chung bởi header, A4/A5, 4 cổng Grove I2C và QWIIC.

| Option | Lợi | Hại |
|---|---|---|
| 4,7 kΩ | Phổ biến, cân bằng | Có thể chậm nếu nhiều thiết bị và dây dài |
| 2,2 kΩ | Cạnh lên nhanh hơn khi tải điện dung lớn | Dòng kéo xuống lớn hơn; một số thiết bị chịu dòng thấp hơn |
| 10 kΩ | Ít tốn dòng | Dễ chậm hoặc lỗi khi nhiều thiết bị |

Nghiêng về: 4,7 kΩ.

### D3. Tụ lọc nguồn cho module
Đặt gần chân 3V3 của module, thường gồm một tụ lớn (cỡ 10 µF) và một tụ nhỏ (cỡ 0,1 µF). Cần đối chiếu giá trị chính xác với Hardware Design Guidelines của Espressif.
| Option | Lợi | Hại |
|---|---|---|
| Làm đúng theo khuyến nghị Espressif | Có căn cứ để trích dẫn, giảm rủi ro reset ngẫu nhiên khi Wi-Fi bật | Cần đọc tài liệu kỹ |
| Thêm tụ dự phòng (footprint trống) | Thể hiện thiết kế có dư địa chỉnh | Thêm linh kiện chưa chắc cần |

Nghiêng về: làm đúng theo khuyến nghị, bỏ tụ dự phòng vì không có bước đo thực tế.

### D4. Test point
Vì không có board thật để đo, test point mất vai trò debug; chỉ còn giá trị trình bày.
| Option | Lợi | Hại |
|---|---|---|
| Có test point cho `+3V3`, `+5V`, `GND`, `EN` | Thể hiện thiết kế có tính đến kiểm tra, bản thiết kế "giống thật" hơn | Tốn chỗ và công vẽ |
| Không có | Gọn, ít việc | Mất điểm trình bày (nếu giảng viên đánh giá tính thực tế) |

Nghiêng về: có test point cho `+3V3`, `+5V`, `GND` nếu còn chỗ; không bắt buộc.

---

## Nhóm E. Cần thống nhất với bạn cùng nhóm

- Cỡ chân dán chung cho linh kiện thụ động (D1).
- Cổng Grove cấp 5 V hay 3,3 V (R3 nói 5 V cho Grove, R9 nói QWIIC 3,3 V).
- Nguồn cấp cho NeoPixel (C2), và tên net + điểm lấy `VIN`.
- LDO 3,3 V cần chịu dòng tối thiểu theo sơ đồ giải pháp, kiểm tra thêm với ngân sách dòng thực tế.
- Quy ước thư viện: ai tạo symbol/footprint/3D nào, lưu ở đâu trong `libs/`, để không bị trùng.
- Rule DRC chung (độ rộng đường, khoảng cách, đường kính via) để hai sheet và PCB không mâu thuẫn.

---

## Bảng chốt (điền sau khi quyết)

| Mục | Linh kiện chọn | Mã / hãng | Datasheet | Footprint KiCad | Mô hình 3D |
|---|---|---|---|---|---|
| A1 Module | | | | | |
| B1 Header Arduino | | | | | |
| B2 Grove | | | | | |
| B3 QWIIC | | | | | |
| B4 Servo | | | | | |
| B5 Nút nhấn | | | | | |
| C1 LED | | | | | |
| C2 NeoPixel (+ dịch mức nếu có) | | | | | |
| D1 Cỡ chân dán thụ động | | | | | |
| D2 Pull-up I2C | | | | | |
| D3 Tụ lọc module | | | | | |
| D4 Test point | | | | | |