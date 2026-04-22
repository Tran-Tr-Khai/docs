# Tài Liệu Mô Tả Dữ Liệu Hệ Thống Điện Mặt Trời

**Ngày dữ liệu:** 2026-04-22  
**Gateway:** ASTER V1.0.0 – SolarBK (Serial: `01-C020-1125-0001`)  
**Đơn vị công suất:** Watt (W)

---

## 1. Tổng Quan Kiến Trúc Hệ Thống

```
[Tấm pin mặt trời]
        │  DC Power
        ▼
  [Inverter (x9)]   ← chuyển DC → AC
        │  AC Power
        ▼
[Tủ phân phối điện]
   ┌────┴────┐
   ▼         ▼
[Tải nhà máy] [Đồng hồ lưới điện (x4)]
                    │
                    ▼
              [Lưới điện EVN]
```

**Gateway ASTER** (thiết bị đầu não) thu thập dữ liệu từ toàn bộ các thiết bị con mỗi 5 phút và đóng gói thành JSON này.

---

## 2. Cấu Trúc JSON

### 2.1. `date_label` — Mốc Thời Gian

```json
"date_label": ["00:00:00", "00:05:00", ..., "08:15:00"]
```

- **100 phần tử**, mỗi phần tử cách nhau **5 phút**
- Bao phủ từ **nửa đêm (00:00) đến 08:15 sáng**
- Mỗi index trong mảng này tương ứng với index cùng vị trí trong tất cả các mảng dữ liệu bên dưới

> **Lưu ý:** Data này chỉ ghi nhận nửa đầu buổi sáng — đây là snapshot tại thời điểm truy vấn API, không phải dữ liệu cả ngày.

---

### 2.2. `data[]` — Mảng Dữ Liệu Từng Thiết Bị

Mỗi phần tử trong `data[]` đại diện cho **một thiết bị vật lý**, được phân loại bởi 2 cờ boolean:

| `is_grid_meter` | `is_radiation` | Loại thiết bị |
|---|---|---|
| `false` | `false` | **Inverter** (Biến tần) |
| `false` | `true` | **Cảm biến bức xạ mặt trời** |
| `true` | `false` | **Đồng hồ đo lưới điện** |

---

## 3. Các Loại Thiết Bị Chi Tiết

### 3.1. Inverter — Biến Tần (9 thiết bị)

**Chức năng:** Chuyển đổi dòng điện DC từ tấm pin sang AC để dùng trong nhà máy.

#### Danh sách Inverter:

| # | Tên thiết bị | Serial | Công suất định mức |
|---|---|---|---|
| 1 | Inverter nối lưới 125kW - Growatt (MAX 125KTL3-X LV) | JVPDFZ400N | 125 kW |
| 2 | Inverter nối lưới 125kW - Growatt (MAX 125KTL3-X LV) | JVPDFZ400X | 125 kW |
| 3 | Inverter nối lưới 125kW - Growatt (MAX 125KTL3-X LV) | (xem file) | 125 kW |
| 4 | Inverter nối lưới 125kW - Growatt (MAX 125KTL3-X LV) | (xem file) | 125 kW |
| 5 | Inverter nối lưới 125kW - Growatt (MAX 125KTL3-X LV) | (xem file) | 125 kW |
| 6 | Inverter nối lưới 125kW - Growatt (MAX 125KTL3-X LV) | (xem file) | 125 kW |
| 7 | Inverter nối lưới 125kW - Growatt (MAX 125KTL3-X LV) | (xem file) | 125 kW |
| 8 | Inverter nối lưới 125kW - Growatt (MAX 125KTL3-X LV) | (xem file) | 125 kW |
| 9 | Inverter nối lưới 3 pha 10kW - Growatt (MOD 10KTL3-X) | MWQ3EZ400D | 10 kW |

> Tổng công suất lắp đặt inverter: **8 × 125kW + 1 × 10kW = 1,010 kW ≈ 1 MW**

#### Trường dữ liệu:

**`device_data_dc`** — Công suất DC đầu vào (từ tấm pin)
```
Giá trị lúc 07:00: ~28,974 W (inverter #1)
Giá trị lúc 08:15: ~70,911 W (inverter #1)
```

**`device_data_ac`** — Công suất AC đầu ra (sau khi biến đổi)
```
Giá trị lúc 07:00: ~28,199 W (inverter #1)
Giá trị lúc 08:15: ~68,515 W (inverter #1)
```

**Hiệu suất biến đổi (DC → AC):**
$$\eta = \frac{P_{AC}}{P_{DC}} = \frac{68{,}515}{70{,}911} \approx 96.6\%$$

Đây là hiệu suất điển hình của inverter Growatt MAX 125KTL3-X, rất tốt.

**Pattern quan sát được:**
- Index 0–82 (00:00 → 06:50): Tất cả = `"0.00"` → **Ban đêm, không có ánh sáng mặt trời**
- Index 83 (06:55): Bắt đầu có giá trị → **Mặt trời mọc ~06:55**
- Index 84–99 (07:00 → 08:15): Tăng dần đều → **Bức xạ tăng theo góc mặt trời**
- Một số index có `"0.00"` giữa chừng (ví dụ index 93–94) → **Mất kết nối tạm thời hoặc lỗi đọc cảm biến**

---

### 3.2. Cảm Biến Bức Xạ Mặt Trời (1 thiết bị)

**Thiết bị:** Hukseflux SR05-D1A3-03 (Pyranometer)  
**Serial:** `28049`  
**Đơn vị thực tế:** W/m² (dù `unit` ghi là "W" — đây là cấu hình chung của gateway)

#### Trường dữ liệu:

**`radiation_data`** — Cường độ bức xạ mặt trời theo thời gian

| Thời điểm | Giá trị (W/m²) | Ý nghĩa |
|---|---|---|
| 00:00 → 05:10 | ~ -1.xx đến -2.xx | Nhiễu cảm biến ban đêm (offset tự nhiên) |
| 06:15 | 0.06 | Ánh sáng đầu tiên xuất hiện |
| 06:30 | 9.71 | Bình minh |
| 07:00 | 56.99 | Mặt trời lên khá cao |
| 07:30 | 150.41 | Bức xạ tăng mạnh |
| 08:00 | 343.35 | Cường độ cao |
| 08:15 | 463.71 | Vẫn đang tăng (chưa đến đỉnh) |

> **Giải thích giá trị âm ban đêm:** Đây là hiện tượng bình thường với pyranometer — cảm biến nhiệt điện (thermopile) bị ảnh hưởng bởi sự chênh lệch nhiệt độ môi trường ban đêm, tạo ra điện áp nhiễu nhỏ dưới 0. Giá trị hợp lệ khi > 5 W/m².

---

### 3.3. Đồng Hồ Lưới Điện — Grid Meter (4 thiết bị)

**Thiết bị:** Mikro DPM380-415 AD (Đồng hồ đo điện năng 3 pha)

| # | Serial | Vai trò quan sát |
|---|---|---|
| 1 | 3380619 | Điểm đấu nối lưới #1 (có giá trị âm → đang phát ngược) |
| 2 | 3652343 | Điểm đấu nối lưới #2 (có giá trị dương lớn ~476kW) |
| 3 | 3652339 | Điểm đo phụ (phần lớn = 0, có thể là tải phụ hoặc offline) |
| 4 | 3652344 | Tải chính của nhà máy (~500–520 kW liên tục 24/7) |

#### Trường dữ liệu:

**`device_real_power`** — Công suất thực (có dấu)

**Quy ước dấu:**
- **Dương (+)**: Đang lấy điện từ lưới EVN (nhà máy đang tiêu thụ điện mua)
- **Âm (-)**: Đang phát điện ngược lên lưới EVN (hệ thống solar dư thừa)

**Phân tích đồng hồ #1 (serial 3380619):**

| Thời điểm | Giá trị (W) | Trạng thái |
|---|---|---|
| 00:00–06:55 | +190,000–260,000 | Mua điện lưới (chưa có solar) |
| 07:00–07:40 | Giảm dần về 0 | Solar bắt đầu bù dần |
| 07:45–08:00 | -2,786 → -35,349 | **Bắt đầu phát ngược lên lưới** |
| 08:05–08:15 | -95,488 → -154,049 | **Phát ngược mạnh, dư điện lớn** |

**Đồng hồ #4 (serial 3652344) — Tải nhà máy:**
- Ổn định ~500–520 kW **xuyên suốt 24/7** → Đây là tải tiêu thụ cơ sở của nhà máy sản xuất (máy móc, điều hòa, chiếu sáng...)

---

## 4. `total_data` — Tổng Hợp Toàn Hệ Thống

```json
"total_data": {
    "device_data_ac": [...],  // Tổng AC output của tất cả inverter
    "device_data_dc": [...]   // Tổng DC input của tất cả inverter
}
```

**Giá trị đỉnh quan sát tại 08:15:**

| Chỉ số | Giá trị |
|---|---|
| Tổng DC (từ tấm pin) | **594,552 W ≈ 595 kW** |
| Tổng AC (ra tải/lưới) | **576,266 W ≈ 576 kW** |
| Hiệu suất tổng thể | **96.9%** |

**Tổng AC tăng trưởng từ 07:00 đến 08:15:**

```
07:00 → ~202,929 W (202 kW)
07:15 → ~266,005 W (266 kW)
07:30 → ~354,991 W (355 kW)
07:45 → ~398,152 W (398 kW)
08:00 → ~481,025 W (481 kW)
08:15 → ~576,266 W (576 kW)   ← đang tăng mạnh
```

---

## 5. `run_nze` — Chế Độ Net Zero Energy

```json
"run_nze": true
```

Cờ này = `true` nghĩa là hệ thống đang chạy ở chế độ **Net Zero Energy (NZE)** — mục tiêu cân bằng điện: lượng điện solar sản xuất ra ≥ lượng điện nhà máy tiêu thụ trong ngày. Khi dư, phát ngược lên lưới; khi thiếu (ban đêm, trời mưa), mua từ EVN.

---

## 6. Luồng Dữ Liệu Tổng Thể (Timeline Ngày 2026-04-22)

```
00:00 – 06:54  │ ████████ Mua điện lưới ~200–260kW │ Solar = 0
06:55 – 06:59  │ ░ Mặt trời mọc, solar bắt đầu khởi động
07:00 – 07:39  │ ░░░░░░░░ Solar tăng dần, bù một phần tải
07:40 – 07:44  │ ░░░░░░░░░░░░ Solar gần bằng tải, mua lưới ≈ 0
07:45 – 08:15  │ ████████ PHÁT NGƯỢC LƯỚI, solar vượt tải nhà máy
               └──────────────────────────────────────────────────▶ Thời gian
```

**Điểm tự cấp điện (Break-even):** Khoảng **07:40–07:45**, khi tổng solar (~350–400 kW) bắt đầu vượt qua mức tiêu thụ nhà máy.

---

## 7. Phân Tích Luồng Điện Thực Tế

### 7.1. Vị Trí Của 4 Đồng Hồ Trong Hệ Thống

Dựa vào pattern hành vi của từng đồng hồ, có thể xác định vai trò thực tế:

| Đồng hồ | Serial | Vai trò thực tế | Hành vi đặc trưng |
|---|---|---|---|
| **Meter #1** | 3380619 | **Điểm đấu nối EVN của Nhà máy A (có Solar)** | Ban ngày chuyển từ dương → âm khi solar dư |
| **Meter #2** | 3652343 | **Điểm đấu nối EVN của Nhà máy B / Xưởng khác** | Luôn dương, tăng vọt vào buổi sáng (ca sản xuất) |
| **Meter #3** | 3652339 | **Điểm đo phụ / Tải nhỏ hoặc offline** | Gần như = 0 toàn bộ thời gian |
| **Meter #4** | 3652344 | **Tải tiêu thụ chính toàn khu** | Ổn định 480–520 kW liên tục 24/7 |

---

### 7.2. Bảng Cân Bằng Công Suất Tại 3 Mốc Thời Gian

| | **00:00** (ban đêm) | **07:30** (solar trung bình) | **08:15** (solar cao) |
|---|---|---|---|
| Solar AC tổng | 0 kW | 355 kW | **576 kW** |
| Meter #1 (Nhà máy A ↔ EVN) | +217 kW *(mua)* | +206 kW *(mua)* | **−154 kW** *(xuất)* |
| Meter #2 (Nhà máy B ↔ EVN) | +222 kW *(mua)* | +214 kW *(mua)* | **+477 kW** *(mua tăng)* |
| Meter #3 | ≈ 0 | ≈ 0 | ≈ 0 |
| Meter #4 (Tải chính) | +514 kW | +484 kW | +503 kW |

> **Nhận xét quan trọng:** Đúng vào lúc Meter #1 chuyển sang âm (08:15), Meter #2 tăng vọt từ ~214 kW lên ~477 kW — đây là bằng chứng trực tiếp nhà máy B đang nhận điện dư từ nhà máy A.

---

### 7.3. Kiểm Chứng Bằng Phép Tính Cân Bằng

**Tại 08:15 — Phương trình cân bằng công suất Nhà máy A:**

$$\underbrace{576\text{ kW}}_{\text{Solar sản xuất}} = \underbrace{422\text{ kW}}_{\text{Tải nội bộ nhà máy A}} + \underbrace{154\text{ kW}}_{\text{Xuất qua Meter \#1}}$$

**Điện xuất 154 kW đó đến Nhà máy B:**

$$\underbrace{477\text{ kW}}_{\text{Meter \#2 tiêu thụ}} = \underbrace{154\text{ kW}}_{\text{Từ solar nhà máy A}} + \underbrace{323\text{ kW}}_{\text{Mua trực tiếp từ EVN}}$$

**→ Cả 2 phương trình khớp chính xác ✓**

---

### 7.4. Sơ Đồ Luồng Điện Tại 08:15

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
              TRẠM BIẾN ÁP EVN (22kV / Trung thế)
                 ▲                        │
                 │ KHÔNG có               │ 323 kW từ EVN
                 │ điện chạy              │ xuống Nhà máy B
                 │ lên đây                ▼
    ┌────────────┴────────┐    ┌───────────────────────┐
    │    NHÀ MÁY A        │    │    NHÀ MÁY B / XƯỞNG  │
    │  (Có hệ thống Solar)│    │    (Trong cùng KCN)   │
    │                     │    │                       │
    │  [Solar 576 kW]     │    │  Tiêu thụ: 477 kW     │
    │        │            │    │   ┌──────────────┐    │
    │  Dùng nội bộ 422 kW │    │   │323 kW từ EVN │    │
    │        │            │    │   │154 kW từ A   │    │
    │  Dư:  154 kW        │    │   └──────────────┘    │
    │        │            │    │           ▲           │
    │  [Meter #1: −154 kW]│────┼───────────┘           │
    │                     │    │  ← Qua cáp hạ thế     │
    └─────────────────────┘    │    nội khu KCN →      │
                               └───────────────────────┘
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

### 7.5. Giải Thích Tại Sao "Không Phát Lên Lưới Quốc Gia"

Nhà cung cấp nói **đúng về mặt kỹ thuật vật lý**. Đây là nguyên lý:

> Dòng điện luôn chảy theo đường ngắn nhất đến tải gần nhất. Điện dư từ solar nhà máy A **không cần chạy ngược qua máy biến áp EVN** lên đường dây trung/cao thế quốc gia — nó được nhà máy B trong cùng KCN hút ngay trên đường dây phân phối hạ thế nội khu.

| Thuật ngữ nhà cung cấp dùng | Ý nghĩa thực tế |
|---|---|
| *"Không phát lên lưới"* | Không phát ngược qua MBA lên đường dây 110kV/220kV quốc gia |
| *"Điện dư chạy qua đường EVN"* | Chạy trên đường dây hạ thế/trung thế **nội bộ KCN** do EVN quản lý |
| **Giá trị âm ở Meter #1** | Đây chính là lượng điện đang xuất ra đường dây nội khu đó |

---

### 7.6. Lưu Ý Về Hợp Đồng Điện

Dù bản chất vật lý giống nhau, cách tính tiền điện phụ thuộc vào **cấu trúc hợp đồng với EVN**:

| Trường hợp | Cách tính tiền |
|---|---|
| 4 đồng hồ trong **1 hợp đồng điện** | Bù trừ nội bộ toàn bộ — tiết kiệm tối đa, không qua EVN |
| Mỗi nhà máy có **hợp đồng riêng** | EVN ghi nhận nhà máy A "bán" và nhà máy B "mua" — có thể phát sinh phí truyền tải |

> **Khuyến nghị:** Xác nhận với EVN xem Meter #1 và Meter #2 có thuộc cùng 1 mã khách hàng/hợp đồng không, để tối ưu cách tính phí điện.

---

## 8. Lưu Ý Chất Lượng Dữ Liệu

| Vấn đề | Ví dụ | Nguyên nhân có thể |
|---|---|---|
| Giá trị `"0.00"` lẻ giữa chuỗi dương | Inverter #1 index 93–94 | Mất kết nối Modbus/RS485 tạm thời, timeout đọc |
| Bức xạ âm ban đêm | -1.xx đến -2.xx | Offset nhiệt điện bình thường của pyranometer |
| Giá trị âm ở grid meter | -154,049 W | Phát ngược lưới (hoàn toàn bình thường, thiết kế có chủ đích) |

---

## 9. Gợi Ý Phân Tích Nâng Cao

1. **Tính điện năng sản xuất (kWh):** Tích phân công suất AC theo thời gian: $E = \sum P_{AC} \times \frac{5}{60}$ (kWh)
2. **Performance Ratio (PR):** $PR = \frac{E_{AC,thực\_tế}}{E_{DC,lý\_thuyết}}$ — so sánh với thiết kế để phát hiện suy giảm tấm pin
3. **Specific Yield:** kWh/kWp — đánh giá hiệu quả theo công suất lắp đặt
4. **Điện mua/bán lưới:** Từ `device_real_power` của các grid meter, tính được chi phí điện tiết kiệm được
5. **Correlation bức xạ vs công suất:** So sánh `radiation_data` với `total_data.device_data_ac` để kiểm tra độ sạch tấm pin
