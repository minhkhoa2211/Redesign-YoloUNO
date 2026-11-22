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
| 1 | Dòng `+3V3` worst-case **vượt** mốc 800 mA của đề | 853 mA → LDO nên chọn loại **≥ 1 A** |
| 2 | Dòng Buck 5 V cần tới **3.43 A** nếu không giới hạn NeoPixel; **3.01 A** nếu firmware giới hạn NeoPixel ≤ 20 mA/LED | LDO 5 V (AMS1117-5.0) loại ngay: tỏa 16.2 W |
| 3 | Jack 6 V là góc khó nhất: dòng vào lớn, $V_{IN}$ tại Buck chỉ còn ≈ 5.78 V | TPS5430 ($V_{IN\_MIN}$ = 5.5 V) bị loại |
| 4 | TVS phải thỏa chuỗi: $V_{no\text{-}load} \le V_{RWM}$ và $V_C < V_{IN\_MAX}$ của Buck | SMBJ15A có thể rò khi adapter hở mạch 17 V; SMBJ18A kẹp 29.2 V > 28 V của MP1584 |
| 5 | Schottky OR-ing làm tụt `+5V` còn ≈ 4.5 V → LDO đời cũ hết dropout, servo yếu | Ideal-diode IC giữ 5.0 V ở cùng dòng |
| 6 | LDO SOT-223 ở tải đỉnh 853 mA **vượt giới hạn nhiệt** | $T_J$ ≈ 134–180 °C (peak), ≈ 93–119 °C (kéo dài) |

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
| 1 | ESP32-S3-WROOM-1 | 120 mA `[Giả định: ƯL]` — trung bình khi Wi-Fi STA hoạt động | **500 mA** `[Đề]` — đỉnh Wi-Fi | Worst là đỉnh ngắn (burst TX), không kéo dài |
| 2 | 12 cổng Grove (3.3 V) | 60 mA = 4 module × 15 mA `[Giả định: KB]` | 300 mA = 12 × 25 mA `[Giả định: KB]` | `[Chưa có đủ thông tin - Cần nhóm tự xác nhận: dòng thật từng module Grove — đo bằng USB power meter hoặc đọc datasheet module]` |
| 3 | Cổng QWIIC | 10 mA `[Giả định: KB]` | 50 mA `[Giả định: KB]` | 1 cổng, vài cảm biến I²C |
| 4 | LED PWR + D13 (1 kΩ, 0603) | 2 mA | 3 mA | $(3.3-2.1)/1\,\text{k}\Omega \approx 1.2$ mA/LED, $V_F \approx 2.1$ V `[Giả định: DS]`; giả định LED PWR cấp từ `+3V3` `[Giả định: KB]` |
| | **Σ `+3V3`** | **192 mA** | **853 mA** | |
| | *Σ kéo dài (ESP32 trung bình + tải Grove/QWIIC/LED worst)* | — | *473 mA* | Dùng cho tính nhiệt LDO ở mục 2.4 |

> ⚠️ **Phát hiện:** 853 mA > 800 mA (mốc của đề). Nếu giữ đúng 800 mA, nhóm phải giới hạn tổng dòng Grove ≤ 247 mA, hoặc chọn LDO **≥ 1 A** (biên 17 %). Tài liệu này chọn hướng thứ hai.

## 0.3. Ngân sách dòng rail `+5V`

| # | Tải | Typical | Worst-case (W1) | Worst-case có giới hạn NeoPixel (W2) | Cơ sở |
|---|---|---|---|---|---|
| 1 | 4 Servo SG90 | 4 × 100 = **400 mA** | 4 × 250 = **1000 mA** | 1000 mA | `[Giả định: KB — SG90: chạy không tải ≈ 100 mA, có tải ≈ 250 mA]`. Stall 650 mA/con (2600 mA cho 4 con) là **sự cố**, xem ghi chú dưới bảng |
| 2 | NeoPixel WS2812B, 8 LED | 8 × 20 = **160 mA** | 8 × 60 = **480 mA** | 8 × 20 = **160 mA** | `[Giả định: KB — 8 LED; 60 mA/LED là trắng 100 %, 20 mA/LED ≈ 1/3 độ sáng]` |
| 3 | Chân xuất `+5V` trên header | 100 mA | 300 mA | 300 mA | `[Chưa có đủ thông tin - Cần nhóm tự xác nhận: nhóm cho phép shield/module ngoài lấy tối đa bao nhiêu mA từ chân +5V]` |
| 4 | **Đầu vào LDO 3.3 V** = $I_{3V3} + I_{GND}$ | 192 + 5 = **197 mA** | 853 + 5 = **858 mA** | 858 mA | $I_{GND}\approx 5$ mA `[Giả định: DS — AMS1117; LDO hiện đại < 1 mA]` |
| | **Σ `+5V`** | **857 mA** | **2638 mA** | **2318 mA** | |

**Ghi chú stall:** nếu cả 4 servo kẹt cùng lúc + NeoPixel trắng: 2600 + 480 + 300 + 858 = **4238 mA**. Đây là sự cố cơ khí, **không dùng để chọn Buck** (sẽ ra Buck ≥ 5.5 A, phi thực tế); xử lý bằng giới hạn dòng của Buck và firmware dừng servo khi kẹt.

## 0.4. Chứng minh cộng dồn dòng `+3V3` → `+5V`

LDO là bộ ổn áp tuyến tính: dòng ra bằng dòng vào trừ dòng chân GND.

$$I_{LDO,in} = I_{3V3} + I_{GND}$$

Công suất lấy từ `+5V` cho nhánh 3.3 V:

$$P_{in} = 5\,V \cdot I_{LDO,in} = \underbrace{3.3\,V \cdot I_{3V3}}_{\text{tải thật}} + \underbrace{(5-3.3)\,V \cdot I_{3V3}}_{\text{tỏa nhiệt trên LDO}} + 5\,V \cdot I_{GND}$$

Vì vậy **mọi mA ở `+3V3` đều là mA ở `+5V`**:

$$I_{5V,total} = I_{servo} + I_{neo} + I_{hdr} + I_{3V3} + I_{GND}$$

$$\text{W1: } 1000 + 480 + 300 + 853 + 5 = 2638\ \text{mA}$$
$$\text{W2: } 1000 + 160 + 300 + 853 + 5 = 2318\ \text{mA}$$

## 0.5. Dòng tối thiểu cần chốt cho Buck 5 V và LDO 3.3 V

$$I_{Buck,min} = 1.3 \times I_{5V,worst}$$

| Kịch bản | $I_{5V}$ | × 1.3 (hệ số an toàn ≥ 30 % `[Đề]`) | Nhận xét |
|---|---|---|---|
| Typical | 857 mA | — | Chỉ để tính hiệu suất/nhiệt |
| **W1** — không giới hạn NeoPixel | 2638 mA | **3.43 A** | Cần Buck lớp 4–5 A |
| **W2** — firmware giới hạn NeoPixel ≤ 20 mA/LED | 2318 mA | **3.01 A** | Vừa khít Buck 3 A (biên thực tế 29.4 %, tức ≈ 30 %) |
| Stall (tham khảo) | 4238 mA | 5.51 A | Không dùng chọn linh kiện |

- **Buck 5 V:** tối thiểu **3.0 A** nếu nhóm chấp nhận giới hạn NeoPixel bằng firmware (W2); nếu không, **≥ 3.5 A** (lớp 4–5 A).
- **LDO 3.3 V:** worst 853 mA → chọn **≥ 1 A** (mốc đề 800 mA không đủ biên).

## 0.6. Ràng buộc khi chạy chỉ bằng USB

Host USB 2.0 cấp 500 mA `[Đề]`. Trừ phần LDO 3.3 V (197 mA) còn **303 mA** cho servo + NeoPixel + header, trong khi nhu cầu Typical là 400 + 160 + 100 = **660 mA**. Kết luận: khi chỉ cắm USB, firmware **phải** hạn chế servo/NeoPixel.
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
   4× Servo header     NeoPixel (qua level-shifter       [LDO 3.3 V ≥ 1 A]
   + chân xuất +5V     74AHCT125, VCC = +5V)                   │ +3V3
                                                               ▼
                                          ESP32-S3-WROOM-1 · 12× Grove · QWIIC · LED
```

## 1.2. Định nghĩa Net

| Net | Vị trí vật lý | Ý nghĩa / điện áp dự kiến |
|---|---|---|
| `VIN_RAW` | Chân + jack DC, **trước** mọi bảo vệ | 6–12 V (hở mạch tới 17 V `[Giả định: ƯL]`), có thể có xung cắm nóng |
| `VIN_FUSED` | Sau PTC | Như `VIN_RAW` trừ sụt áp PTC (≈ 0.09 V ở 2.2 A `[Giả định: DS]`); TVS đặt tại nút này |
| `VIN_PROT` | Sau P-MOS chống ngược cực, vào chân VIN của Buck | ≈ 5.78 V (tối thiểu, adapter 6 V) → 12 V; **luôn dương**, không bao giờ âm |
| `SW_BUCK` | Nút đóng cắt của Buck | Xung vuông, nút nhiễu lớn nhất — giữ nhỏ, xa FB |
| `+5V_BUCK` | Đầu ra Buck, trước ideal-diode | 5.09 V đặt bằng FB `[Giả định: ƯL]` (xem mục 2.2) |
| `VBUS_RAW` | Chân VBUS của USB-C | 4.75–5.25 V `[Đề]` |
| `VBUS` | Sau PTC USB | Nguồn USB đã bảo vệ, vào kênh IN2 của ideal-diode |
| `+5V` | **Rail hệ thống 5 V** sau khối chọn nguồn | Cấp servo, NeoPixel, header +5V, đầu vào LDO |
| `+3V3` | Đầu ra LDO | Cấp ESP32-S3, Grove, QWIIC, LED |
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
| 12 V, Typ (857 mA) | 0.40 A | 0.35 V | 0.14 W | 0.024 V | **0.010 W** |
| 12 V, W2 (2318 mA) | 1.10 A | 0.42 V | 0.46 W | 0.066 V | **0.073 W** |
| 6 V, W2 (2318 mA) | 2.2 A | 0.50 V | **1.10 W** | 0.132 V | **0.29 W** |

Tỷ lệ tổn hao SS34 / P-MOS: 14× · 6.3× · 3.8×.

**Hệ quả lên Buck ở góc 6 V** (đã trừ PTC 40 mΩ × 2.2 A = 0.09 V `[Giả định: DS]`):

| Phương án | $V_{IN}$ tại Buck (adapter 6.0 V) | Adapter −5 % (5.7 V) |
|---|---|---|
| SS34 | 6.0 − 0.50 − 0.09 = **5.41 V** | 5.11 V |
| P-MOS | 6.0 − 0.13 − 0.09 = **5.78 V** | 5.48 V |

| | **Schottky SS34** | **P-MOS ideal diode** |
|---|---|---|
| **Lợi** | 1 linh kiện, không có lỗi mạch gate, rẻ | Sụt áp nhỏ (0.13 V so với 0.5 V), tỏa nhiệt ≤ 0.29 W, giữ thêm ≈ 0.37 V cho Buck ở jack 6 V |
| **Hại** | Tỏa tới 1.1 W trên 1 linh kiện SMC; làm $V_{IN}$ xuống 5.41 V — sát $V_{IN\_MIN}$ nhiều Buck | Cần Zener + R gate; sai chiều Drain/Source sẽ mất bảo vệ |

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

- **Chọn:** dòng giữ ≥ dòng vào worst × hệ số suy giảm theo nhiệt: $I_{hold} \ge 2.2\,A / 0.8 \approx 2.75\,A$ → **PTC 3 A hold** `[Giả định: ƯL — hold giảm ≈ 20 % ở 60 °C]`, điện áp định mức ≥ 24 V.
- **Sụt áp:** $R \approx 40\,m\Omega$ `[Giả định: DS]` → 0.09 V ở 2.2 A; tỏa 0.19 W.

| | |
|---|---|
| **Lợi** | Bảo vệ đường đồng và adapter khi ngắn mạch phía sau; kết hợp TVS: nếu cắm nhầm adapter 24 V, TVS dẫn mạnh → PTC ngắt, TVS sống sót |
| **Hại** | Thêm 0.09 V sụt áp; **không** bảo vệ xung nhanh (cắt sau hàng giây) — đó là việc của TVS; hold current suy giảm khi nóng |

**Thứ tự đặt:** `Jack → PTC → TVS → P-MOS → Buck`. PTC đứng trước TVS để TVS không bị dòng kéo dài đốt cháy.

---

## 2.2. Khối Hạ áp 5V (5V Step-Down)

### 2.2.1. Bảng so sánh

Điểm tính: $V_{IN}$ = 12 V; Typ = 857 mA; W2 = 2318 mA. Tất cả thông số IC `[Giả định: DS]`, hiệu suất `[Giả định: ƯL]`.

| Tiêu chí | **AMS1117-5.0** (LDO) | **MP1584** | **TPS54202** | **TPS5430** | **AP63205** |
|---|---|---|---|---|---|
| Kiểu | LDO | Buck, cần diode ngoài | Buck đồng bộ | Buck, cần diode ngoài | Buck đồng bộ, **fixed 5 V** |
| Dải $V_{IN}$ | ≈ 6.3–15 V (5 V + dropout 1.1–1.3 V) | 4.5–28 V | 4.5–28 V | **5.5**–36 V | 3.8–32 V |
| Dòng tối đa | 1 A | **3 A** | 2 A | **3 A** | 2 A |
| Tần số đóng cắt | — | 100 kHz–1.5 MHz (hay dùng 1.5 MHz) | 500 kHz | 500 kHz | 1.1 MHz |
| **1) Hiệu suất** | 41.7 % ($=5/12$) | 88 % | 90 % | 90 % | 90 % |
| **2a) Tỏa nhiệt @ Typ 0.857 A** | **6.0 W** ($=7\,V \times 0.857\,A$) | 0.58 W | 0.48 W | 0.48 W | 0.48 W |
| **2b) Tỏa nhiệt @ W2 2.318 A** | **16.2 W** | 1.58 W | không đạt (2 A < 2.318 A) | 1.29 W | không đạt |
| **3) Dòng tải tối đa** | 1 A ❌ | 3 A ✅ biên 29.4 % | 2 A ❌ | 3 A ✅ biên 29.4 % | 2 A ❌ |
| **4) Linh kiện ngoại vi** (số cái) | ≈ 3 | ≈ 9 (L, D, Cin, Cout, Cboot, R1, R2…) | ≈ 7 | ≈ 9 | **≈ 5** (L, Cin, Cout, Cboot…) |
| **5) Diện tích PCB** `[Giả định: ƯL — gồm cuộn cảm + tụ + diode]` | 60 mm² chưa tính đồng tản nhiệt (cần ≫ 16 W → không khả thi) | ≈ 150 mm² | ≈ 100 mm² | ≈ 220 mm² | ≈ 90 mm² |
| $V_{IN}$ thực tại Buck @ jack 6 V = **5.78 V** | ❌ (< 6.3 V) | ✅ dư 1.28 V | ✅ dư 1.28 V | ⚠️ dư 0.28 V; adapter −5 % → 5.48 V < 5.5 V ❌ | ✅ dư 1.98 V |
| $V_{IN}$ hở mạch 17 V | ❌ vượt 15 V | ✅ | ✅ | ✅ | ✅ |

Công thức tỏa nhiệt: $P_{diss} = P_{out}\,\dfrac{1-\eta}{\eta}$; riêng LDO $P = (V_{IN}-V_{out})\,I$.

### 2.2.2. Phân tích sâu

1. **LDO 5 V bị loại hoàn toàn.** 16.2 W trên SOT-223 vượt xa khả năng tản (xem mục 2.4: ≈ 1 W đã vượt). Ngoài ra AMS1117-5.0 cần $V_{IN}$ ≥ ≈ 6.3 V nên không bao giờ ra được 5 V khi jack 6 V.
2. **$V_{IN\_MIN}$ và jack 6 V.** Sau PTC + P-MOS còn 5.78 V (5.48 V nếu adapter −5 %). TPS5430 chỉ dư 0.28 V → loại. MP1584 và TPS54202 dư 1.28 V.
3. **Cảnh báo duty cycle ở 6 V (áp dụng mọi Buck).** Để ra 5.09 V từ 5.78 V cần duty ≈ 88 % *và* bù thêm sụt áp nội trở: $I \times (R_{DS(on)} + R_{DCR}) \approx 2.3\,A \times (0.13 + 0.05)\,\Omega \approx 0.41\,V$ `[Giả định: ƯL]` → cần ≥ ≈ 5.5 V. Ở góc **6 V + tải W2** Buck gần hoặc chạm dropout, `+5V` có thể tụt nhẹ (≈ 4.9–5.0 V).
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

**MP1584 bản adjustable, đặt 5.09 V.** Là phương án duy nhất trong 4 IC vừa đạt dòng 3 A vừa có $V_{IN\_MIN}$ = 4.5 V. Điều kiện kèm theo:
1. Firmware giới hạn NeoPixel ≤ 20 mA/LED (kịch bản W2), hoặc nhóm đổi sang Buck 5 A (ví dụ TPS54540, 4.5–42 V `[Giả định: DS]`) nếu muốn W1.
2. Xác nhận duty cycle/dropout MP1584 như mục trên.

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
| Jack, Typ 0.857 A: sụt áp / tổn hao / `+5V` | 0.40 V / 0.34 W / **4.69 V** | 0 / 0 / 5.09 V | 0.034 V / 0.029 W / 5.06 V | 0.072 V / 0.062 W / 5.02 V |
| Jack, W2 2.318 A: sụt áp / tổn hao / `+5V` | 0.52 V / **1.21 W** / **4.57 V** | 0 / 0 / 5.09 V | 0.093 V / 0.215 W / **5.00 V** | 0.195 V / — / ❌ vượt 2 A |
| USB 0.5 A: `+5V` (VBUS = 5.0 V → 4.775 V sau cáp + PTC) | 4.43 V (SS14, $V_F$ 0.35 V) | 4.75 V | 4.76 V | 4.73 V |
| USB 0.5 A: `+5V` (VBUS = 4.75 V → 4.525 V) | **4.18 V** | 4.50 V | 4.51 V | 4.48 V |
| Ưu tiên khi cắm cả hai | Nguồn điện áp cao hơn thắng | Jack thắng (cần thiết kế gate) | Nguồn điện áp cao hơn thắng `[Chưa có đủ thông tin - Cần nhóm tự xác nhận: đọc mục chọn kênh của LM66200]` | Có chân chọn ưu tiên `[Chưa có đủ thông tin - Cần nhóm tự xác nhận: bảng chế độ MODE/CTRL]` |
| Biên dòng so với 2.318 A | cần SS34 (3 A, biên 29 %) | — | 2.5 A, biên **8 %** | ❌ 2 A < 2.318 A |
| Độ phức tạp | Thấp nhất (2 diode) | Trung bình (P-MOS + trở/Zener gate + EN) | Thấp (1 IC + cap) | Thấp (1 IC + R_ILIM + cap) |

### 2.3.3. Tác động thực tế lên Servo và NeoPixel

| Điều kiện | `+5V` | % so với 5.0 V | Servo SG90 (4.8–6 V `[Giả định: DS]`, mô-men ∝ V `[Giả định: ƯL]`) | NeoPixel (VDD 3.5–5.3 V `[Giả định: DS]`) |
|---|---|---|---|---|
| Jack, W2, **Schottky OR** | 4.57 V | 91 % | Dưới 4.8 V; mô-men −9 % | Vẫn chạy; khi servo khởi động sụt thêm → LED **xanh dương/trắng** hụt trước, trắng ngả vàng |
| Jack, W2, **LM66200** | 5.00 V | 100 % | Đúng định mức | Màu ổn định |
| USB (VBUS 4.75 V), **Schottky OR** | **4.18 V** | 83.5 % | Mô-men −16.5 %; **giật/reset** vì host chỉ 500 mA | Ngả vàng, hụt xanh dương rõ |
| USB (VBUS 4.75 V), **LM66200** | 4.51 V | 90 % | −10 %; vẫn bị giới hạn 500 mA của host | Chấp nhận được |

- **Xung khởi động servo** (đỉnh tới 650 mA/con `[Giả định: KB]`): với Schottky, $V_F$ ≈ 0.55 V → `+5V` ≈ 4.54 V → AMS1117-3.3 (cần ≥ 3.3 + 1.3 = 4.6 V ở dropout tối đa `[Giả định: DS]`) **mất ổn áp → 3V3 tụt → ESP32 brownout reset**.
- **Mức logic NeoPixel:** $V_{IH} = 0.7 \times V_{DD}$ `[Giả định: DS]`. Với `+5V` = 5.0 V → 3.5 V > 3.3 V của ESP32 ⇒ **bắt buộc có level-shifter** (74AHCT125, $V_{IH}$ ≈ 2 V ở VCC 5 V `[Giả định: DS]`), bất kể chọn power-path nào. Thêm điện trở nối tiếp 330–470 Ω trên Data và tụ 470–1000 µF gần nguồn dải LED `[Giả định: ƯL — thông lệ NeoPixel]`.

### 2.3.4. Khuyến nghị chốt

**LM66200 làm ideal-diode OR-ing**, Buck đặt 5.09 V. Lý do cốt lõi: giữ `+5V` ≥ 5.0 V ở dòng worst (Schottky chỉ còn 4.57 V) và chặn ngược thật sự.

Rủi ro còn lại (phải kiểm tra datasheet):
1. Biên dòng chỉ 8 % so với W2; chấp nhận được vì W2 là tổng của mọi tải *đồng thời cực đại*.
2. Ưu tiên khi cắm cả hai: nếu datasheet xác nhận "điện áp cao hơn thắng", Buck 5.09 V chỉ hơn VBUS điển hình (5.0 V) 90 mV nhưng VBUS có thể tới 5.25 V → USB có thể thắng khi cắm cả hai. Khi đó chuyển sang TPS2113A (có ưu tiên) *và* hạ W2 xuống ≤ 1.54 A ($=2\,A/1.3$), hoặc dùng cặp P-MOS tự ngắt.
3. **Phương án dự phòng cho bản thử đầu tiên:** 2 Schottky SS34 — đơn giản, an toàn, chấp nhận `+5V` ≈ 4.5 V; kèm LDO 3.3 V loại dropout thấp.

### 2.3.5. Linh kiện bắt buộc cho USB-C

| Linh kiện | Giá trị | Vai trò |
|---|---|---|
| Điện trở CC1, CC2 | **5.1 kΩ** ±1 %, mỗi chân **một** trở xuống GND | Báo "thiết bị sink" cho host; cáp C-to-C **không có điện** nếu thiếu Rd |
| IC ESD | **USBLC6-2SC6** trên D+/D− `[Giả định: DS — điện dung ≈ 3.5 pF, hợp Full-Speed 12 Mb/s]` | Bảo vệ ESD cho GPIO19/20 |
| PTC VBUS | ≈ 0.75 A hold `[Giả định: ƯL]` (lớn hơn 500 mA mặc định) | Chống ngắn mạch phía board vào cổng USB |
| Tụ VBUS | 10 µF + 100 nF | Lọc/hot-plug |
| Shell (tùy chọn) | 1 MΩ ‖ 4.7 nF tới GND `[Giả định: ƯL]` | Chống nhiễu vỏ |

---

## 2.4. Khối Ổn áp 3.3V (3.3 V LDO)

### 2.4.1. Kiểm tra Dropout

Điện áp thấp nhất của `+5V` (đã trừ power-path LM66200, mục 2.3):

| Chế độ | `+5V` thấp nhất | Headroom $= V_{5V} - 3.3$ |
|---|---|---|
| Jack, W2 | 5.00 V | 1.70 V |
| USB, VBUS 4.75 V, 0.5 A | 4.51 V | 1.21 V |

Dropout `[Giả định: DS]`: AMS1117-3.3 ≈ 1.1 V typ / 1.3 V max ở 0.8–1 A; AP2114H-3.3 cùng lớp ≈ 1.1–1.3 V.

- Chế độ jack: dư 0.4–0.6 V ✅.
- Chế độ USB: ở 0.5 A dropout nhỏ hơn mức 1 A, headroom 1.21 V ✅ nhưng sát. Nếu dùng Schottky OR-ing (`+5V` = 4.18 V, headroom 0.88 V) → **không đạt**.

### 2.4.2. Nhiệt độ mối nối $T_J$

$$P_{loss} = (V_{IN}-3.3)\,I_{out} + V_{IN}\,I_{GND}, \qquad T_J = T_A + P_{loss}\,\theta_{JA}$$

Điều kiện: $V_{IN}$ = **5.1 V** (Buck 5.09 V, không sụt — trường hợp xấu nhất về nhiệt), $I_{GND}$ = 5 mA `[Giả định: DS]`, $T_A$ = **40 °C** `[Giả định: KB — bên trong vỏ hộp kín; nhóm thay bằng nhiệt độ thực]`, SOT-223 $\theta_{JA}$: **90 °C/W** (đồng tối thiểu) và **60 °C/W** (mảng đồng ≥ ≈ 600 mm² dưới tab) `[Giả định: ƯL]`.

| Tình huống | $I_{out}$ | $P_{loss}$ | $T_J$ @ 90 °C/W | $T_J$ @ 60 °C/W |
|---|---|---|---|---|
| Typical | 192 mA | 0.37 W | 73 °C ✅ | 62 °C ✅ |
| Worst kéo dài | 473 mA | 0.88 W | **119 °C** ⚠️ | 93 °C ✅ |
| Mốc 800 mA của đề | 800 mA | 1.47 W | 172 °C ❌ | 128 °C ❌ |
| Worst đỉnh | 853 mA | 1.56 W | **181 °C** ❌ | **134 °C** ❌ |

Giới hạn $T_{J,max}$ = 125 °C, ngắt nhiệt ≈ 150–165 °C `[Giả định: DS]`. Dòng liên tục tối đa:

$$I_{max} = \frac{T_{J,max}-T_A}{\theta_{JA}\,(V_{IN}-3.3)} = \frac{85}{60 \times 1.8} \approx 0.79\ \text{A (60 °C/W)}, \quad \frac{85}{90 \times 1.8} \approx 0.52\ \text{A (90 °C/W)}$$

**Kết luận nhiệt:** LDO chịu được tải kéo dài 473 mA **chỉ khi** có mảng đồng tản nhiệt tốt. Tải đỉnh 853 mA chỉ chấp nhận được nếu kéo dài tính bằng mili-giây (burst Wi-Fi), không phải liên tục.

**Giải pháp:**
1. Mảng đồng dưới tab SOT-223 ≥ ≈ 600 mm², nhiều via nhiệt xuống mặt đồng bên kia.
2. Giảm $V_{IN}$ của LDO (mỗi 0.1 V ít hơn giảm ≈ 5.5 % công suất).
3. **Nếu nhóm xác nhận tải 3V3 kéo dài > ≈ 0.6 A → đổi sang Buck 3.3 V** (ví dụ AP63203 fixed 3.3 V, 2 A, cùng họ AP63205 `[Giả định: DS]`): tổn hao ≈ $2.8\,W \times (1-0.9)/0.9$ ≈ **0.31 W** so với 1.56 W của LDO. Đánh đổi: thêm cuộn cảm, nhiễu đóng cắt gần RF (cần LC lọc và đặt xa ăng-ten).

### 2.4.3. Ổn định vòng hồi tiếp và ESR tụ đầu ra

LDO đời cũ (AMS1117, LM1117) bù pha bằng "zero" tạo bởi ESR của tụ đầu ra: $f_z = \dfrac{1}{2\pi\,ESR\,C}$.

| Tụ đầu ra 22 µF | ESR | $f_z$ | Kết quả |
|---|---|---|---|
| Tantalum | 0.3 Ω | ≈ **24 kHz** | Nằm trong cửa sổ ESR của AMS1117 (≈ 0.3–22 Ω `[Giả định: DS]`) → ổn định |
| MLCC | 0.01 Ω | ≈ **723 kHz** | Zero quá cao, mất bù pha → **dễ tự dao động** |

| | **AMS1117-3.3** | **AP2114H-3.3** | **SPX3819** | **RT9080** |
|---|---|---|---|---|
| Dòng tối đa `[Giả định: DS]` | 1 A | 1 A | **0.5 A** | **0.6 A** |
| Dropout `[Giả định: DS]` | ≈ 1.1–1.3 V | ≈ 1.1–1.3 V | ≈ 0.34 V @ 0.5 A | ≈ 0.2–0.3 V |
| Tụ đầu ra | Cần ESR trong cửa sổ | Ổn định với MLCC | Ổn định với MLCC | Ổn định với MLCC |
| $I_{GND}$ | ≈ 5 mA | thấp | thấp | thấp |
| Đạt ≥ 800 mA (đề) / 853 mA (worst) | ✅ | ✅ | ❌ | ❌ |
| Phù hợp board này? | Chỉ nếu dùng tụ tantalum; MLCC thì sai | ✅ **chọn** | ❌ thiếu dòng | ❌ thiếu dòng |

> ⚠️ SPX3819 và RT9080 (có trong danh sách gợi ý của đề) **không đạt mốc 800 mA**, chỉ dùng nếu nhóm chia tải 3V3 sang nhiều LDO.

**Chốt:** LDO ≥ 1 A, ổn định với MLCC (AP2114H-3.3 hoặc tương đương như AP7361C-33E `[Chưa có đủ thông tin - Cần nhóm tự xác nhận: đọc datasheet — dòng, dropout ở 1 A, dải ESR/tụ cho phép]`). Tụ đầu ra gốm **22 µF** (lưu ý X5R 6.3 V mất ≈ 30–40 % dung lượng ở 3.3 V `[Giả định: ƯL]`).

Kiểm tra sụt áp tải bước nhảy 400 mA, đáp ứng LDO ≈ 10 µs `[Giả định: ƯL]`:

$$\Delta V \approx \frac{\Delta I \cdot \Delta t}{C} = \frac{0.4 \times 10\,\mu s}{22\,\mu F} \approx 0.18\ \text{V} \Rightarrow 3.12\ \text{V}$$

→ trên ngưỡng nguồn tối thiểu 3.0 V của ESP32-S3 `[Giả định: DS]`.

---

# Phần 3: Bảng Tổng kết Chốt Phương án & Checklist Trước khi Vẽ Schematic

## 3.1. Bảng chốt

| Khối | Phương án đề xuất | Linh kiện tiêu biểu | Lý do cốt lõi |
|---|---|---|---|
| Bảo vệ đầu vào | PTC → TVS → P-MOS ideal diode (Zener clamp Gate) | PTC 3 A hold; **SMBJ16A**; P-MOS −30 V (AO3401A); Zener 10 V; R_G 100 kΩ | Tổn hao 0.29 W thay vì 1.10 W; $V_{GS}$ không vượt 12 V; $V_C$ = 26 V < 28 V của Buck |
| Hạ áp 5 V | Buck adjustable ≥ 3 A, $V_{IN\_MIN}$ ≤ 4.5 V | **MP1584**, L = 4.7 µH ($I_{sat}$ ≥ 4.5 A), FB 53.6 kΩ / 10 kΩ → 5.09 V | Duy nhất đạt cả 3 A và $V_{IN\_MIN}$ thấp; LDO 5 V tỏa 16.2 W |
| Chọn nguồn | Ideal-diode OR-ing | **LM66200** (dự phòng: 2× SS34) | `+5V` 5.00 V ở 2.3 A (Schottky: 4.57 V); chặn dòng ngược vào USB |
| USB-C | Rd + ESD + PTC | 2× 5.1 kΩ; **USBLC6-2SC6**; PTC ≈ 0.75 A | C-to-C cần Rd; bảo vệ D+/D− |
| Ổn áp 3.3 V | LDO ≥ 1 A, ổn định MLCC + đồng tản nhiệt (hoặc Buck 3.3 V nếu tải kéo dài > 0.6 A) | **AP2114H-3.3** (dự phòng: AP63203) | 853 mA worst > 800 mA; $T_J$ 93 °C (kéo dài, 60 °C/W) |
| NeoPixel | Level-shifter | 74AHCT125 | $V_{IH}$ = 3.5 V > 3.3 V của ESP32 |

## 3.2. Danh sách `[Chưa có đủ thông tin]` nhóm cần điền

| # | Hạng mục | Cách xác nhận |
|---|---|---|
| 1 | Loại adapter và điện áp hở mạch | Đo bằng đồng hồ vạn năng khi không tải; quyết định TVS (SMBJ15A/16A/18A) |
| 2 | Loại servo thật và số servo chạy cùng lúc | Datasheet servo; đo dòng bằng power meter |
| 3 | Số NeoPixel và mức giới hạn độ sáng | Quyết định W1 hay W2 (3.43 A hay 3.01 A) |
| 4 | Dòng thật từng module Grove | Datasheet module, hoặc đo khi chạy |
| 5 | Dòng cho phép trên chân `+5V` của header | Nhóm tự đặt giới hạn, ghi vào README |
| 6 | Nhiệt độ môi trường $T_A$ và diện tích đồng LDO | Quyết định hộp kín/hở; kết quả sau layout |
| 7 | GPIO cho `VBUS_SENSE`, `PWR_SRC`/PG | Phối hợp với nhóm MCU khi chốt pin-map |
| 8 | Các số `[Giả định: DS]` của MP1584, LM66200, AP2114H, SMBJ, AO3401A | Mở datasheet, đối chiếu từng giá trị, đặc biệt: duty tối đa/dropout MP1584; kiểu chọn kênh LM66200; dropout AP2114H ở 1 A; $V_{GS(max)}$ P-MOS |

## 3.3. Checklist quan trọng trước khi vẽ Schematic và layout

1. **Duty/dropout ở 6 V:** xác nhận Buck vẫn ra ≥ 5.0 V khi $V_{IN}$ = 5.78 V, tải W2 (mục 2.2.2). Nếu không đạt, ghi rõ trong tài liệu: "jack ≥ 7.5 V để chạy tải nặng".
2. **TVS ↔ Buck ↔ P-MOS:** kiểm $V_{RWM}$ ≥ điện áp hở mạch adapter; $V_C$ < $V_{IN\_MAX}$ của Buck *và* < $V_{DS}$ của P-MOS; tụ vào Buck ≥ 35 V.
3. **P-MOS ideal diode:** Drain = `VIN_FUSED`, Source = `VIN_PROT`; có Zener 10 V giữa Source–Gate; thử cắm ngược cực trên mô phỏng/bảng mạch thử.
4. **Layout Buck:** vòng nóng (Cin → IC → diode/cuộn) nhỏ nhất; Cin sát chân VIN–GND; đường FB đi xa nút `SW_BUCK` và lấy ở tụ ra; mặt phẳng GND liền dưới Buck; cuộn cảm $I_{sat}$ ≥ 4.5 A.
5. **LDO 3.3 V:** mảng đồng ≥ ≈ 600 mm² dưới tab, có via nhiệt; tụ đầu ra gốm 22 µF đặt sát chân OUT; đối chiếu dải ESR/tụ trong datasheet LDO đã chọn.
6. **Tụ lọc ESP32-S3:** 10–22 µF + 100 nF đặt sát chân 3V3 của module; EN có RC theo hướng dẫn thiết kế phần cứng của Espressif `[Chưa có đủ thông tin - Cần nhóm tự xác nhận: giá trị RC trong "ESP32-S3 Hardware Design Guidelines"]`.
7. **USB-C:** 5.1 kΩ **riêng** cho CC1 và CC2; USBLC6-2SC6 đặt sát đầu nối; D+/D− đi cặp vi sai ngắn, không qua via thừa; PTC trên VBUS; điểm test `+5V`, `+3V3`, `GND`, `EN`.

---

*Tài liệu này là khung tính toán dựa trên giả định có gắn nhãn. Chỉ chốt linh kiện sau khi đã đối chiếu mọi mục `[Giả định: DS]` với datasheet gốc và điền các mục `[Chưa có đủ thông tin]` bằng số đo thực tế của nhóm.*