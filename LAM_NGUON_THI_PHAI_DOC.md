# Tài liệu thiết kế khối nguồn — Yolo UNO (ESP32-S3-WROOM-1)

> **Mục đích:** làm kim chỉ nam để chốt linh kiện khối nguồn **trước khi vẽ Schematic**.
> **Phạm vi:** Jack DC 5521 (6–12 V) + USB-C (5 V) → `+5V` → `+3V3`.
> **Cách đọc:** mọi con số đều đi kèm cơ sở. Số nào không đến từ đề bài đều có nhãn `[Giả định: …]` — nhóm phải thay bằng số thực hoặc đối chiếu datasheet trước khi chốt.

---

## Quy ước nhãn (đọc trước)

| Nhãn | Ý nghĩa | Hành động của nhóm |
|---|---|---|
| `[Đề]` | Thông số bắt buộc từ đề bài / chuẩn USB | Dùng nguyên |
| `[Giả định: DS]` | Thông số **linh kiện** lấy từ trí nhớ về datasheet, **chưa mở datasheet đối chiếu trực tiếp** | Mở datasheet kiểm tra từng số trước khi chốt |
| `[Giả định: ƯL]` | Số tự ước lượng (hiệu suất, diện tích PCB, điện trở đường dẫn…) | Đo/tính lại khi có layout |
| `[Giả định: KB]` | Kịch bản tải do người viết chọn để tính mẫu (loại servo, số LED, dòng Grove…) | **Thay bằng số thực của nhóm** |
| `[Chưa có đủ thông tin - Cần nhóm tự xác nhận: …]` | Phụ thuộc phần cứng thực tế của nhóm | Làm theo hướng dẫn trong ngoặc |

Công thức vật lý thuần (ví dụ $\eta_{LDO}=V_{out}/V_{in}$) không cần nhãn.

---

## Kết luận nhanh (số liệu chính — chi tiết ở các phần sau)

| # | Kết luận | Số liệu then chốt |
|---|---|---|
| 1 | Dòng `+3V3` worst-case **853 mA** (vượt mốc 800 mA của đề); chuyển sang Buck đồng bộ **AP63203 (2 A)** | Dư gấp đôi nhu cầu (biên an toàn 134 %), $V_{IN\_MIN} = 3.8\text{ V}$ an toàn tuyệt đối với USB |
| 2 | Dòng Buck 5 V tối thiểu cần **2.58 A** ($1.3 \times 1986\text{ mA}$); chọn Buck **3.0 A (MP1584)** đạt biên an toàn **51.1 %** ($\ge 30\%$ `[Đề]`) | Tiết kiệm 232 mA từ rail 5V nhờ Buck 3.3 V; LDO 5 V (AMS1117-5.0) loại ngay: tỏa 13.9 W |
| 3 | Jack 6 V là góc khó nhất: dòng vào lớn (1.88 A), $V_{IN}$ tại Buck chỉ còn ≈ 5.81 V | TPS5430 ($V_{IN\_MIN}$ = 5.5 V) bị loại |
| 4 | TVS phải thỏa chuỗi: $V_{no\text{-}load} \le V_{RWM}$ và $V_C < V_{IN\_MAX}$ của Buck | SMBJ15A có thể rò khi adapter hở mạch 17 V; SMBJ18A kẹp 29.2 V > 28 V của MP1584 |
| 5 | Schottky OR-ing làm tụt `+5V` còn ≈ 4.59 V → servo yếu; Ideal-diode IC (LM66200) giữ **5.01 V** ở cùng dòng | Biên dòng LM66200 đạt 25.9 % so với tải Worst-case 1.986 A |
| 6 | Buck 3.3 V AP63203 **triệt tiêu hoàn toàn sự cố quá nhiệt** của LDO | Tỏa nhiệt đỉnh giảm từ 1.56 W xuống **0.31 W**; $T_J$ mát mẻ, không cần mảng đồng tản nhiệt lớn |

---

# Phần 0: Ngân sách Công suất & Bảng tính Dòng điện (Power Budget)

## 0.1. Điều kiện đầu vào

| Nguồn | Min | Typ | Max | No-load spike | Ghi chú |
|---|---|---|---|---|---|
| **Jack DC 5521** | 6.0 V `[Đề]` | 9 V `[Giả định: KB]` — adapter phổ biến | 12 V `[Đề]` | 17 V `[Giả định: ƯL]` — adapter 12 V không ổn áp, hở mạch ≈ 1.4× danh định; xung cắm nóng có thể tới ≈ 2× V<sub>IN</sub> ≈ 24 V `[Giả định: ƯL]` | `[Chưa có đủ thông tin - Cần nhóm tự xác nhận: loại adapter (ổn áp SMPS hay biến áp thường); đo điện áp hở mạch bằng đồng hồ]` |
| **USB-C VBUS** | 4.75 V `[Đề]` (chuẩn USB 2.0) | 5.0 V | 5.25 V `[Đề]` | — | Chỉ có Rd 5.1 kΩ → host cấp dòng mặc định **500 mA** (USB 2.0) / 900 mA (USB 3.x) `[Đề]` |

## 0.2. Ngân sách dòng rail `+3V3`

| # | Tải | Typical | Worst-case | Cơ sở |
|---|---|---|---|---|
| 1 | ESP32-S3-WROOM-1 | 120 mA `[Giả định: ƯL]` — trung bình khi Wi-Fi STA hoạt động (RX peak: 95–97 mA theo Bảng 6-4) | **500 mA** (Yêu cầu $I_{VDD} \ge 0.5\text{ A}$ của Datasheet) | **Đối chiếu Datasheet ESP32-S3-WROOM-1 (v1.8):**<br>- Bảng 6-2 (*Recommended Operating Conditions*): $I_{VDD}$ (Current delivered by external power supply) **Min = 0.5 A** (500 mA).<br>- Bảng 6-4 (*Wi-Fi Active Mode*): Đỉnh phát TX cực đại đạt **355 mA** (802.11b, 1 Mbps, @20.5 dBm); 297 mA (802.11g, 54 Mbps, @18 dBm); 286 mA (802.11n, HT20, @17.5 dBm); 285 mA (802.11n, HT40, @17 dBm). Đỉnh thu RX là 95 mA (HT20) / 97 mA (HT40).<br>- Bảng 6-5 (*Bluetooth LE Active Mode*): Đỉnh phát TX cực đại đạt **344 mA** (@20.0 dBm); 202 mA (@9.0 dBm); 187 mA (@0 dBm); 119 mA (@-15.0 dBm). Đỉnh thu RX là 93 mA.<br>→ Worst-case 500 mA trùng khớp yêu cầu cấp nguồn ngoài tối thiểu của hãng để chịu dòng đỉnh phát RF. |
| 2 | 12 cổng Grove (3.3 V) | 60 mA = 4 module × 15 mA `[Giả định: KB]` | 300 mA = 12 × 25 mA `[Giả định: KB]` | `[Chưa có đủ thông tin - Cần nhóm tự xác nhận: dòng thật từng module Grove — đo bằng USB power meter hoặc đọc datasheet module]` |
| 3 | Cổng QWIIC | 10 mA `[Giả định: KB]` | 50 mA `[Giả định: KB]` | 1 cổng, vài cảm biến I²C |
| 4 | LED PWR + D13 (1 kΩ, 0603) | 2 mA | 3 mA | $(3.3-2.1)/1\,\text{k}\Omega \approx 1.2$ mA/LED, $V_F \approx 2.1$ V `[Giả định: DS]`; giả định LED PWR cấp từ `+3V3` `[Giả định: KB]` |
| | **Σ `+3V3`** | **192 mA** | **853 mA** | |
| | *Σ kéo dài (ESP32 trung bình + tải Grove/QWIIC/LED worst)* | — | *473 mA* | Dùng cho tính công suất/nhiệt ở mục 2.4 |

> ⚠️ **Phát hiện & Giải pháp:** 853 mA > 800 mA (mốc của đề). Thay vì dùng LDO (bị quá nhiệt nghiêm trọng), khối 3.3 V được chốt dùng IC hạ áp xung đồng bộ **AP63203 (2 A)**, đáp ứng thoải mái dòng đỉnh phát Wi-Fi với hiệu suất 90 % và biên an toàn 134 %.

### 0.2.1. Bảng thông số kỹ thuật thực tế từ Datasheet ESP32-S3-WROOM-1 (v1.8)

Các số liệu dưới đây được trích xuất trực tiếp từ datasheet *ESP32-S3-WROOM-1 & WROOM-1U Datasheet v1.8* (thay thế hoàn toàn các giả định ước lượng trước đây liên quan đến thông số điện của MCU):

| Bảng / Mục Datasheet | Thông số | Giá trị thực tế trong Datasheet | Ý nghĩa cho thiết kế khối nguồn |
|---|---|---|---|
| **Bảng 6-1**<br>Absolute Maximum Ratings | $VDD33$ (Power supply voltage)<br>$T_{STORE}$ (Storage temperature) | **Min -0.3 V, Max 3.6 V**<br>**Min -40 °C, Max 105 °C** | Vượt quá 3.6 V sẽ gây hỏng module vĩnh viễn. |
| **Bảng 6-2**<br>Recommended Operating Conditions | $VDD33$ (Điện áp nguồn cấp khuyến nghị)<br>$I_{VDD}$ (Dòng cấp từ nguồn ngoài)<br>$T_A$ (Nhiệt độ môi trường làm việc) | **Min 3.0 V, Typ 3.3 V, Max 3.6 V**<br>**Min 0.5 A** (500 mA)<br>Bản 65 °C: Max 65 °C<br>Bản 85 °C: -40 °C đến 85 °C<br>Bản 105 °C: Max 105 °C | Nguồn cấp 3.3 V phải luôn duy trì điện áp ≥ 3.0 V (kể cả khi tải bước nhảy) và cấp dòng tối thiểu **≥ 0.5 A** cho riêng ESP32-S3. |
| **Bảng 6-4**<br>Current Consumption for Wi-Fi in Active Mode (3.3 V, 25 °C) | **TX Peak (100% duty cycle):**<br>- 802.11b, 1 Mbps, @20.5 dBm<br>- 802.11g, 54 Mbps, @18 dBm<br>- 802.11n, HT20, MCS 7, @17.5 dBm<br>- 802.11n, HT40, MCS 7, @17 dBm<br>**RX Peak:**<br>- 802.11b/g/n, HT20<br>- 802.11n, HT40 | <br>**355 mA**<br>**297 mA**<br>**286 mA**<br>**285 mA**<br><br>**95 mA**<br>**97 mA** | Đỉnh dòng phát Wi-Fi đạt cực đại **355 mA** (chuẩn 11b @20.5 dBm). Đỉnh thu RX là 95–97 mA. |
| **Bảng 6-5**<br>Current Consumption for Bluetooth LE in Active Mode (3.3 V, 25 °C) | **TX Peak:**<br>- Bluetooth LE @ 20.0 dBm<br>- Bluetooth LE @ 9.0 dBm<br>- Bluetooth LE @ 0 dBm<br>- Bluetooth LE @ -15.0 dBm<br>**RX Peak:**<br>- Bluetooth LE | <br>**344 mA**<br>**202 mA**<br>**187 mA**<br>**119 mA**<br><br>**93 mA** | Đỉnh dòng phát Bluetooth LE cực đại là **344 mA** (ở công suất tối đa 20 dBm). Đỉnh thu RX là 93 mA. |
| **Bảng 6-3**<br>DC Characteristics (3.3 V, 25 °C) | $C_{IN}$ (Điện dung chân GPIO)<br>$V_{IH}$ (Điện áp vào mức cao)<br>$V_{IL}$ (Điện áp vào mức thấp)<br>$I_{IH}, I_{IL}$ (Dòng rò vào mức cao/thấp)<br>$V_{OH}$ (Điện áp ra mức cao)<br>$V_{OL}$ (Điện áp ra mức thấp)<br>$I_{OH}$ (Dòng source mức cao, PAD_DRIVER=3)<br>$I_{OL}$ (Dòng sink mức thấp, PAD_DRIVER=3)<br>$R_{PU}$ (Trở kéo lên nội yếu)<br>$R_{PD}$ (Trở kéo xuống nội yếu) | **Typ 2 pF**<br>**Min $0.75 \times VDD$, Max $VDD + 0.3\text{ V}$** (2.475 V – 3.6 V tại 3.3 V)<br>**Min -0.3 V, Max $0.25 \times VDD$** (-0.3 V – 0.825 V tại 3.3 V)<br>**Max 50 nA**<br>**Min $0.8 \times VDD$** (2.64 V tại 3.3 V)<br>**Max $0.1 \times VDD$** (0.33 V tại 3.3 V)<br>**Typ 40 mA** (khi $V_{OH} \ge 2.64\text{ V}$)<br>**Typ 28 mA** (khi $V_{OL} = 0.495\text{ V}$)<br>**Typ 45 kΩ**<br>**Typ 45 kΩ** | $V_{OH} \ge 2.64\text{ V} < 3.5\text{ V}$ ($V_{IH}$ của NeoPixel cấp 5 V) nên bắt buộc có Level-Shifter. Trở kéo nội 45 kΩ dùng để tính toán điện trở kéo ngoài (I2C/nút bấm). |
| **Mục 5**<br>Peripherals (Reset Pin EN) | $V_{IH\_nRST}$ (Điện áp nhả reset chân EN)<br>$V_{IL\_nRST}$ (Điện áp reset chân EN) | **Min $0.75 \times VDD$, Max $VDD + 0.3\text{ V}$** (2.475 V – 3.6 V tại 3.3 V)<br>**Min -0.3 V, Max $0.25 \times VDD$** (-0.3 V – 0.825 V tại 3.3 V) | Căn cứ tính toán mạch reset RC cho chân EN, đảm bảo khi điện áp nguồn 3.3 V tụt thì chân EN kích hoạt reset đúng ngưỡng. |

## 0.3. Ngân sách dòng rail `+5V`

| # | Tải | Typical | Worst-case | Cơ sở |
|---|---|---|---|---|
| 1 | 4 Servo SG90 | 4 × 100 = **400 mA** | 4 × 250 = **1000 mA** | `[Giả định: KB — SG90: chạy không tải ≈ 100 mA, có tải ≈ 250 mA]`. Stall 650 mA/con (2600 mA cho 4 con) là **sự cố**, xem ghi chú dưới bảng |
| 2 | 1× NeoPixel WS2812B onboard | **20 mA** | **60 mA** | `[Đề]` — 1 LED WS2812B onboard cấp nguồn `+5V`; Typical = 20 mA, Worst-case (sáng trắng 100%, RGB = 255) = 60 mA |
| 3 | Chân xuất `+5V` trên header | 100 mA | 300 mA | `[Chưa có đủ thông tin - Cần nhóm tự xác nhận: nhóm cho phép shield/module ngoài lấy tối đa bao nhiêu mA từ chân +5V]` |
| 4 | **Đầu vào Buck 3.3 V (AP63203)** | **141 mA** | **626 mA** | Quy đổi công suất qua Buck ($\eta = 90\%$ `[Giả định: ƯL]`): $I_{in} = \frac{3.3 \times I_{3V3}}{0.90 \times 5.0}$; tiết kiệm 56 mA (Typ) và 232 mA (Worst) so với LDO |
| | **Σ `+5V`** | **661 mA** | **1986 mA** | |

**Ghi chú stall:** nếu cả 4 servo kẹt cùng lúc + NeoPixel trắng: 2600 + 60 + 300 + 626 = **3586 mA**. Đây là sự cố cơ khí, **không dùng để chọn Buck** (sẽ ra Buck ≥ 4.7 A, phi thực tế); xử lý bằng giới hạn dòng của Buck và firmware dừng servo khi kẹt.

## 0.4. Công thức quy đổi dòng `+3V3` → `+5V` qua Buck 3.3 V

Khác với LDO (vốn có $I_{in} \approx I_{out}$ và đốt phần áp thừa thành nhiệt), Buck 3.3 V (**AP63203**) hoạt động theo nguyên lý bảo toàn công suất với hiệu suất cao ($\eta = 90\%$ `[Giả định: ƯL]`):

$$P_{in} = \frac{P_{out}}{\eta} \iff 5.0\,\text{V} \cdot I_{Buck3V3,in} = \frac{3.3\,\text{V} \cdot I_{3V3}}{\eta}$$

Suy ra công thức dòng rút từ rail `+5V` để nuôi khối 3.3 V:

$$I_{Buck3V3,in} = \frac{3.3\,\text{V} \cdot I_{3V3}}{\eta \cdot 5.0\,\text{V}} = \frac{3.3}{4.5} \cdot I_{3V3} \approx 0.733 \times I_{3V3}$$

- **Typical ($I_{3V3} = 192\text{ mA}$):** $I_{Buck3V3,in} = \frac{3.3 \times 192}{0.90 \times 5.0} \approx \mathbf{141\text{ mA}}$ (tiết kiệm 56 mA so với mức 197 mA của LDO).
- **Worst-case ($I_{3V3} = 853\text{ mA}$):** $I_{Buck3V3,in} = \frac{3.3 \times 853}{0.90 \times 5.0} \approx \mathbf{626\text{ mA}}$ (tiết kiệm **232 mA** so với mức 858 mA của LDO).

Tổng dòng yêu cầu trên rail `+5V`:

$$I_{5V,total} = I_{servo} + I_{neo} + I_{hdr} + I_{Buck3V3,in}$$

$$\text{Worst-case: } 1000 + 60 + 300 + 626 = \mathbf{1986\ mA}$$
$$\text{Typical: } 400 + 20 + 100 + 141 = \mathbf{661\ mA}$$

## 0.5. Dòng tối thiểu cần chốt cho Buck 5 V và Buck 3.3 V

$$I_{Buck5V,min} = 1.3 \times I_{5V,worst} = 1.3 \times 1986\ \text{mA} = \mathbf{2.58\ A}$$

| Kịch bản | $I_{5V}$ | × 1.3 (hệ số an toàn ≥ 30 % `[Đề]`) | Nhận xét |
|---|---|---|---|
| Typical | 661 mA | — | Chỉ để tính hiệu suất/nhiệt |
| **Worst-case** (NeoPixel trắng 100%) | 1986 mA | **2.58 A** | Chọn Buck **3.0 A (MP1584)** đạt biên an toàn thực tế **51.1 %** ($\ge 30\%$ `[Đề]`), vượt xa yêu cầu đề bài |
| Stall (tham khảo) | 3586 mA | 4.66 A | Không dùng chọn linh kiện |

- **Buck 5 V:** chọn **3.0 A (MP1584)**; dòng yêu cầu tối thiểu 2.58 A, định mức 3.0 A đạt biên an toàn thực tế **51.1 %** so với mức Worst-case 1.986 A (thỏa mãn và vượt xa mốc $\ge 30\%$ `[Đề]`).
- **Khối 3.3 V:** chọn IC Buck đồng bộ **AP63203 (2 A)**; dòng tải Worst-case 853 mA chỉ chiếm 42.7 % định mức (biên an toàn 134 %, vượt xa mốc 800 mA của đề).

## 0.6. Ràng buộc khi chạy chỉ bằng USB

Host USB 2.0 cấp 500 mA `[Đề]`. Trừ phần đầu vào Buck 3.3 V (141 mA) còn **359 mA** cho các tải 5 V (dư hơn 56 mA so với dùng LDO), trong khi nhu cầu Typical tải 5 V là 400 + 20 + 100 = **520 mA**. 
Kết luận: 1 NeoPixel (20–60 mA) chạy hoàn toàn bình thường trên USB, nhưng firmware **vẫn phải** hạn chế số servo chạy đồng thời khi chỉ cắm nguồn USB.
`[Chưa có đủ thông tin - Cần nhóm tự xác nhận: GPIO nào đo VBUS/nhận biết nguồn đang dùng — nhóm MCU quyết định]`

---

# Phần 1: Kiến trúc Cây nguồn & Quy chuẩn Đặt tên Net

## 1.1. Sơ đồ cây nguồn

```
  DC Jack 5521 (6–12 V)                              USB-C (VBUS 5 V)
        │ VIN_RAW                                         │ VBUS_RAW        CC1/CC2: 5.1 kΩ → GND (mỗi chân 1 trở)
        ▼                                                 ▼
   [PTC, hold ≈ 3 A]                                 [PTC, hold ≈ 0.75 A] ──[USBLC6-2SC6 ESD]── USB_D+ / USB_D-
        │ VIN_FUSED                                       │ VBUS                                         │
        ├──[TVS SMBJ16A]──► GND                           │                                              ▼
        ▼                                                 │                                        (sang MCU Sheet)
   [P-MOS ideal diode + Zener clamp Gate]                 │
        │ VIN_PROT                                        │
        ▼                                                 │
   [Buck 5 V adjustable ≈ 3 A (MP1584)]                   │
        │ +5V_BUCK                                        │
        ▼                                                 ▼
     IN1 ───────────[ Ideal-diode OR-ing (LM66200) ]─────── IN2
                                  │
                                  ▼ +5V
        ┌─────────────────────────┼───────────────────────────┐
        ▼                         ▼                           ▼
   4× Servo header     1× NeoPixel onboard            [Buck 3.3 V fixed (AP63203, 2 A)]
   + chân xuất +5V     (qua 74AHCT1G125, VCC = +5V,          │ +3V3
                       Data từ GPIO45)                        ▼
                                          ESP32-S3-WROOM-1 · 12× Grove · QWIIC · LED
```

## 1.2. Định nghĩa Net

| Net | Vị trí vật lý | Ý nghĩa / điện áp dự kiến |
|---|---|---|
| `VIN_RAW` | Chân + jack DC, **trước** mọi bảo vệ | 6–12 V (hở mạch tới 17 V `[Giả định: ƯL]`), có thể có xung cắm nóng |
| `VIN_FUSED` | Sau PTC | Như `VIN_RAW` trừ sụt áp PTC (≈ 0.075 V ở 1.88 A `[Giả định: DS]`); TVS đặt tại nút này |
| `VIN_PROT` | Sau P-MOS chống ngược cực, vào chân VIN của Buck | ≈ 5.81 V (tối thiểu, adapter 6 V) → 12 V; **luôn dương**, không bao giờ âm |
| `SW_BUCK` | Nút đóng cắt của Buck | Xung vuông, nút nhiễu lớn nhất — giữ nhỏ, xa FB |
| `+5V_BUCK` | Đầu ra Buck, trước ideal-diode | 5.09 V đặt bằng FB `[Giả định: ƯL]` (xem mục 2.2) |
| `VBUS_RAW` | Chân VBUS của USB-C | 4.75–5.25 V `[Đề]` |
| `VBUS` | Sau PTC USB | Nguồn USB đã bảo vệ, vào kênh IN2 của ideal-diode |
| `+5V` | **Rail hệ thống 5 V** sau khối chọn nguồn | Cấp servo, NeoPixel, header +5V, đầu vào Buck 3.3 V |
| `+3V3` | Đầu ra Buck 3.3 V (AP63203) | Cấp ESP32-S3, Grove, QWIIC, LED |
| `GND` | Mặt đất chung | Một plane duy nhất, không tách |
| `CC1`, `CC2` | Chân cấu hình USB-C | Mỗi chân qua 5.1 kΩ xuống GND |

## 1.3. Tín hiệu giữa Power Sheet và MCU Sheet

| Tín hiệu | Hướng | Mô tả | Ghi chú |
|---|---|---|---|
| `+3V3` | Power → MCU / Peripherals | Rail cấp cho ESP32-S3, Grove, QWIIC | |
| `+5V` | Power → Peripherals | Servo, NeoPixel, header | |
| `GND` | Chung | | |
| `USB_D+` | Power ↔ MCU | Từ USB-C qua ESD tới GPIO20 (D+) của ESP32-S3 `[Giả định: DS]` | Cặp vi sai, giữ ngắn |
| `USB_D-` | Power ↔ MCU | Tới GPIO19 (D−) `[Giả định: DS]` | |
| `NEO_DATA` | MCU → Power/Peripherals | Tín hiệu điều khiển NeoPixel từ GPIO45 qua 74AHCT1G125 (VCC = 5 V) kèm R 330 Ω | GPIO45 là chân strapping (VDD_SPI), **cấm gắn trở pull-up** |
| `EN` | Nội bộ MCU | Nút Reset + RC; **không** thuộc Power Sheet | Chỉ liệt kê để rõ ranh giới |
| `VBUS_SENSE` (tùy chọn) | Power → MCU | Chia áp 100 kΩ/100 kΩ → ADC (5.25 V → 2.63 V < 3.3 V) | `[Chưa có đủ thông tin - Cần nhóm tự xác nhận: GPIO ADC còn trống; nhóm MCU chốt pin-map]` |
| `PWR_SRC` / `PWR_GOOD` (tùy chọn) | Power → MCU | Trạng thái nguồn đang dùng (chân ST của ideal-diode) hoặc PG của Buck | `[Chưa có đủ thông tin - Cần nhóm tự xác nhận: có cần firmware biết nguồn hay không; nếu có thì GPIO nào]` |

---

# Phần 2: Phân tích & Lựa chọn Linh kiện từng khối

## 2.1. Khối Bảo vệ Đầu vào (Input Protection)

### 2.1.1. Schottky nối tiếp vs P-MOSFET ideal diode

Dòng vào jack suy từ công suất: $I_{IN} = \dfrac{5\,V \cdot I_{5V}}{\eta \cdot V_{IN}}$, với $\eta$ = 88 % (worst) / 90 % (typ) `[Giả định: ƯL]`.

| Điểm làm việc | $I_{IN}$ | SS34: $V_F$ `[Giả định: DS]` | SS34: $P_{loss}=V_F I$ | P-MOS $R_{DS(on)}$ = 60 mΩ `[Giả định: DS]`: $\Delta V = I R$ | P-MOS: $P_{loss}=I^2R$ |
|---|---|---|---|---|---|
| 12 V, Typ (661 mA) | 0.31 A | 0.35 V | 0.11 W | 0.019 V | **0.006 W** |
| 12 V, Worst (1986 mA) | 0.94 A | 0.42 V | 0.39 W | 0.056 V | **0.053 W** |
| 6 V, Worst (1986 mA) | 1.88 A | 0.50 V | **0.94 W** | 0.113 V | **0.21 W** |

Tỷ lệ tổn hao SS34 / P-MOS: 18× · 7.4× · 4.5×.

**Hệ quả lên Buck ở góc 6 V** (đã trừ PTC 40 mΩ × 1.88 A = 0.075 V `[Giả định: DS]`):

| Phương án | $V_{IN}$ tại Buck (adapter 6.0 V) | Adapter −5 % (5.7 V) |
|---|---|---|
| SS34 | 6.0 − 0.50 − 0.075 = **5.43 V** | 5.13 V |
| P-MOS | 6.0 − 0.113 − 0.075 = **5.81 V** | 5.51 V |

| | **Schottky SS34** | **P-MOS ideal diode** |
|---|---|---|
| **Lợi** | 1 linh kiện, không có lỗi mạch gate, rẻ | Sụt áp nhỏ (0.113 V so với 0.50 V), tỏa nhiệt ≤ 0.21 W, giữ thêm ≈ 0.38 V cho Buck ở jack 6 V ($V_{IN}$ đạt 5.81 V, adapter −5 % vẫn đạt 5.51 V) |
| **Hại** | Tỏa tới 0.94 W trên 1 linh kiện SMC; làm $V_{IN}$ xuống 5.43 V — sát $V_{IN\_MIN}$ nhiều Buck | Cần Zener + R gate; sai chiều Drain/Source sẽ mất bảo vệ |

**Khuyến nghị: P-MOSFET** (loại −30 V, $V_{GS(max)}$ ±12 V, $R_{DS(on)}$ ≤ 60 mΩ tại $V_{GS}$ = −4.5 V, ví dụ AO3401A `[Giả định: DS]`). Cấu hình: **Drain nối `VIN_FUSED`, Source nối `VIN_PROT`** (body-diode dẫn theo chiều vào tải); Gate qua 100 kΩ xuống GND.

### 2.1.2. Ràng buộc $V_{GS}$ của P-MOSFET

Gate nối GND qua trở → $V_{GS} = -V_{IN}$ (xấp xỉ). Với $V_{IN}$ tối đa 17 V `[Giả định: ƯL]`:

$$|V_{GS}| = 17\ \text{V} > 12\ \text{V} \Rightarrow \text{vượt giới hạn, hỏng oxit Gate}$$

**Vì sao chia áp thuần túy không cứu được:** $V_{GS}$ phải nằm trong khoảng [4.5 V (đủ mở), 12 V (giới hạn)] → tỷ lệ cho phép 12 / 4.5 = **2.67**. Nhưng dải $V_{IN}$ là 17 / 6 = **2.83** > 2.67 → không có tỷ số chia nào thỏa cả hai đầu.

**Giải pháp: Zener kẹp.** Zener 10 V `[Giả định: DS — ví dụ BZX84-C10]` giữa Source (cực K) và Gate (cực A), Gate qua 100 kΩ xuống GND:

| $V_{IN}$ | $V_{GS}$ | Zener | Dòng Zener |
|---|---|---|---|
| 6 V | −5.9 V (đủ mở) | không dẫn | 0 |
| 12 V | −10 V | bắt đầu kẹp | ≈ 20 µA |
| 17 V | −10 V (an toàn, < 12 V) | kẹp | $(17-10)/100\,k\Omega$ = 70 µA, 0.7 mW |

### 2.1.3. Ràng buộc chéo TVS ↔ IC Buck

Chuỗi điều kiện phải thỏa đồng thời:

$$V_{IN,no\text{-}load,max} \;\le\; V_{RWM} \;<\; V_{BR,min} \;\ldots\; V_{C} \;<\; V_{IN\_MAX,\,Buck} \;\;\text{và}\;\; V_C < V_{DS,MOSFET}$$

Số liệu TVS 600 W `[Giả định: DS]`:

| TVS | $V_{RWM}$ | $V_{BR}$ (min) | $V_C$ (max) |
|---|---|---|---|
| SMBJ15A | 15 V | 16.7 V | 24.4 V |
| SMBJ16A | 16 V | 17.8 V | 26.0 V |
| SMBJ18A | 18 V | 20.0 V | 29.2 V |

Kiểm tra với $V_{IN\_MAX}$ vận hành của Buck `[Giả định: DS]` và P-MOS −30 V `[Giả định: DS]`:

| TVS | Hở mạch 17 V có bị rò? | MP1584 (28 V) | TPS54202 (28 V) | AP63205 (32 V) | TPS5430 (36 V) | P-MOS (30 V) |
|---|---|---|---|---|---|---|
| SMBJ15A | **Có** (17 V > 16.7 V, TVS dẫn) | ✅ biên 3.6 V | ✅ 3.6 V | ✅ 7.6 V | ✅ 11.6 V | ✅ 5.6 V |
| SMBJ16A | Rò nhẹ (17 V > $V_{RWM}$ nhưng < 17.8 V) | ✅ 2.0 V | ✅ 2.0 V | ✅ 6.0 V | ✅ 10.0 V | ✅ 4.0 V |
| SMBJ18A | Không | ❌ 29.2 > 28 V | ❌ | ✅ 2.8 V | ✅ 6.8 V | ⚠️ chỉ 0.8 V |

**Khuyến nghị:** nếu dùng MP1584 → **SMBJ16A** (biên 2.0 V ở phía Buck, 4.0 V ở phía P-MOS). Nếu nhóm xác nhận chỉ dùng adapter **ổn áp** 12 V (hở mạch ≤ 12.6 V) thì **SMBJ15A** là đủ và kẹp thấp hơn.
`[Chưa có đủ thông tin - Cần nhóm tự xác nhận: đo điện áp hở mạch thật của adapter; nếu > 17 V thì phải đổi cả TVS lẫn Buck sang loại $V_{IN\_MAX}$ ≥ 36 V]`

Tụ vào của Buck: dùng loại **≥ 35 V** (tụ gốm 25 V mất rất nhiều dung lượng khi có DC-bias và quá sát $V_C$) `[Giả định: ƯL]`.

### 2.1.4. Cầu chì tự phục hồi (PTC)

- **Chọn:** dòng giữ ≥ dòng vào worst × hệ số suy giảm theo nhiệt: $I_{hold} \ge 1.88\,A / 0.8 \approx 2.35\,A$ → **PTC 3 A hold** `[Giả định: ƯL — hold giảm ≈ 20 % ở 60 °C]`, điện áp định mức ≥ 24 V.
- **Sụt áp:** $R \approx 40\,m\Omega$ `[Giả định: DS]` → 0.075 V ở 1.88 A; tỏa 0.14 W.

| | |
|---|---|
| **Lợi** | Bảo vệ đường đồng và adapter khi ngắn mạch phía sau; kết hợp TVS: nếu cắm nhầm adapter 24 V, TVS dẫn mạnh → PTC ngắt, TVS sống sót |
| **Hại** | Thêm 0.075 V sụt áp; **không** bảo vệ xung nhanh (cắt sau hàng giây) — đó là việc của TVS; hold current suy giảm khi nóng |

**Thứ tự đặt:** `Jack → PTC → TVS → P-MOS → Buck`. PTC đứng trước TVS để TVS không bị dòng kéo dài đốt cháy.

---

## 2.2. Khối Hạ áp 5V (5V Step-Down)

### 2.2.1. Bảng so sánh

Điểm tính: $V_{IN}$ = 12 V; Typ = 661 mA; Worst = 1986 mA. Tất cả thông số IC `[Giả định: DS]`, hiệu suất `[Giả định: ƯL]`.

| Tiêu chí | **AMS1117-5.0** (LDO) | **MP1584** | **TPS54202** | **TPS5430** | **AP63205** |
|---|---|---|---|---|---|
| Kiểu | LDO | Buck, cần diode ngoài | Buck đồng bộ | Buck, cần diode ngoài | Buck đồng bộ, **fixed 5 V** |
| Dải $V_{IN}$ | ≈ 6.3–15 V (5 V + dropout 1.1–1.3 V) | 4.5–28 V | 4.5–28 V | **5.5**–36 V | 3.8–32 V |
| Dòng tối đa | 1 A | **3 A** | 2 A | **3 A** | 2 A |
| Tần số đóng cắt | — | 100 kHz–1.5 MHz (hay dùng 1.5 MHz) | 500 kHz | 500 kHz | 1.1 MHz |
| **1) Hiệu suất** | 41.7 % ($=5/12$) | 88 % | 90 % | 90 % | 90 % |
| **2a) Tỏa nhiệt @ Typ 0.661 A** | **4.63 W** ($=7\,V \times 0.661\,A$) | 0.45 W | 0.37 W | 0.37 W | 0.37 W |
| **2b) Tỏa nhiệt @ Worst 1.986 A** | **13.9 W** ($=7\,V \times 1.986\,A$) | 1.35 W | 1.10 W (sát 2 A) | 1.10 W | 1.10 W (sát 2 A) |
| **3) Dòng tải tối đa** | 1 A ❌ | 3 A ✅ biên **51.1 %** | 2 A ⚠️ sát 1.986 A | 3 A ✅ biên **51.1 %** | 2 A ⚠️ sát 1.986 A |
| **4) Linh kiện ngoại vi** (số cái) | ≈ 3 | ≈ 9 (L, D, Cin, Cout, Cboot, R1, R2…) | ≈ 7 | ≈ 9 | **≈ 5** (L, Cin, Cout, Cboot…) |
| **5) Diện tích PCB** `[Giả định: ƯL — gồm cuộn cảm + tụ + diode]` | 60 mm² chưa tính đồng tản nhiệt (cần ≫ 13 W → không khả thi) | ≈ 150 mm² | ≈ 100 mm² | ≈ 220 mm² | ≈ 90 mm² |
| $V_{IN}$ thực tại Buck @ jack 6 V = **5.81 V** | ❌ (< 6.3 V) | ✅ dư 1.31 V | ✅ dư 1.31 V | ⚠️ dư 0.31 V; adapter −5 % → 5.51 V sát 5.5 V ❌ | ✅ dư 2.01 V |
| $V_{IN}$ hở mạch 17 V | ❌ vượt 15 V | ✅ | ✅ | ✅ | ✅ |

Công thức tỏa nhiệt: $P_{diss} = P_{out}\,\dfrac{1-\eta}{\eta}$; riêng LDO $P = (V_{IN}-V_{out})\,I$.

### 2.2.2. Phân tích sâu

1. **LDO 5 V bị loại hoàn toàn.** 13.9 W trên SOT-223 vượt xa khả năng tản (xem mục 2.4: ≈ 1 W đã vượt). Ngoài ra AMS1117-5.0 cần $V_{IN}$ ≥ ≈ 6.3 V nên không bao giờ ra được 5 V khi jack 6 V.
2. **$V_{IN\_MIN}$ và jack 6 V.** Sau PTC + P-MOS còn 5.81 V (5.51 V nếu adapter −5 %). TPS5430 chỉ dư 0.31 V (adapter −5 % còn 5.51 V sát ngưỡng $V_{IN\_MIN} = 5.5\text{ V}$) → loại. MP1584 và TPS54202 dư 1.31 V.
3. **Cảnh báo duty cycle ở 6 V (áp dụng mọi Buck).** Để ra 5.09 V từ 5.81 V cần duty ≈ 88 % *và* bù thêm sụt áp nội trở: $I \times (R_{DS(on)} + R_{DCR}) \approx 1.88\,A \times (0.13 + 0.05)\,\Omega \approx 0.34\,V$ `[Giả định: ƯL]` → cần ≥ ≈ 5.43 V. Ở góc **6 V + tải Worst-case** Buck hoạt động an toàn hơn nhiều so với trước, `+5V` duy trì ổn định.
   `[Chưa có đủ thông tin - Cần nhóm tự xác nhận: đọc datasheet MP1584 các mục "Maximum Duty Cycle", "Minimum Off-Time", "Dropout" ở fsw đã chọn; hạ fsw xuống ≈ 500 kHz nếu cần tăng duty tối đa]`
4. **Tần số đóng cắt (trade-off).**

   | fsw cao (1.5 MHz) | fsw thấp (500 kHz) |
   |---|---|
   | Cuộn cảm nhỏ: $L = \dfrac{V_{out}(1-V_{out}/V_{in})}{f_{sw}\,\Delta I}$ = $\dfrac{5 \times 0.706}{1.5\,MHz \times 1.05\,A}$ ≈ **2.2 µH** (ΔI = 35 % × 3 A) | Cuộn lớn gấp ≈ 3 lần |
   | Tổn hao đóng cắt tăng, hiệu suất thấp hơn vài % | Hiệu suất cao hơn, duty tối đa lớn hơn |
   | Thời gian bật tối thiểu khó khi $V_{IN}$ = 17 V: $t_{on}$ = 0.29/1.5 MHz ≈ 196 ns | Dễ hơn |

   Chọn cuộn 4.7 µH (ΔI ≈ 0.5 A), **$I_{sat}$ ≥ 4.5 A** (= 1.3 × (3 A + ΔI/2)) `[Giả định: ƯL]`.
5. **Fixed 5 V vs Adjustable.**

   | | Fixed (AP63205) | Adjustable (MP1584) |
   |---|---|---|
   | **Lợi** | Bớt 2 trở FB, không sai số trở, ít lỗi nối FB | Tinh chỉnh được áp ra: R2 = 10 kΩ, R1 = 53.6 kΩ → $0.8\,V \times (1+5.36)$ = **5.09 V** `[Giả định: DS — $V_{FB}$ = 0.8 V]` để bù sụt áp power-path và ưu tiên Buck hơn VBUS |
   | **Hại** | Không chỉnh được; mất ưu thế khi cần 5.09 V | 2 trở 1 % → sai số áp ra ≈ ±3 % `[Giả định: ƯL]`; đường FB nhạy nhiễu |

### 2.2.3. Khuyến nghị chốt

**MP1584 bản adjustable, đặt 5.09 V.** Là phương án tối ưu vừa đạt dòng 3 A (biên an toàn thực tế **51.1 %** so với mức Worst-case 1986 mA, vượt xa yêu cầu $\ge 30\%$ `[Đề]`) vừa có $V_{IN\_MIN}$ = 4.5 V, ngay cả khi NeoPixel sáng trắng 100 %, không cần firmware giới hạn độ sáng LED.
Điều kiện kèm theo: Xác nhận duty cycle/dropout MP1584 như mục trên.

---

## 2.3. Khối Chọn nguồn USB / DC (Power-Path)

### 2.3.1. Nguy cơ dòng chảy ngược

Khi cắm đồng thời 2 nguồn mà không có cách ly:

- **Buck → cổng USB máy tính.** Giả định điện trở đường dẫn $R_{path}$ = 0.125 (cáp) + 0.2 (PTC) + 0.05 (đường mạch) = 0.375 Ω `[Giả định: ƯL]`. Với Buck 5.09 V và VBUS 4.9 V:
  $$I_{rev} = \frac{5.09-4.9}{0.375} \approx 0.51\ \text{A đẩy ngược vào cổng USB}$$
  Cổng USB máy tính không được thiết kế để nhận dòng, có thể ngắt bảo vệ hoặc hỏng.
- **USB → đầu ra Buck.** Khi chỉ có USB, `+5V` nạp ngược qua cuộn cảm vào diode thân của MOSFET trên trong Buck, làm nổi `VIN_PROT` ≈ 4 V → có thể kích Buck chạy bất thường. Giảm nhẹ bằng chia áp EN/UVLO (chỉ cho Buck chạy khi `VIN_PROT` > ≈ 5.3 V).

### 2.3.2. So sánh 3 phương án (thêm TPS2113A làm tham chiếu)

Điểm tính: Buck đặt 5.09 V; PTC USB 0.2 Ω, cáp 0.125 Ω; chế độ USB: dòng 0.5 A (giới hạn host).

| Tiêu chí | **2 Schottky OR-ing** (SS34 / SS14) | **P-MOS tự ngắt** (nhánh USB) | **LM66200** (dual ideal diode) | **TPS2113A** (power-mux) |
|---|---|---|---|---|
| Chặn dòng ngược | Có (mỗi nhánh 1 diode) | Có cho USB; Buck nối thẳng, cần EN/UVLO | Có (chặn ngược thật) | Có |
| Thông số chính `[Giả định: DS]` | $V_F$ 0.40–0.52 V | $R_{DS(on)}$ 60 mΩ | $R_{ON}$ ≈ 40 mΩ, 2.5 A, 1.6–5.5 V | $R_{DS(on)}$ ≈ 84 mΩ, **2 A**, 2.8–5.5 V |
| Jack, Typ 0.661 A: sụt áp / tổn hao / `+5V` | 0.36 V / 0.24 W / **4.73 V** | 0 / 0 / 5.09 V | **0.026 V / 0.017 W / 5.06 V** | 0.056 V / 0.037 W / 5.03 V |
| Jack, Worst 1.986 A: sụt áp / tổn hao / `+5V` | 0.50 V / **0.99 W** / **4.59 V** | 0 / 0 / 5.09 V | **0.080 V / 0.158 W / 5.01 V** | 0.167 V / — / ⚠️ 2 A sát 1.986 A |
| USB 0.5 A: `+5V` (VBUS = 5.0 V → 4.775 V sau cáp + PTC) | 4.43 V (SS14, $V_F$ 0.35 V) | 4.75 V | 4.76 V | 4.73 V |
| USB 0.5 A: `+5V` (VBUS = 4.75 V → 4.525 V) | **4.18 V** | 4.50 V | 4.51 V | 4.48 V |
| Ưu tiên khi cắm cả hai | Nguồn điện áp cao hơn thắng | Jack thắng (cần thiết kế gate) | Nguồn điện áp cao hơn thắng `[Chưa có đủ thông tin - Cần nhóm tự xác nhận: đọc mục chọn kênh của LM66200]` | Có chân chọn ưu tiên `[Chưa có đủ thông tin - Cần nhóm tự xác nhận: bảng chế độ MODE/CTRL]` |
| Biên dòng so với 1.986 A | cần SS34 (3 A, biên 51.1 %) | — | 2.5 A, biên **25.9 %** | ❌ 2 A ≈ 1.986 A (chỉ 0.7 %) |
| Độ phức tạp | Thấp nhất (2 diode) | Trung bình (P-MOS + trở/Zener gate + EN) | Thấp (1 IC + cap) | Thấp (1 IC + R_ILIM + cap) |

### 2.3.3. Tác động thực tế lên Servo và NeoPixel

| Điều kiện | `+5V` | % so với 5.0 V | Servo SG90 (4.8–6 V `[Giả định: DS]`, mô-men ∝ V `[Giả định: ƯL]`) | NeoPixel (VDD 3.5–5.3 V `[Giả định: DS]`) |
|---|---|---|---|---|
| Jack, Worst, **Schottky OR** | 4.59 V | 91.8 % | Dưới 4.8 V; mô-men −8.2 % | Vẫn chạy; khi servo khởi động sụt thêm → LED **xanh dương/trắng** hụt trước, trắng ngả vàng |
| Jack, Worst, **LM66200** | 5.01 V | 100.2 % | Đúng định mức | Màu ổn định |
| USB (VBUS 4.75 V), **Schottky OR** | **4.18 V** | 83.5 % | Mô-men −16.5 %; **giật/reset** vì host chỉ 500 mA | Ngả vàng, hụt xanh dương rõ |
| USB (VBUS 4.75 V), **LM66200** | 4.51 V | 90 % | −10 %; vẫn bị giới hạn 500 mA của host | Chấp nhận được |

- **Xung khởi động servo** (đỉnh tới 650 mA/con `[Giả định: KB]`): với Schottky, $V_F$ ≈ 0.55 V → `+5V` ≈ 4.54 V. Với Buck AP63203 ($V_{IN\_MIN} = 3.8\text{ V}$), rail 3.3 V vẫn ổn định hoàn toàn, không lo brownout reset như khi dùng LDO đời cũ.
- **Mạch điều khiển & Level Shifter cho NeoPixel:**
  - Theo Bảng 6-3 Datasheet ESP32-S3, mức điện áp ngõ ra của GPIO là: $V_{OH} \ge 0.8 \times VDD = 0.8 \times 3.3\text{ V} = \mathbf{2.64\ V}$ (ở $I_{OH} = 40\text{ mA}$ typ, PAD_DRIVER = 3); $V_{OL} \le 0.1 \times VDD = \mathbf{0.33\ V}$ (ở $I_{OL} = 28\text{ mA}$ typ). Trong khi đó, NeoPixel WS2812B (cấp nguồn `+5V` = 5.0 V) yêu cầu điện áp vào mức cao $V_{IH} = 0.7 \times V_{DD} = 0.7 \times 5.0\text{ V} = \mathbf{3.5\ V} > 3.3\text{ V}$. Vì $V_{OH}$ tối đa của ESP32-S3 (ngay cả danh định 3.3 V vẫn < 3.5 V) ⇒ **bắt buộc phải có Level-Shifter**.
  - LED dùng nguồn `+5V`, nhận tín hiệu từ **`GPIO45`** thông qua IC chuyển mức 1 kênh **`74AHCT1G125`** (cấp nguồn `+5V`, dịch mức `3.3 V → 5 V` vì $V_{IH} = 0.7 \times 5.0\text{ V} = 3.5\text{ V} > 3.3\text{ V}$, ngõ vào tương thích TTL $V_{IH\_min} = 2.0\text{ V}$). Kèm tụ lọc gốm `100 nF` đặt sát chân nguồn VCC của LED và trở nối tiếp `330 Ω` trên đường Data để triệt tiêu sóng phản xạ.
- **Lưu ý Strapping Pin (`GPIO45`):**
  - Chân `GPIO45` là chân strapping chọn điện áp `VDD_SPI` (cần mức LOW khi khởi động để cấp 3.3 V cho Flash nội/ngoài; nếu bị kéo HIGH sẽ chuyển sang 1.8 V làm Flash không chạy và treo chip).
  - **Quy tắc bắt buộc:** **Tuyệt đối cấm gắn trở kéo lên (pull-up)** trên đường tín hiệu `GPIO45`. Ngõ vào của `74AHCT1G125` là ngõ vào High-Z, không kéo chân lên mức cao lúc reset nên an toàn.
- **Giải thích loại bỏ phương án cấp nguồn `+3V3` cho NeoPixel:**
  - WS2812B yêu cầu dải nguồn hoạt động `3.5–5.3 V` (cấp `3.3 V` gây sụt áp, hụt ánh sáng LED xanh dương do $V_F$ cao và rất dễ nhấp nháy khi ESP32 phát Wi-Fi làm dao động rail 3.3 V).
  - Đồng thời, việc chuyển `60 mA` sang nhánh `+3V3` hoàn toàn không giúp bỏ được Buck 5 V (vẫn cần cấp cho 4 servo và header 5V); giữ NeoPixel ở rail 5V giúp tận dụng tối đa Buck 5V MP1584 và tránh tạo thêm tải đóng cắt lên Buck 3.3 V.

### 2.3.4. Khuyến nghị chốt

**LM66200 làm ideal-diode OR-ing**, Buck đặt 5.09 V. Lý do cốt lõi: giữ `+5V` ≥ 5.0 V (đạt 5.01 V) ở dòng worst (Schottky chỉ còn 4.59 V) và chặn ngược thật sự.

Rủi ro còn lại (phải kiểm tra datasheet):
1. Biên dòng đạt **25.9 %** so với tải Worst-case 1.986 A ($2.5\text{ A} / 1.986\text{ A} - 1 \approx 25.9\%$, tăng mạnh từ mức 8 % / 12.7 % trước đây); hoàn toàn đáp ứng tốt vì 1.986 A là tổng tải cực đại đồng thời của toàn hệ thống.
2. Ưu tiên khi cắm cả hai: nếu datasheet xác nhận "điện áp cao hơn thắng", Buck 5.09 V chỉ hơn VBUS điển hình (5.0 V) 90 mV nhưng VBUS có thể tới 5.25 V → USB có thể thắng khi cắm cả hai. Khi đó chuyển sang TPS2113A (có ưu tiên) *và* hạ tải Worst-case xuống ≤ 1.54 A ($=2\,A/1.3$), hoặc dùng cặp P-MOS tự ngắt.
3. **Phương án dự phòng cho bản thử đầu tiên:** 2 Schottky SS34 — đơn giản, an toàn, chấp nhận `+5V` ≈ 4.59 V.

### 2.3.5. Linh kiện bắt buộc cho USB-C

| Linh kiện | Giá trị | Vai trò |
|---|---|---|
| Điện trở CC1, CC2 | **5.1 kΩ** ±1 %, mỗi chân **một** trở xuống GND | Báo "thiết bị sink" cho host; cáp C-to-C **không có điện** nếu thiếu Rd |
| IC ESD | **USBLC6-2SC6** trên D+/D− `[Giả định: DS — điện dung ≈ 3.5 pF, hợp Full-Speed 12 Mb/s]` | Bảo vệ ESD cho GPIO19/20 |
| PTC VBUS | ≈ 0.75 A hold `[Giả định: ƯL]` (lớn hơn 500 mA mặc định) | Chống ngắn mạch phía board vào cổng USB |
| Tụ VBUS | 10 µF + 100 nF | Lọc/hot-plug |
| Shell (tùy chọn) | 1 MΩ ‖ 4.7 nF tới GND `[Giả định: ƯL]` | Chống nhiễu vỏ |

---

## 2.4. Khối Ổn áp 3.3V — Buck Đồng bộ AP63203

### 2.4.1. Vì sao chuyển đổi từ LDO sang Buck đồng bộ AP63203

Trước đây, thiết kế dự kiến dùng LDO 3.3 V (AP2114H-3.3 hoặc AMS1117-3.3). Tuy nhiên, phân tích sâu cho thấy LDO gặp 3 giới hạn nghiêm trọng. Nhóm quyết định chuyển toàn bộ khối 3.3 V sang IC hạ áp xung đồng bộ **AP63203** (bản fixed `3.3 V`, dòng ra tối đa `2 A`, dải $V_{IN}$ `3.8–32 V`, tần số đóng cắt $f_{sw} = 1.1\text{ MHz}$ `[Giả định: DS]`, hiệu suất $\eta = 90\%$ `[Giả định: ƯL]`). 

Ba lý do kỹ thuật cốt lõi:
1. **Triệt tiêu hoàn toàn quá nhiệt ($T_J$ mát mẻ):** 
   - Trên LDO, ở tải đỉnh 853 mA (burst Wi-Fi), công suất tỏa nhiệt là $P_{loss} = (5.1 - 3.3)\text{ V} \times 0.853\text{ A} + 5.1\text{ V} \times 5\text{ mA} = \mathbf{1.56\text{ W}}$, đẩy $T_J$ lên $134\text{--}181^\circ\text{C}$ (vượt ngưỡng chịu đựng $125^\circ\text{C}$ của IC).
   - Với Buck đồng bộ **AP63203**, công suất tỏa nhiệt ở cùng tải đỉnh 853 mA giảm xuống chỉ còn:
     $$P_{loss} = P_{out} \cdot \frac{1 - \eta}{\eta} = 3.3\,\text{V} \times 0.853\,\text{A} \times \frac{1 - 0.90}{0.90} \approx \mathbf{0.31\ W}$$
   - Ở chế độ tải kéo dài (473 mA), Buck chỉ tỏa **0.17 W** (so với 0.88 W của LDO); ở Typical (192 mA), Buck chỉ tỏa **0.07 W** (so với 0.37 W của LDO). Nhiệt độ mối nối $T_J$ của AP63203 chỉ tăng thêm $\approx 10\text{--}15^\circ\text{C}$ so với môi trường, loại bỏ hoàn toàn yêu cầu phủ mảng đồng tản nhiệt $\ge 600\text{ mm}^2$ dưới tab IC.
2. **Tiết kiệm dòng cho rail `+5V` và nguồn USB:**
   - LDO rút dòng từ `+5V` bằng đúng dòng ra ở 3.3 V ($I_{in} \approx I_{out}$), tiêu tốn tới 858 mA ở đỉnh và 197 mA ở Typical.
   - Buck đồng bộ AP63203 chuyển đổi theo nguyên lý bảo toàn năng lượng với $\eta = 90\%$, dòng rút từ `+5V` chỉ còn:
     $$I_{in} = \frac{3.3\,\text{V} \cdot I_{3V3}}{0.90 \times 5.0\,\text{V}} \approx 0.733 \times I_{3V3}$$
     - Mức Typical ($I_{3V3} = 192\text{ mA}$): chỉ rút **141 mA** (tiết kiệm **56 mA**).
     - Mức Worst-case ($I_{3V3} = 853\text{ mA}$): chỉ rút **626 mA** (tiết kiệm **232 mA**).
   - Sự sụt giảm 232 mA này giảm trực tiếp áp lực gánh tải lên Buck 5 V (MP1584), giúp MP1584 đạt biên an toàn lên tới **51.1 %**; đồng thời giúp chế độ USB dư thêm 56 mA headroom.
3. **An toàn điện áp đầu vào ở góc USB ($V_{IN\_MIN} = 3.8\text{ V}$):**
   - Khi cắm USB, điện áp `+5V` sau power-path có thể xuống tới 4.51 V (hoặc 4.18 V nếu dùng Schottky). LDO cần dropout 1.1–1.3 V nên headroom 1.21 V rất sát ngưỡng mất ổn áp.
   - AP63203 có ngưỡng điện áp vào tối thiểu $V_{IN\_MIN} = \mathbf{3.8\text{ V}}$, thấp hơn rất nhiều so với mức 4.51 V của USB (headroom dư $> 0.7\text{ V}$), đảm bảo ngõ ra 3.3 V luôn ổn định tuyệt đối trong mọi điều kiện nguồn USB.

### 2.4.2. Bảng so sánh thông số kỹ thuật: LDO (AP2114H-3.3) vs Buck Đồng bộ (AP63203)

| Tiêu chí | **AP2114H-3.3** (LDO) | **AP63203** (Buck đồng bộ fixed 3.3 V) | Nhận xét ưu thế của AP63203 |
|---|---|---|---|
| Kiểu ổn áp | Tuyến tính (LDO) | Đóng cắt đồng bộ (Sync Buck) | Tích hợp sẵn cặp FET High/Low-side, không cần diode Schottky ngoài |
| Dải $V_{IN}$ | 4.4–5.5 V (bị giới hạn bởi dropout) | **3.8–32 V** | Dải áp rộng, không ngại sụt áp nguồn USB |
| Dòng tải tối đa | 1.0 A `[Giả định: DS]` | **2.0 A** `[Giả định: DS]` | Dư gấp 2.3× so với đỉnh 853 mA (biên an toàn **134 %**) |
| Tần số đóng cắt | — | **1.1 MHz** `[Giả định: DS]` | Tần số cao, cho phép dùng cuộn cảm nhỏ |
| Hiệu suất $\eta$ ở $V_{IN}=5\text{ V}$ | 66 % ($=3.3/5.0$) | **90 %** `[Giả định: ƯL]` | Vượt trội 24 % |
| **Tỏa nhiệt @ Typ 192 mA** | 0.37 W ($T_J \approx 62\text{--}73^\circ\text{C}$) | **0.07 W** ($T_J \approx T_A + 3^\circ\text{C}$) | Mát hoàn toàn |
| **Tỏa nhiệt @ Kéo dài 473 mA** | 0.88 W ($T_J \approx 93\text{--}119^\circ\text{C}$ ⚠️) | **0.17 W** ($T_J \approx T_A + 8^\circ\text{C}$) | Rất an toàn |
| **Tỏa nhiệt @ Đỉnh 853 mA** | **1.56 W** ($T_J \approx 134\text{--}181^\circ\text{C}$ ❌) | **0.31 W** ($T_J \approx T_A + 15^\circ\text{C}$ ✅) | **Giảm 80 % nhiệt lượng**, triệt tiêu nguy cơ quá nhiệt |
| Dòng rút từ `+5V` @ Worst | 858 mA | **626 mA** | Tiết kiệm **232 mA** cho rail 5 V |
| Yêu cầu tản nhiệt PCB | Bắt buộc mảng đồng $\ge 600\text{ mm}^2$ | Chỉ cần pad đồng tiêu chuẩn IC | Giải phóng diện tích layout |

### 2.4.3. Đánh đổi kỹ thuật & Giải pháp thiết kế chống nhiễu RF cho ESP32-S3

Việc chuyển từ LDO sang Buck mang lại ưu thế tuyệt đối về nhiệt và dòng, nhưng đánh đổi lại là sự xuất hiện của gợn sóng đóng cắt (switching ripple) và trường điện từ phát xạ từ cuộn cảm ở tần số $f_{sw} = 1.1\text{ MHz}$. 

Để đảm bảo hiệu năng RF của module **ESP32-S3-WROOM-1** (độ nhạy RX Wi-Fi và công suất phát TX) không bị suy giảm, nhóm áp dụng các giải pháp phần cứng bắt buộc:
1. **Linh kiện thụ động ngoại vi:**
   - Cuộn cảm và tụ vào: chọn theo khuyến nghị của nhà sản xuất Diodes Inc. `[Chưa có đủ thông tin - Cần nhóm tự xác nhận: mở datasheet AP63203 để lấy đúng trị số cuộn cảm (thường 2.2–3.3 µH) và tụ vào/ra chuẩn]`.
   - Cuộn cảm phải có dòng bão hòa $I_{sat} \ge 1.3 \times 2\text{ A} \approx 2.6\text{ A}$ và điện trở một chiều DCR thấp.
2. **Cấu hình tụ lọc ngõ ra triệt tiêu gợn sóng:**
   - Ngay tại ngõ ra của AP63203: đặt song song **$2\times 22\,\mu\text{F}$ tụ gốm MLCC** (loại X5R/X7R, điện áp định mức $\ge 10\text{ V}$) để giảm tổng trở kháng cao tần và gìm gợn sóng đóng cắt xuống dưới $20\text{ mV}_{p-p}$.
   - Tại chân cấp nguồn của MCU: bố trí cặp tụ **$10\,\mu\text{F} + 100\text{ nF}$** đặt sát ngay chân `3V3` của module ESP32-S3 để lọc triệt để xung nhiễu phát thanh tần cao.
3. **Quy tắc bố trí Layout chống bức xạ nhiễu:**
   - **Vòng nóng (Hot Loop):** Thu hẹp tối đa diện tích vòng lặp dòng điện xung tần số cao bao gồm: chân VIN của AP63203 → tụ hồi tiếp Cin → chân GND của IC.
   - **Nút SW:** Đường nối chân SW sang cuộn cảm phải cực ngắn, diện tích đồng vừa đủ tải dòng, không phủ đồng rộng để tránh biến thành ăng-ten phát xạ E-field.
   - **Khoảng cách ly RF:** Cụm Buck 3.3 V (IC, cuộn cảm, tụ) phải bố trí cách xa khu vực ăng-ten PCB trên đầu module ESP32-S3, đặt ở phía đối diện của board mạch; mặt phẳng GND liền khối (solid ground plane) nằm ngay dưới cụm Buck ở lớp Layer 2 để chắn từ trường phát xạ.

Kiểm tra sụt áp tải bước nhảy 400 mA (thời gian đáp ứng Buck $\approx 10\,\mu\text{s}$, dung lượng lọc ngõ ra $C_{out} = 2 \times 22\,\mu\text{F} = 44\,\mu\text{F}$):

$$\Delta V \approx \frac{\Delta I \cdot \Delta t}{C_{out}} = \frac{0.4 \times 10\,\mu s}{44\,\mu F} \approx 0.09\ \text{V} \Rightarrow V_{min} \approx 3.21\ \text{V}$$

→ Trên ngưỡng nguồn tối thiểu 3.0 V của ESP32-S3 (xác nhận theo Datasheet ESP32-S3 Bảng 6-2: dải nguồn khuyến nghị $VDD33$ từ **3.0 V đến 3.6 V**, danh định 3.3 V; Bảng 6-1 giới hạn tuyệt đối: **-0.3 V đến 3.6 V**).

---

# Phần 3: Bảng Tổng kết Chốt Phương án & Checklist Trước khi Vẽ Schematic

## 3.1. Bảng chốt

| Khối | Phương án đề xuất | Linh kiện tiêu biểu | Lý do cốt lõi |
|---|---|---|---|
| Bảo vệ đầu vào | PTC → TVS → P-MOS ideal diode (Zener clamp Gate) | PTC 3 A hold; **SMBJ16A**; P-MOS −30 V (AO3401A); Zener 10 V; R_G 100 kΩ | Tổn hao 0.21 W thay vì 0.94 W; $V_{GS}$ không vượt 12 V; $V_C$ = 26 V < 28 V của Buck |
| Hạ áp 5 V | Buck adjustable ≥ 3 A, $V_{IN\_MIN}$ ≤ 4.5 V | **MP1584**, L = 4.7 µH ($I_{sat}$ ≥ 4.5 A), FB 53.6 kΩ / 10 kΩ → 5.09 V | Duy nhất đạt cả 3 A (biên an toàn **51.1 %** so với worst 1.986 A) và $V_{IN\_MIN}$ thấp; LDO 5 V tỏa 13.9 W |
| Chọn nguồn | Ideal-diode OR-ing | **LM66200** (dự phòng: 2× SS34) | `+5V` 5.01 V ở 1.986 A (Schottky: 4.59 V); biên dòng **25.9 %**; chặn dòng ngược vào USB |
| USB-C | Rd + ESD + PTC | 2× 5.1 kΩ; **USBLC6-2SC6**; PTC ≈ 0.75 A | C-to-C cần Rd; bảo vệ D+/D− |
| Ổn áp 3.3 V | Buck fixed 3.3 V đồng bộ | **AP63203** (2 A, fixed 3.3 V, $f_{sw} = 1.1\text{ MHz}$) | Tỏa nhiệt đỉnh chỉ **0.31 W** (so với 1.56 W của LDO), giảm dòng vào 5 V xuống **626 mA** (tiết kiệm 232 mA), $V_{IN\_MIN} = 3.8\text{ V}$ an toàn cho USB |
| NeoPixel | 1× NeoPixel WS2812B onboard (GPIO45) | **74AHCT1G125** + R 330 Ω + C 100 nF | Cấp nguồn `+5V`, $V_{IH} = 3.5\text{ V} > 3.3\text{ V}$ của ESP32; GPIO45 không gắn pull-up (strapping pin VDD_SPI) |

## 3.2. Danh sách `[Chưa có đủ thông tin]` nhóm cần điền

| # | Hạng mục | Cách xác nhận |
|---|---|---|
| 1 | Loại adapter và điện áp hở mạch | Đo bằng đồng hồ vạn năng khi không tải; quyết định TVS (SMBJ15A/16A/18A) |
| 2 | Loại servo thật và số servo chạy cùng lúc | Datasheet servo; đo dòng bằng power meter |
| 3 | Dòng thật từng module Grove | Datasheet module, hoặc đo khi chạy |
| 4 | Dòng cho phép trên chân `+5V` của header | Nhóm tự đặt giới hạn, ghi vào README |
| 5 | Nhiệt độ môi trường $T_A$ và nhiệt độ vỏ hộp | Quyết định hộp kín/hở; đo nhiệt độ thực tế khi chạy tải |
| 6 | GPIO cho `VBUS_SENSE`, `PWR_SRC`/PG | Phối hợp với nhóm MCU khi chốt pin-map |
| 7 | Các số `[Giả định: DS]` của MP1584, LM66200, AP63203, SMBJ, AO3401A | Mở datasheet, đối chiếu từng giá trị, đặc biệt: trị số cuộn cảm/tụ khuyến nghị của AP63203; duty tối đa/dropout MP1584; kiểu chọn kênh LM66200; $V_{GS(max)}$ P-MOS |

## 3.3. Checklist quan trọng trước khi vẽ Schematic và layout

1. **Duty/dropout ở 6 V:** xác nhận Buck vẫn ra ≥ 5.0 V khi $V_{IN}$ = 5.81 V, tải Worst-case (mục 2.2.2). Nếu không đạt, ghi rõ trong tài liệu: "jack ≥ 7.5 V để chạy tải nặng".
2. **TVS ↔ Buck ↔ P-MOS:** kiểm $V_{RWM}$ ≥ điện áp hở mạch adapter; $V_C$ < $V_{IN\_MAX}$ của Buck *và* < $V_{DS}$ của P-MOS; tụ vào Buck ≥ 35 V.
3. **P-MOS ideal diode:** Drain = `VIN_FUSED`, Source = `VIN_PROT`; có Zener 10 V giữa Source–Gate; thử cắm ngược cực trên mô phỏng/bảng mạch thử.
4. **Layout Buck 5 V (MP1584):** vòng nóng (Cin → IC → diode/cuộn) nhỏ nhất; Cin sát chân VIN–GND; đường FB đi xa nút `SW_BUCK` và lấy ở tụ ra; mặt phẳng GND liền dưới Buck; cuộn cảm $I_{sat}$ ≥ 4.5 A.
5. **Buck 3.3 V (AP63203) & Chống nhiễu RF:**
   - Bố trí $2\times 22\,\mu\text{F}$ tụ gốm MLCC ở ngõ ra Buck AP63203 để dập gợn sóng switching;
   - Bố trí cặp tụ $10\,\mu\text{F} + 100\text{ nF}$ đặt sát chân VDD33 của module ESP32-S3;
   - Vòng lặp nóng (Cin → chân VIN/GND của AP63203) nhỏ nhất; đặt toàn bộ cụm Buck 3.3 V cách xa khu vực ăng-ten PCB của ESP32-S3 để chống suy giảm độ nhạy Wi-Fi/Bluetooth.
6. **NeoPixel và Strapping Pin (GPIO45):** kiểm tra tuyệt đối **không gắn điện trở kéo lên (pull-up)** trên đường tín hiệu `GPIO45` trước ngõ vào IC `74AHCT1G125` (chân strapping chọn điện áp `VDD_SPI`, phải giữ LOW khi chip khởi động/reset để Flash chạy ở 3.3 V). Đảm bảo bố trí tụ lọc gốm 100 nF sát chân VCC của WS2812B và điện trở nối tiếp 330 Ω trên đường Data.
7. **Tụ lọc và chân EN ESP32-S3:** 
   - Tụ lọc: 10–22 µF + 100 nF đặt sát chân 3V3 của module; mạch reset EN có RC theo khuyến nghị phần cứng của Espressif.
   - Ngưỡng điện áp chân EN (Mục 5 Datasheet): $V_{IH\_nRST} = 0.75 \times VDD$ (Min = 2.475 V tại 3.3 V) đến $VDD + 0.3\text{ V}$ (Max = 3.6 V); $V_{IL\_nRST} = -0.3\text{ V}$ (Min) đến $0.25 \times VDD$ (Max = 0.825 V tại 3.3 V).
   - Nguồn cấp VDD33 (Bảng 6-1 & 6-2): Định mức 3.0 V – 3.6 V (Typ 3.3 V); Giới hạn tuyệt đối -0.3 V đến 3.6 V; Nguồn ngoài phải đảm bảo $I_{VDD} \ge 0.5\text{ A}$.
8. **USB-C:** 5.1 kΩ **riêng** cho CC1 và CC2; USBLC6-2SC6 đặt sát đầu nối; D+/D− đi cặp vi sai ngắn, không qua via thừa; PTC trên VBUS; điểm test `+5V`, `+3V3`, `GND`, `EN`.

---

*Tài liệu này là khung tính toán dựa trên giả định có gắn nhãn. Chỉ chốt linh kiện sau khi đã đối chiếu mọi mục `[Giả định: DS]` với datasheet gốc và điền các mục `[Chưa có đủ thông tin]` bằng số đo thực tế của nhóm.*