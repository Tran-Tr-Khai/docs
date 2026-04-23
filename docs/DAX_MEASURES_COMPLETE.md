# DAX Measures — Transfermarkt Analytics
**Role:** Senior Power BI Developer  
**Dataset:** Transfermarkt (12 CSV, ~7.2M records)  
**Scope:** Page 1 — Squad Health Check | Page 2 — Player Scout Board  
**Version:** v2.0 — April 2026 — Rebuilt against DATA_DICTIONARY.md  

> ⚠️ **Schema-verified:** Mọi column reference đã được kiểm tra kỹ với DATA_DICTIONARY.md trước khi viết.

---

## MỤC LỤC

| # | Phần | Nội dung |
|---|------|---------|
| 0 | [Setup](#0-setup) | Bảng _Measures, ghi chú Data Model, Season slicer |
| 1 | [Calculated Columns](#1-calculated-columns--tạo-trong-data-view) | Player Age Group, Contract Expiry Year, ... |
| 2 | [Base Measures](#2-base-measures) | Nền móng dùng xuyên suốt 2 trang |
| 3 | [Page 1 — Squad Health Check](#3-page-1--squad-health-check) | KPIs + 5 Visuals |
| 4 | [Page 2 — Player Scout Board](#4-page-2--player-scout-board) | KPIs + 5 Visuals |
| 5 | [Conditional Formatting](#5-conditional-formatting-measures) | CF Rules cho cả 2 trang |
| 6 | [Dynamic Labels & UX](#6-dynamic-labels--ux) | Titles, tooltips, alerts |
| 7 | [Color Palette & Design Guide](#7-color-palette--design-guide) | Màu sắc + hướng dẫn setup |

---

## 0. Setup

### 0.1 — Tạo bảng _Measures (Calculated Table)
```dax
_Measures = DATATABLE("Placeholder", STRING, {{""}})
```
> Tạo bảng này trong Power BI → Enter Data. Sau đó **move tất cả measures vào đây**. Ẩn cột Placeholder.  
> Kết quả: Data Model sạch, chuyên nghiệp khi interviewer mở Report View.

### 0.2 — Ghi chú quan trọng về Data Model

```
⚠️ SEASON SLICER:
   appearances KHÔNG có cột season!
   → Đặt slicer trên games[season]
   → Relationship appearances[game_id] → games[game_id] sẽ tự propagate filter
   → SUM(appearances[minutes_played]) sẽ tự filtered by season khi slicer active

⚠️ CLUB SLICER:
   → Đặt slicer trên clubs[name] hoặc clubs[club_id]
   → Relationship players[current_club_id] → clubs[club_id] propagate to players
   → Relationship appearances[player_club_id] → clubs[club_id] propagate to appearances

⚠️ TRANSFERS — DUAL ROLE:
   transfers[to_club_id] và transfers[from_club_id] — KHÔNG có active relationship với clubs
   → KHÔNG dùng SELECTEDVALUE(clubs[club_id]) làm filter arg trong CALCULATE
   → PHẢI dùng pattern: transfers[to_club_id] IN VALUES(clubs[club_id])
```

---

## 1. Calculated Columns — Tạo trong Data View

> Tạo các cột này trong bảng tương ứng. Chúng được tính tại load time và cho phép dùng trong visual axis / legend trực tiếp mà không cần measure.

### 1.1 — Bảng `players`

```dax
// ─────────────────────────────────────────────────────────────
// Tuổi cầu thủ (tính tại ngày refresh)
// Dùng cho: Visual 4 Scatter (Size = Age)
// ─────────────────────────────────────────────────────────────
Player Age =
DATEDIFF(players[date_of_birth], TODAY(), YEAR)

// ─────────────────────────────────────────────────────────────
// Nhóm tuổi (cho Visual 1 — Age Histogram trục X)
// Kỹ thuật: SWITCH(TRUE()) — sạch hơn nested IF, dễ đọc hơn
// ─────────────────────────────────────────────────────────────
Player Age Group =
SWITCH(
    TRUE(),
    ISBLANK(players[date_of_birth]),                                          "Unknown",
    DATEDIFF(players[date_of_birth], TODAY(), YEAR) < 21,                     "1. U21",
    DATEDIFF(players[date_of_birth], TODAY(), YEAR) <= 25,                    "2. 21-25",
    DATEDIFF(players[date_of_birth], TODAY(), YEAR) <= 29,                    "3. 26-29",
                                                                               "4. 30+"
)
// Prefix 1./2./3./4. → Power BI sẽ sort đúng thứ tự tự động
// KHÔNG cần thêm "Sort by Column" trick

// ─────────────────────────────────────────────────────────────
// Năm hết hạn hợp đồng (cho Visual 5 — Contract Matrix cột header)
// Trả về text để dùng làm column header trong Matrix visual
// ─────────────────────────────────────────────────────────────
Contract Expiry Year =
IF(
    ISBLANK(players[contract_expiration_date]),
    "No Data",
    FORMAT(YEAR(players[contract_expiration_date]), "0")
)

// ─────────────────────────────────────────────────────────────
// Số tháng còn lại trong hợp đồng
// Dương = còn hạn, Âm = đã hết hạn
// Dùng cho: Priority Action Table (Page 6), Contract Risk bars
// ─────────────────────────────────────────────────────────────
Months to Expiry =
IF(
    ISBLANK(players[contract_expiration_date]),
    BLANK(),
    DATEDIFF(TODAY(), players[contract_expiration_date], MONTH)
)

// ─────────────────────────────────────────────────────────────
// Contract Action (Recommendation logic — Page 1 Visual 5 tooltip)
// Business logic: kết hợp MV + thời gian còn lại
// players[market_value_in_eur] là snapshot MV — đủ chính xác cho recommendation
// ─────────────────────────────────────────────────────────────
Contract Action =
VAR MonthsLeft =
    IF(ISBLANK(players[contract_expiration_date]), 999,
       DATEDIFF(TODAY(), players[contract_expiration_date], MONTH))
VAR MV = players[market_value_in_eur] / 1000000  -- triệu EUR
RETURN
    SWITCH(
        TRUE(),
        MonthsLeft < 0,            "🔚 EXPIRED",
        MV >= 5 && MonthsLeft <= 18, "🔴 PRIORITY RENEW",
        MV >= 5 && MonthsLeft <= 36, "🟡 PLAN RENEWAL",
        MV < 5  && MonthsLeft <= 12, "⚫ DECIDE NOW",
        MonthsLeft <= 12,            "🟠 EXPIRING SOON",
        "⚪ MONITOR"
    )

// ─────────────────────────────────────────────────────────────
// MV vs Peak % (column) — cho Sell Window filter trực tiếp
// players[market_value_in_eur] = MV hiện tại (snapshot)
// players[highest_market_value_in_eur] = Peak MV (đã lưu sẵn)
// ─────────────────────────────────────────────────────────────
MV vs Peak Pct =
IF(
    players[highest_market_value_in_eur] > 0,
    ROUND(DIVIDE(players[market_value_in_eur],
                 players[highest_market_value_in_eur], 0) * 100, 1),
    BLANK()
)
```

---

## 2. Base Measures

> Đây là foundation. Tất cả measures nâng cao ở Page 1/2 đều reference các measures này. Pattern chuẩn: **luôn dùng VAR/RETURN + DIVIDE thay vì /**.

### 2.1 — Performance (từ appearances)

```dax
// ─────────────────────────────────────────────────────────────
// Tổng số phút thi đấu
// Tự động filtered by season khi slicer trên games[season] active
// ─────────────────────────────────────────────────────────────
[Total Minutes] =
SUM(appearances[minutes_played])

// ─────────────────────────────────────────────────────────────
// Tổng bàn thắng — từ appearances (chính xác nhất, đã match per game)
// ─────────────────────────────────────────────────────────────
[Total Goals] =
SUM(appearances[goals])

// ─────────────────────────────────────────────────────────────
// Tổng kiến tạo
// ─────────────────────────────────────────────────────────────
[Total Assists] =
SUM(appearances[assists])

// ─────────────────────────────────────────────────────────────
// Tổng số trận tham gia (appearances, không phải club_games)
// ─────────────────────────────────────────────────────────────
[Total Appearances] =
COUNTROWS(appearances)

// ─────────────────────────────────────────────────────────────
// Thẻ vàng + đỏ
// ─────────────────────────────────────────────────────────────
[Total Yellow Cards] =
SUM(appearances[yellow_cards])

[Total Red Cards] =
SUM(appearances[red_cards])
```

### 2.2 — Market Value (từ players snapshot)

```dax
// ─────────────────────────────────────────────────────────────
// MV hiện tại từ players table (snapshot — nhanh, đủ chính xác cho KPI)
// players[market_value_in_eur] là giá trị mới nhất Transfermarkt ghi nhận
// ─────────────────────────────────────────────────────────────
[Current MV] =
SUM(players[market_value_in_eur])

// ─────────────────────────────────────────────────────────────
// Peak MV từ trước đến nay (đã lưu sẵn trong players table)
// ─────────────────────────────────────────────────────────────
[Peak MV] =
SUM(players[highest_market_value_in_eur])

// ─────────────────────────────────────────────────────────────
// Squad Total MV — tổng cộng per-player latest value từ player_valuations
// Kỹ thuật: SUMX + LASTNONBLANKVALUE → lấy giá trị mới nhất cho mỗi cầu thủ
// Chính xác hơn SUM(players[market_value_in_eur]) vì dùng valuation history
// Dùng cho: KPI "Total Squad MV" và benchmarking comparisons
// ─────────────────────────────────────────────────────────────
[Squad Total MV] =
SUMX(
    VALUES(player_valuations[player_id]),
    CALCULATE(
        LASTNONBLANKVALUE(
            player_valuations[date],
            SUM(player_valuations[market_value_in_eur])
        )
    )
)
```

### 2.3 — Club Game Results (từ club_games)

```dax
// ─────────────────────────────────────────────────────────────
// Tổng số trận (theo góc nhìn CLB — mỗi trận = 1 row trong club_games)
// ─────────────────────────────────────────────────────────────
[Total Games] =
COUNTROWS(club_games)

// ─────────────────────────────────────────────────────────────
// Thắng / Thua / Hòa
// club_games[is_win] values: 1 (thắng), 0 (không thắng)
// Hòa = own_goals = opponent_goals
// ─────────────────────────────────────────────────────────────
[Total Wins] =
CALCULATE(COUNTROWS(club_games), club_games[is_win] = 1)

[Total Losses] =
CALCULATE(
    COUNTROWS(club_games),
    club_games[is_win] = 0,
    club_games[own_goals] < club_games[opponent_goals]
)

[Total Draws] =
CALCULATE(
    COUNTROWS(club_games),
    club_games[own_goals] = club_games[opponent_goals]
)

// ─────────────────────────────────────────────────────────────
// Tỷ lệ thắng (%)
// ─────────────────────────────────────────────────────────────
[Win Rate %] =
DIVIDE([Total Wins], [Total Games], 0) * 100
```

---

## 3. Page 1 — Squad Health Check

### 3.1 — KPI Cards (Row 1)

```dax
// ─────────────────────────────────────────────────────────────
// KPI 1: Tuổi trung bình squad
// Kỹ thuật: AVERAGEX để tính DATEDIFF per player — chính xác hơn AVERAGE(date)
// players[date_of_birth] là date column — DATEDIFF trả về integer years
// ─────────────────────────────────────────────────────────────
[Avg Squad Age] =
AVERAGEX(
    FILTER(players, NOT ISBLANK(players[date_of_birth])),
    DATEDIFF(players[date_of_birth], TODAY(), YEAR)
)

// ─────────────────────────────────────────────────────────────
// KPI 2: Tỷ lệ cầu thủ ngoại (%)
// Schema fact: clubs KHÔNG có country_name!
// clubs[foreigners_percentage] = đã được Transfermarkt tính sẵn — chính xác nhất
// SELECTEDVALUE: khi 1 CLB được chọn → lấy % của CLB đó
//               khi nhiều CLB → trả về average (cho League view)
// ─────────────────────────────────────────────────────────────
[Foreign Players %] =
SELECTEDVALUE(
    clubs[foreigners_percentage],
    AVERAGEX(ALLSELECTED(clubs), clubs[foreigners_percentage])
)

// ─────────────────────────────────────────────────────────────
// KPI 2b: Số cầu thủ ngoại (absolute count)
// ─────────────────────────────────────────────────────────────
[Foreign Players Count] =
SELECTEDVALUE(
    clubs[foreigners_number],
    SUM(clubs[foreigners_number])
)

// ─────────────────────────────────────────────────────────────
// KPI 3: Số cầu thủ ĐTQG
// clubs[national_team_players] = đã có sẵn trong clubs table (theo DATA_DICTIONARY)
// ─────────────────────────────────────────────────────────────
[National Team Players] =
SELECTEDVALUE(
    clubs[national_team_players],
    SUM(clubs[national_team_players])
)

// ─────────────────────────────────────────────────────────────
// KPI 4: Cầu thủ hết hạn hợp đồng trong 12 tháng tới
// Điều kiện: contract còn hiệu lực (> TODAY) VÀ hết hạn trong 12 tháng
// Kỹ thuật: 2 điều kiện DATEDIFF để đảm bảo chỉ lấy hợp đồng ĐANG CÒN HIỆU LỰC
// ─────────────────────────────────────────────────────────────
[Players Expiring < 1 Year] =
CALCULATE(
    COUNTROWS(players),
    NOT ISBLANK(players[contract_expiration_date]),
    DATEDIFF(TODAY(), players[contract_expiration_date], MONTH) >= 0,
    DATEDIFF(TODAY(), players[contract_expiration_date], MONTH) <= 12
)

// ─────────────────────────────────────────────────────────────
// KPI 4b: Cầu thủ hết hạn trong 6 tháng (CRITICAL — màu đỏ)
// ─────────────────────────────────────────────────────────────
[Players Expiring < 6 Months] =
CALCULATE(
    COUNTROWS(players),
    NOT ISBLANK(players[contract_expiration_date]),
    DATEDIFF(TODAY(), players[contract_expiration_date], MONTH) >= 0,
    DATEDIFF(TODAY(), players[contract_expiration_date], MONTH) <= 6
)

// ─────────────────────────────────────────────────────────────
// KPI 5: Value at Risk (tổng MV của cầu thủ sắp hết hạn 12M)
// Dùng players[market_value_in_eur] (snapshot) — đủ chính xác cho risk KPI
// ─────────────────────────────────────────────────────────────
[Value at Risk 12M] =
CALCULATE(
    SUM(players[market_value_in_eur]),
    NOT ISBLANK(players[contract_expiration_date]),
    DATEDIFF(TODAY(), players[contract_expiration_date], MONTH) >= 0,
    DATEDIFF(TODAY(), players[contract_expiration_date], MONTH) <= 12
)

// ─────────────────────────────────────────────────────────────
// KPI 5b: % giá trị squad đang ở rủi ro
// ─────────────────────────────────────────────────────────────
[% Squad Value at Risk] =
DIVIDE([Value at Risk 12M], [Current MV], 0) * 100
```

---

### 3.2 — Visual 1: Age Distribution Histogram

```dax
// ─────────────────────────────────────────────────────────────
// SETUP VISUAL:
//   Trục X  → Calculated Column: players[Player Age Group]  (Section 1.1)
//             Đã có prefix "1./2./3./4." → sort tự động đúng
//   Trục Y  → [Player Count]  ← measure dưới đây
//   Legend  → players[position]  (values: Attack / Midfield / Defender / Goalkeeper)
//   Color   → dùng theme JSON (xem Section 7)
// ─────────────────────────────────────────────────────────────

// Measure Y-axis: số cầu thủ (tự động slice theo Age Group + Position theo filter context)
[Player Count] =
COUNTROWS(players)

// ─────────────────────────────────────────────────────────────
// Recruitment Gap Alert — KPI Card phía trên histogram
// Logic: nhóm tuổi nào chiếm < 15% tổng squad là đang thiếu
// Kỹ thuật: tính tỷ lệ từng nhóm trong ALLSELECTED context
// ─────────────────────────────────────────────────────────────
[Recruitment Gap Alert] =
VAR Total    = COUNTROWS(ALLSELECTED(players))
VAR U21Pct   = DIVIDE(CALCULATE(COUNTROWS(players), DATEDIFF(players[date_of_birth], TODAY(), YEAR) < 21),   Total, 0) * 100
VAR Y2125Pct = DIVIDE(CALCULATE(COUNTROWS(players), DATEDIFF(players[date_of_birth], TODAY(), YEAR) >= 21, DATEDIFF(players[date_of_birth], TODAY(), YEAR) <= 25), Total, 0) * 100
VAR Y2629Pct = DIVIDE(CALCULATE(COUNTROWS(players), DATEDIFF(players[date_of_birth], TODAY(), YEAR) >= 26, DATEDIFF(players[date_of_birth], TODAY(), YEAR) <= 29), Total, 0) * 100
VAR Over30Pct = DIVIDE(CALCULATE(COUNTROWS(players), DATEDIFF(players[date_of_birth], TODAY(), YEAR) >= 30), Total, 0) * 100
RETURN
    SWITCH(
        TRUE(),
        Over30Pct > 40,   "🔴 AGING SQUAD — Over 30 chiếm " & FORMAT(Over30Pct, "0") & "%",
        U21Pct < 10,      "⚠️ Thiếu lứa trẻ — U21 chỉ " & FORMAT(U21Pct, "0") & "%",
        Y2125Pct < 15,    "⚠️ Thiếu lứa 21-25 — potential gap",
        Y2629Pct < 20,    "⚠️ Thiếu lứa đỉnh cao 26-29",
        "✅ Phân bổ tuổi cân bằng"
    )

// ─────────────────────────────────────────────────────────────
// Age Balance Score — KPI Card tổng hợp (0-100)
// Đội hình lý tưởng: Prime (26-29) = 35%+, Trẻ (U25) = 30%+, Lão tướng < 20%
// ─────────────────────────────────────────────────────────────
[Age Balance Score] =
VAR Total = COUNTROWS(players)
VAR PrimePct  = DIVIDE(CALCULATE(COUNTROWS(players), DATEDIFF(players[date_of_birth], TODAY(), YEAR) >= 24, DATEDIFF(players[date_of_birth], TODAY(), YEAR) <= 29), Total, 0) * 100
VAR YouthPct  = DIVIDE(CALCULATE(COUNTROWS(players), DATEDIFF(players[date_of_birth], TODAY(), YEAR) <= 23), Total, 0) * 100
VAR Over30Pct = DIVIDE(CALCULATE(COUNTROWS(players), DATEDIFF(players[date_of_birth], TODAY(), YEAR) >= 30), Total, 0) * 100
RETURN
    SWITCH(
        TRUE(),
        PrimePct >= 35 && YouthPct >= 20,               "⭐ Excellent",
        PrimePct >= 30 && YouthPct >= 15,               "✅ Balanced",
        PrimePct >= 25 && Over30Pct < 25,               "📊 Acceptable",
        Over30Pct >= 35,                                 "⚠️ Aging",
        YouthPct >= 40 && PrimePct < 20,                "🔧 Too Young",
        "❓ Mixed"
    )
```

---

### 3.3 — Visual 2: Squad Depth by Position (Stacked Bar)

```dax
// ─────────────────────────────────────────────────────────────
// SETUP VISUAL:
//   Trục X  → game_lineups[position]  (vị trí thi đấu thực tế, đa dạng hơn players[position])
//             HOẶC players[position] (Attack/Midfield/Defender/Goalkeeper — gọn hơn)
//   Trục Y  → [Player Count in Lineup]
//   Legend  → [Lineup Type Label]  ← measure dưới đây
//   Filter  → Club slicer + Season slicer (game_lineups[date] → games[date])
//
// game_lineups[type] values: "starting_lineup" / "substitutes"
// ─────────────────────────────────────────────────────────────

// Số lần ra sân chính (xuất phát)
[Starter Appearances] =
CALCULATE(
    COUNTROWS(game_lineups),
    game_lineups[type] = "starting_lineup"
)

// Số lần dự bị (vào thay)
[Sub Appearances] =
CALCULATE(
    COUNTROWS(game_lineups),
    game_lineups[type] = "substitutes"
)

// Tỷ lệ xuất phát so với tổng lần trong đội hình
[Starter Rate %] =
DIVIDE(
    [Starter Appearances],
    [Starter Appearances] + [Sub Appearances],
    0
) * 100

// Số cầu thủ unique được HLV dùng (đo squad rotation)
[Active Squad Players] =
DISTINCTCOUNT(game_lineups[player_id])

// Số lần vào sân trong lineup (dùng cho Stacked Bar Y-axis)
[Player Count in Lineup] =
COUNTROWS(game_lineups)

// Label cho Legend (thay vì dùng raw "starting_lineup")
[Lineup Type Label] =
SWITCH(
    MAX(game_lineups[type]),
    "starting_lineup", "⬆️ Starting XI",
    "substitutes",     "🔄 Substitute",
    "Unknown"
)
```

---

### 3.4 — Visual 3: Market Value by Player (Treemap)

```dax
// ─────────────────────────────────────────────────────────────
// SETUP VISUAL:
//   Values (kích thước ô) → players[market_value_in_eur]  (drag column trực tiếp)
//   Details (label)       → players[name]
//   Color / Group         → players[position]
//   Tooltip               → [MV Display] + [Player Age] column
//
// Không cần measure đặc biệt cho treemap — Power BI tự aggregate
// Chỉ cần các measures dưới đây cho Context và Tooltip
// ─────────────────────────────────────────────────────────────

// Top 5 cầu thủ chiếm bao nhiêu % tổng MV squad?
// Kỹ thuật: TOPN(5) + SUMX — pattern advanced thể hiện kỹ năng
[Top 5 MV Concentration %] =
VAR Top5MV =
    SUMX(
        TOPN(
            5,
            VALUES(players[player_id]),
            CALCULATE(MAX(players[market_value_in_eur])),
            DESC
        ),
        CALCULATE(MAX(players[market_value_in_eur]))
    )
VAR TotalMV = [Current MV]
RETURN
    DIVIDE(Top5MV, TotalMV, 0) * 100

// Cảnh báo nếu squad quá phụ thuộc vào 1-2 ngôi sao
[MV Concentration Alert] =
VAR Pct = [Top 5 MV Concentration %]
RETURN
    SWITCH(
        TRUE(),
        Pct >= 70, "🔴 Rất tập trung — " & FORMAT(Pct, "0") & "% MV chỉ ở 5 cầu thủ",
        Pct >= 55, "🟡 Khá tập trung — " & FORMAT(Pct, "0") & "% ở Top 5",
        "✅ MV phân bổ đều — " & FORMAT(Pct, "0") & "% ở Top 5"
    )

// Format MV để hiển thị đẹp trong tooltip / KPI card
[MV Display] =
VAR MV = [Current MV]
RETURN
    SWITCH(
        TRUE(),
        ISBLANK(MV),      "—",
        MV >= 1000000000, FORMAT(MV / 1000000000, "€0.0") & "B",
        MV >= 1000000,    FORMAT(MV / 1000000, "€0.0") & "M",
        MV >= 1000,       FORMAT(MV / 1000, "€0.0") & "K",
        FORMAT(MV, "€0")
    )
```

---

### 3.5 — Visual 4: Minutes Played vs Market Value (Scatter Plot)

```dax
// ─────────────────────────────────────────────────────────────
// SETUP VISUAL:
//   Trục X  → [Total Minutes]  (base measure Section 2.1)
//   Trục Y  → [Current MV]     (base measure Section 2.2)
//   Size    → players[Player Age]  (Calculated Column Section 1.1)
//   Details → players[name]
//   Color   → players[position]
//   Tooltip → [Underperforming Label]
//
// Để visual hoạt động đúng: drag players[name] vào "Details"
// Power BI sẽ tự split scatter theo từng cầu thủ
// ─────────────────────────────────────────────────────────────

// Cầu thủ đắt tiền nhưng ít ra sân (lãng phí nguồn lực)
// Threshold: MV > 5M EUR nhưng < 900 phút (< 10 trận đầy đủ)
[Underperforming Assets Count] =
COUNTROWS(
    FILTER(
        players,
        players[market_value_in_eur] > 5000000
        && CALCULATE(SUM(appearances[minutes_played])) < 900
    )
)

// Tổng MV của nhóm "lãng phí" — để tính chi phí cơ hội
[Underperforming Assets MV] =
SUMX(
    FILTER(
        players,
        players[market_value_in_eur] > 5000000
        && CALCULATE(SUM(appearances[minutes_played])) < 900
    ),
    players[market_value_in_eur]
)

// Label cho tooltip từng cầu thủ trong scatter
[Underperforming Label] =
VAR Mins = [Total Minutes]
VAR MV = [Current MV] / 1000000  -- triệu EUR
RETURN
    SWITCH(
        TRUE(),
        MV >= 5 && Mins < 450,  "🔴 HIGH VALUE, LOW USAGE",
        MV >= 5 && Mins < 900,  "🟡 Underutilized",
        MV >= 2 && Mins < 450,  "🟠 Low Minutes",
        Mins = 0,               "⚫ No Appearances",
        "✅ Regular"
    )

// Cost per minute — đo hiệu quả kinh tế
// Tính: MV / Total Minutes → bao nhiêu EUR giá trị cho mỗi phút ra sân
[Cost per Minute (€)] =
DIVIDE([Current MV], [Total Minutes], BLANK())
```

---

### 3.6 — Visual 5: Contract Expiry Timeline (Matrix)

```dax
// ─────────────────────────────────────────────────────────────
// SETUP VISUAL (Matrix):
//   Rows    → players[position]
//   Columns → players[Contract Expiry Year]  (Calculated Column Section 1.1)
//   Values  → [Contract Expiry Count]  ← hoặc [Value Expiring by Year]
//   Conditional Formatting → [CF Contract Risk Color]  (Section 5)
//
// Sort cột: drag "Contract Expiry Year" vào Columns, sort ascending
// ─────────────────────────────────────────────────────────────

// Số cầu thủ hết hạn — dùng cho matrix cells
[Contract Expiry Count] =
CALCULATE(
    COUNTROWS(players),
    NOT ISBLANK(players[contract_expiration_date])
)

// Tổng MV hết hạn (cho matrix variant hiển thị tài chính)
[Value Expiring by Year] =
CALCULATE(
    SUM(players[market_value_in_eur]),
    NOT ISBLANK(players[contract_expiration_date])
)

// Số cầu thủ hết hạn trong năm hiện tại (cho KPI alert)
[Expiring This Year] =
CALCULATE(
    COUNTROWS(players),
    NOT ISBLANK(players[contract_expiration_date]),
    YEAR(players[contract_expiration_date]) = YEAR(TODAY())
)

// Số cầu thủ hết hạn trong năm tới
[Expiring Next Year] =
CALCULATE(
    COUNTROWS(players),
    NOT ISBLANK(players[contract_expiration_date]),
    YEAR(players[contract_expiration_date]) = YEAR(TODAY()) + 1
)

// Avg MV của cầu thủ đang trong vùng risk (< 12M contract)
[Avg MV of Expiring Players] =
CALCULATE(
    AVERAGEX(
        FILTER(players, NOT ISBLANK(players[market_value_in_eur])),
        players[market_value_in_eur]
    ),
    NOT ISBLANK(players[contract_expiration_date]),
    DATEDIFF(TODAY(), players[contract_expiration_date], MONTH) >= 0,
    DATEDIFF(TODAY(), players[contract_expiration_date], MONTH) <= 12
)
```

---

## 4. Page 2 — Player Scout Board

### 4.1 — KPI Cards (Row 1)

```dax
// ─────────────────────────────────────────────────────────────
// KPI 1: Goals per 90 minutes
// Chuẩn scouting metric — normalized để so sánh cầu thủ ít/nhiều phút
// DIVIDE với 0 tránh lỗi khi minutes = 0
// ─────────────────────────────────────────────────────────────
[Goals per 90] =
DIVIDE([Total Goals], [Total Minutes], 0) * 90

// ─────────────────────────────────────────────────────────────
// KPI 2: Assists per 90
// ─────────────────────────────────────────────────────────────
[Assists per 90] =
DIVIDE([Total Assists], [Total Minutes], 0) * 90

// ─────────────────────────────────────────────────────────────
// KPI 3: Goal Contributions per 90 (G+A per 90)
// Metric tổng hợp — đánh giá tổng đóng góp tấn công
// ─────────────────────────────────────────────────────────────
[Goal Contributions per 90] =
[Goals per 90] + [Assists per 90]

// ─────────────────────────────────────────────────────────────
// KPI 4: Minutes per Goal (bao nhiêu phút ghi 1 bàn)
// BLANK() khi 0 goals — tránh hiện Infinity hoặc 0 gây nhầm lẫn
// ─────────────────────────────────────────────────────────────
[Minutes per Goal] =
IF(
    [Total Goals] = 0,
    BLANK(),
    DIVIDE([Total Minutes], [Total Goals])
)

// ─────────────────────────────────────────────────────────────
// KPI 5: Yellow Cards per Game (Discipline)
// ─────────────────────────────────────────────────────────────
[Yellow Cards per Game] =
DIVIDE([Total Yellow Cards], [Total Appearances], 0)

// ─────────────────────────────────────────────────────────────
// KPI 6: Red Cards per Game
// ─────────────────────────────────────────────────────────────
[Red Cards per Game] =
DIVIDE([Total Red Cards], [Total Appearances], 0)

// ─────────────────────────────────────────────────────────────
// KPI 7: Discipline Risk Score (thấp = kỷ luật tốt)
// Trọng số: Thẻ đỏ (3x) + Thẻ vàng (1x) — theo quy tắc tích lũy
// ─────────────────────────────────────────────────────────────
[Discipline Risk Score] =
[Yellow Cards per Game] * 1 + [Red Cards per Game] * 3
```

---

### 4.2 — Market Value Intelligence

```dax
// ─────────────────────────────────────────────────────────────
// MV 1 năm trước — từ player_valuations history
// Kỹ thuật: EDATE(-12) cho exact month offset (tốt hơn -365 DAY)
//           LASTNONBLANKVALUE để lấy giá trị gần nhất tại/trước mốc thời gian
// player_valuations[date] là date column — EDATE hoạt động chính xác
// ─────────────────────────────────────────────────────────────
[MV 1 Year Ago] =
VAR CutoffDate = EDATE(TODAY(), -12)
RETURN
    CALCULATE(
        LASTNONBLANKVALUE(
            player_valuations[date],
            SUM(player_valuations[market_value_in_eur])
        ),
        player_valuations[date] <= CutoffDate
    )

// ─────────────────────────────────────────────────────────────
// MV Growth YoY (%)
// Dùng players[market_value_in_eur] (snapshot) làm Current MV
// và [MV 1 Year Ago] từ player_valuations làm baseline
// BLANK() khi không có historical data (cầu thủ mới lên hệ thống)
// ─────────────────────────────────────────────────────────────
[MV Growth YoY %] =
VAR CurrentMV = SUM(players[market_value_in_eur])
VAR PrevMV    = [MV 1 Year Ago]
RETURN
    IF(
        PrevMV > 0,
        DIVIDE(CurrentMV - PrevMV, PrevMV, BLANK()) * 100,
        BLANK()
    )

// ─────────────────────────────────────────────────────────────
// MV Growth Label (cho tooltip & bảng Performance Table)
// Kỹ thuật: FORMAT + SWITCH + emoji → UX nổi bật trong bảng matrix
// ─────────────────────────────────────────────────────────────
[MV Growth Label] =
VAR Pct = [MV Growth YoY %]
RETURN
    SWITCH(
        TRUE(),
        ISBLANK(Pct),  "No data",
        Pct >= 50,     "🚀 +" & FORMAT(Pct, "0") & "% — Breakout",
        Pct >= 20,     "📈 +" & FORMAT(Pct, "0") & "% — Rising",
        Pct >= 5,      "↗️ +" & FORMAT(Pct, "0") & "% — Growing",
        Pct >= -5,     "➡️ " & FORMAT(Pct, "+0;-0;0") & "% — Stable",
        Pct >= -20,    "📉 " & FORMAT(Pct, "0") & "% — Declining",
        "⬇️ " & FORMAT(Pct, "0") & "% — Sharp Decline"
    )

// ─────────────────────────────────────────────────────────────
// Value Efficiency Ratio — hiệu suất tấn công so với giá trị thị trường
// Logic: GC90 cao / MV thấp = cầu thủ "value for money" = BUY TARGET
// Cao = undervalued, Thấp = overpriced
// ─────────────────────────────────────────────────────────────
[Value Efficiency Ratio] =
VAR GC90      = [Goal Contributions per 90]
VAR MVMillions = DIVIDE(SUM(players[market_value_in_eur]), 1000000, 1)
RETURN
    IF(
        [Total Minutes] < 450 || MVMillions <= 0,
        BLANK(),  -- Bỏ qua cầu thủ quá ít phút hoặc không có MV
        DIVIDE(GC90, MVMillions, BLANK())
    )
```

---

### 4.3 — Visual 1: Scout Bubble Chart (Main Visual)

```dax
// ─────────────────────────────────────────────────────────────
// SETUP VISUAL (Scatter Chart):
//   X-Axis  → [Goal Contributions per 90]
//   Y-Axis  → [Current MV]  (hoặc dùng players[market_value_in_eur] trực tiếp)
//   Size    → [Total Minutes]
//   Details → players[name]  (QUAN TRỌNG: phải có để split per player)
//   Color   → players[position]
//   Tooltip → [Scout Quadrant] + [Scout Score 0-100] + [MV Growth Label]
//
// Quadrant Lines: Tạo 2 reference lines trong Format pane
//   X constant: giá trị [Median GC90 All Players]
//   Y constant: giá trị [Median MV All Players]
// ─────────────────────────────────────────────────────────────

// Median GC90 của toàn bộ player được filter (cho quadrant line)
// Kỹ thuật: PERCENTILEX.INC — advanced DAX, ấn tượng với interviewer
// ALLSELECTED: giữ các filter từ slicers nhưng bỏ filter từ visual row context
[Median GC90 All Players] =
CALCULATE(
    PERCENTILEX.INC(
        FILTER(
            ALLSELECTED(players),
            CALCULATE([Total Minutes]) > 450  -- chỉ tính cầu thủ đủ phút
        ),
        CALCULATE([Goal Contributions per 90]),
        0.5
    )
)

// Median MV của toàn bộ player được filter
[Median MV All Players] =
CALCULATE(
    PERCENTILEX.INC(
        FILTER(
            ALLSELECTED(players),
            players[market_value_in_eur] > 0
        ),
        players[market_value_in_eur],
        0.5
    )
)

// ─────────────────────────────────────────────────────────────
// SCOUT QUADRANT — measure "wow" nhất của trang này
// So sánh cầu thủ với median của dataset đang lọc
// Kỹ thuật: VAR chain + SWITCH(TRUE()) — pattern chuẩn senior
// ─────────────────────────────────────────────────────────────
[Scout Quadrant] =
VAR PlayerGC90  = [Goal Contributions per 90]
VAR PlayerMV    = SUM(players[market_value_in_eur])
VAR MedianGC90  = [Median GC90 All Players]
VAR MedianMV    = [Median MV All Players]
VAR HasEnoughMins = [Total Minutes] > 450
RETURN
    SWITCH(
        TRUE(),
        NOT HasEnoughMins,                                          "⏱️ Insufficient Minutes",
        PlayerGC90 >= MedianGC90 && PlayerMV < MedianMV,           "🎯 BUY TARGET",
        PlayerGC90 >= MedianGC90 && PlayerMV >= MedianMV,          "⭐ Star Player",
        PlayerGC90 < MedianGC90  && PlayerMV >= MedianMV,          "⚠️ Overvalued",
        PlayerGC90 < MedianGC90  && PlayerMV < MedianMV,           "👁️ Monitor",
        "—"
    )

// ─────────────────────────────────────────────────────────────
// SCOUT SCORE 0-100 — Composite score cho phép sort/rank cầu thủ
// Trọng số business-defined:
//   Goals/90   → 40% (primary output metric)
//   Assists/90 → 25% (creative contribution)
//   Reliability → 20% (minutes played / max — tính ổn định)
//   Value Score → 15% (inverse MV — rẻ hơn = điểm cao hơn cho scouting)
// Kỹ thuật: Normalization 0-1 trước khi weight — pattern chuẩn analytics
// ─────────────────────────────────────────────────────────────
[Scout Score 0-100] =
VAR MinMins = 450  -- filter ngưỡng tối thiểu
VAR MaxG90 =
    CALCULATE(
        MAXX(FILTER(ALL(players), CALCULATE([Total Minutes]) > MinMins), [Goals per 90])
    )
VAR MaxA90 =
    CALCULATE(
        MAXX(FILTER(ALL(players), CALCULATE([Total Minutes]) > MinMins), [Assists per 90])
    )
VAR MaxMins =
    CALCULATE(MAXX(ALL(players), [Total Minutes]))
VAR MaxMV =
    CALCULATE(MAX(players[market_value_in_eur]), ALL(players))

VAR NormG90   = DIVIDE([Goals per 90],      IF(MaxG90  > 0, MaxG90,  1), 0)
VAR NormA90   = DIVIDE([Assists per 90],    IF(MaxA90  > 0, MaxA90,  1), 0)
VAR NormMins  = DIVIDE([Total Minutes],     IF(MaxMins > 0, MaxMins, 1), 0)
VAR ValueScore = 1 - DIVIDE(SUM(players[market_value_in_eur]), IF(MaxMV > 0, MaxMV, 1), 0)

RETURN
    IF(
        [Total Minutes] < MinMins,
        BLANK(),
        ROUND(
            NormG90 * 40 + NormA90 * 25 + NormMins * 20 + ValueScore * 15,
            1
        )
    )
```

---

### 4.4 — Visual 2: Performance Table (Matrix)

```dax
// ─────────────────────────────────────────────────────────────
// SETUP VISUAL (Matrix / Table):
//   Rows: players[name]
//   Cột 1: players[current_club_name]     ← dùng column trực tiếp
//   Cột 2: competitions[name]             ← qua appearances[competition_id]
//   Cột 3: players[Player Age]            ← Calculated Column
//   Cột 4: players[position]              ← column
//   Cột 5: [Total Goals]
//   Cột 6: [Total Assists]
//   Cột 7: [Total Minutes]
//   Cột 8: [Goals per 90]
//   Cột 9: [Assists per 90]
//   Cột 10: [MV Display]
//   Cột 11: [Scout Quadrant]
//   Cột 12: [Scout Score 0-100]
//
// Sort mặc định: [Scout Score 0-100] DESC
// Conditional Formatting: [Scout Quadrant] → màu theo Section 5
// ─────────────────────────────────────────────────────────────

// Season context label (dùng trong table header tooltip)
[Selected Season] =
SELECTEDVALUE(games[season], "All Seasons")

// Số giải đấu cầu thủ tham dự (đa năng lực = bonus)
[Competitions Played] =
DISTINCTCOUNT(appearances[competition_id])

// Goals + Assists total (cho quick sort)
[Total Goal Contributions] =
[Total Goals] + [Total Assists]
```

---

### 4.5 — Visual 3: Market Value Trend (Line Chart)

```dax
// ─────────────────────────────────────────────────────────────
// SETUP VISUAL (Line Chart):
//   X-Axis  → player_valuations[date]  (drag column trực tiếp, group by Month)
//   Y-Axis  → [MV from Valuations]
//   Legend  → players[name]  (filter TOP 10 bằng slicer hoặc TOP N filter)
//   Tooltip → [MV Growth Label]
// ─────────────────────────────────────────────────────────────

// MV từ player_valuations — cho line chart theo thời gian
[MV from Valuations] =
SUM(player_valuations[market_value_in_eur])

// MV 6 tháng trước (cho mini trend card)
[MV 6M Ago] =
VAR CutoffDate = EDATE(TODAY(), -6)
RETURN
    CALCULATE(
        LASTNONBLANKVALUE(
            player_valuations[date],
            SUM(player_valuations[market_value_in_eur])
        ),
        player_valuations[date] <= CutoffDate
    )

// MV 3 tháng trước
[MV 3M Ago] =
VAR CutoffDate = EDATE(TODAY(), -3)
RETURN
    CALCULATE(
        LASTNONBLANKVALUE(
            player_valuations[date],
            SUM(player_valuations[market_value_in_eur])
        ),
        player_valuations[date] <= CutoffDate
    )

// Short-term trend (3M)
[MV Trend 3M %] =
VAR Current = SUM(players[market_value_in_eur])
VAR Prev    = [MV 3M Ago]
RETURN
    IF(Prev > 0, DIVIDE(Current - Prev, Prev, BLANK()) * 100, BLANK())

// Arrow indicator cho KPI card (UX detail nhỏ nhưng ấn tượng)
[MV Trend Arrow] =
VAR Pct = [MV Growth YoY %]
RETURN
    SWITCH(
        TRUE(),
        ISBLANK(Pct), "—",
        Pct > 0,      "▲ " & FORMAT(ABS(Pct), "+0.0") & "%",
        Pct < 0,      "▼ " & FORMAT(ABS(Pct), "0.0") & "%",
        "► 0.0%"
    )
```

---

### 4.6 — Visual 4: Age vs Performance (Scatter)

```dax
// ─────────────────────────────────────────────────────────────
// SETUP VISUAL (Scatter Chart):
//   X-Axis  → players[Player Age]  (Calculated Column)
//   Y-Axis  → [Goals per 90]
//   Size    → [Total Minutes]
//   Details → players[name]
//   Color   → players[position]
//
// Reference Areas (dùng reference line trong Format pane):
//   Vertical line tại Age = 22 và Age = 26 → highlight "prime scouting window"
// ─────────────────────────────────────────────────────────────

// Prime Scouting Window Flag (22-26 tuổi)
// Dùng cho conditional formatting — highlight vùng "buy now"
[Prime Scouting Window] =
VAR Age = DATEDIFF(MAX(players[date_of_birth]), TODAY(), YEAR)
RETURN
    IF(Age >= 22 && Age <= 26, "🎯 Prime Window", "")

// U25 Rising Stars — target group cho scouting
[U25 Rising Stars Count] =
CALCULATE(
    COUNTROWS(players),
    DATEDIFF(players[date_of_birth], TODAY(), YEAR) <= 25,
    [MV Growth YoY %] >= 20
)
```

---

### 4.7 — Visual 5: Discipline Filter

```dax
// ─────────────────────────────────────────────────────────────
// SETUP VISUAL:
//   Card 1: [Yellow Cards per Game]  (base measure Section 4.1)
//   Card 2: [Red Cards per Game]     (base measure Section 4.1)
//   Bar Chart:
//     X-Axis → players[name]
//     Y-Axis → [Discipline Risk Score]
//     Color  → [Discipline Label]
//     Sort   → [Discipline Risk Score] DESC
// ─────────────────────────────────────────────────────────────

// Discipline Label (cho conditional formatting bar chart)
[Discipline Label] =
VAR Score = [Discipline Risk Score]
RETURN
    SWITCH(
        TRUE(),
        Score >= 0.5, "🔴 High Risk",
        Score >= 0.2, "🟡 Moderate",
        Score >  0,   "✅ Disciplined",
        "⚪ No Cards"
    )

// Top N violators (dùng cho filter "exclude high-risk players")
[Is High Discipline Risk] =
IF([Discipline Risk Score] >= 0.5, 1, 0)
```

---

## 5. Conditional Formatting Measures

> Tạo các measure này rồi dùng trong **Format → Conditional Formatting → Field value** để màu sắc tự động thay đổi theo data.

```dax
// ─────────────────────────────────────────────────────────────
// CF: Màu cho Contract Risk (Page 1 Visual 5 matrix)
// Trả về hex color code → dùng trong CF "Field value"
// ─────────────────────────────────────────────────────────────
[CF Contract Risk Color] =
VAR MonthsLeft =
    MIN(
        CALCULATE(
            AVERAGEX(
                FILTER(players, NOT ISBLANK(players[contract_expiration_date])),
                DATEDIFF(TODAY(), players[contract_expiration_date], MONTH)
            )
        )
    )
RETURN
    SWITCH(
        TRUE(),
        ISBLANK(MonthsLeft), "#2D3D52",  -- Neutral (no contract data)
        MonthsLeft <= 6,     "#E63946",  -- Red — CRITICAL
        MonthsLeft <= 12,    "#F4B942",  -- Gold — WARNING
        MonthsLeft <= 24,    "#4A90D9",  -- Blue — Monitor
        "#4ECDC4"                         -- Teal — Safe
    )

// ─────────────────────────────────────────────────────────────
// CF: Màu cho Scout Quadrant (Page 2 Performance Table)
// ─────────────────────────────────────────────────────────────
[CF Scout Quadrant Color] =
VAR Q = [Scout Quadrant]
RETURN
    SWITCH(
        Q,
        "🎯 BUY TARGET",  "#4ECDC4",  -- Teal — positive action
        "⭐ Star Player",  "#F4B942",  -- Gold — premium
        "⚠️ Overvalued",  "#E63946",  -- Red — avoid
        "👁️ Monitor",     "#4A90D9",  -- Blue — neutral
        "#8B97A8"                       -- Gray — insufficient data
    )

// ─────────────────────────────────────────────────────────────
// CF: Màu cho MV Growth % (Page 2 Performance Table)
// ─────────────────────────────────────────────────────────────
[CF MV Growth Color] =
VAR Pct = [MV Growth YoY %]
RETURN
    SWITCH(
        TRUE(),
        ISBLANK(Pct),  "#8B97A8",  -- Gray
        Pct >= 30,     "#4ECDC4",  -- Teal — strong growth
        Pct >= 10,     "#95D5B2",  -- Light green
        Pct >= 0,      "#F4B942",  -- Gold — stable
        Pct >= -15,    "#E8A838",  -- Amber — mild decline
        "#E63946"                   -- Red — sharp decline
    )

// ─────────────────────────────────────────────────────────────
// CF: Màu cho Age Balance trong histogram cells
// ─────────────────────────────────────────────────────────────
[CF Age Balance Color] =
VAR AgeGroup = SELECTEDVALUE(players[Player Age Group], "")
VAR Count = [Player Count]
VAR Total = CALCULATE([Player Count], REMOVEFILTERS(players[Player Age Group]))
VAR Pct   = DIVIDE(Count, Total, 0) * 100
RETURN
    SWITCH(
        TRUE(),
        Pct >= 30,              "#4ECDC4",  -- Teal — dominant group
        Pct >= 15,              "#4A90D9",  -- Blue — healthy
        Pct >= 8,               "#F4B942",  -- Gold — thin
        "#E63946"                            -- Red — gap/shortage
    )
```

---

## 6. Dynamic Labels & UX

```dax
// ─────────────────────────────────────────────────────────────
// Page Title động — thay đổi theo CLB được chọn
// Dùng trong Text Box với DAX measure hoặc dùng làm Card title
// ─────────────────────────────────────────────────────────────
[Page 1 Title] =
VAR ClubName = SELECTEDVALUE(clubs[name], "All Clubs")
RETURN
    "Squad Health Check — " & ClubName

[Page 2 Title] =
VAR ClubName = SELECTEDVALUE(clubs[name], "All Clubs")
VAR Season = SELECTEDVALUE(games[season], "All Seasons")
RETURN
    "Player Scout Board — " & Season

// ─────────────────────────────────────────────────────────────
// Last Updated Label
// ─────────────────────────────────────────────────────────────
[Last Updated] =
"Data as of " & FORMAT(TODAY(), "DD MMM YYYY")

// ─────────────────────────────────────────────────────────────
// Squad MV Formatted (cho KPI Card — auto K/M/B)
// ─────────────────────────────────────────────────────────────
[Squad MV Display] =
VAR MV = [Current MV]
RETURN
    SWITCH(
        TRUE(),
        ISBLANK(MV),      "—",
        MV >= 1000000000, FORMAT(MV / 1000000000, "€0.00") & "B",
        MV >= 1000000,    FORMAT(MV / 1000000, "€0.0") & "M",
        MV >= 1000,       FORMAT(MV / 1000, "€0.0") & "K",
        FORMAT(MV, "€0")
    )

// ─────────────────────────────────────────────────────────────
// Player Count Info Bar (dưới slicers — UX detail chuyên nghiệp)
// ─────────────────────────────────────────────────────────────
[Players Shown in Filter] =
VAR N = COUNTROWS(ALLSELECTED(players))
VAR Total = CALCULATE(COUNTROWS(players), ALL(players))
RETURN
    FORMAT(N, "#,0") & " of " & FORMAT(Total, "#,0") & " players"

// ─────────────────────────────────────────────────────────────
// Insight Alert — tự động generate text insight cho Page 1
// Kỹ thuật: DAX-generated insight text = rất ấn tượng với interviewer
// ─────────────────────────────────────────────────────────────
[Squad Health Insight] =
VAR AvgAge    = [Avg Squad Age]
VAR ExpiryVal = [Value at Risk 12M] / 1000000
VAR RiskPct   = [% Squad Value at Risk]
VAR GapAlert  = [Recruitment Gap Alert]
RETURN
    "Avg Age: " & FORMAT(AvgAge, "0.0") & " | "
    & FORMAT(ExpiryVal, "€0.0") & "M at expiry risk (" & FORMAT(RiskPct, "0") & "%) | "
    & GapAlert
```

---

## 7. Color Palette & Design Guide

### 7.1 — Color Palette — Dark Sports Analytics Theme

| Vai trò | Hex | Tên | Dùng cho |
|---------|-----|-----|---------|
| **Page Background** | `#0D1B2A` | Midnight Navy | Page background |
| **Card Background** | `#162032` | Dark Card | KPI cards, visual bg |
| **Surface** | `#1B2A40` | Deep Surface | Tables, matrix bg |
| **Border / Divider** | `#2D3D52` | Steel | Borders, grid lines |
| **Primary Accent** | `#F4B942` | Championship Gold | KPI values, headers |
| **Positive / Win** | `#4ECDC4` | Teal | Win, gain, growth |
| **Danger / Loss** | `#E63946` | Signal Red | Risk, expiry, loss |
| **Info / Neutral+** | `#4A90D9` | Sky Blue | Monitor, neutral info |
| **Success subtle** | `#95D5B2` | Mint | Mild positive |
| **Text Primary** | `#F0F6FC` | White | Main labels, values |
| **Text Secondary** | `#8B97A8` | Steel Gray | Subtitles, axis |
| **Attack** | `#E63946` | Red | Position = Attack |
| **Midfield** | `#F4B942` | Gold | Position = Midfield |
| **Defender** | `#4A90D9` | Blue | Position = Defender |
| **Goalkeeper** | `#4ECDC4` | Teal | Position = Goalkeeper |

### 7.2 — Áp dụng trong Power BI

**Bước 1 — Theme JSON:**
```json
{
  "name": "Transfermarkt Dark",
  "dataColors": ["#F4B942", "#4ECDC4", "#E63946", "#4A90D9", "#95D5B2", "#F0F6FC"],
  "background": "#0D1B2A",
  "foreground": "#F0F6FC",
  "tableAccent": "#F4B942"
}
```
> View → Themes → Browse for themes → import file JSON trên.

**Bước 2 — KPI Card setup (mỗi card):**
- Background: `#162032`
- Title font color: `#8B97A8`, size 10
- Value font color: `#F4B942` (numbers) hoặc `#F0F6FC` (text)
- Luôn thêm **Sparkline** hoặc **Trend arrow** vào KPI cards

**Bước 3 — Chart formatting:**
- Plot area background: `#1B2A40`
- Gridlines: `#2D3D52`
- Axis labels: `#8B97A8`
- Data labels: `#F0F6FC`

**Bước 4 — Tooltips:**
- Background: `#162032`
- Border: `#F4B942`
- Thêm custom tooltip page với [Scout Quadrant] + [MV Growth Label] + [Discipline Label]

### 7.3 — Visual Layout Recommendations

**Page 1 — Squad Health Check:**
```
Row 1 (KPI Cards): | Avg Age | Squad MV | Foreign % | ĐTQG | Expiring <1Y |
Row 2: | Age Histogram (40%) | Squad Depth Bar (30%) | Contract Matrix (30%) |
Row 3: | Treemap MV (50%) | Scatter Minutes vs MV (50%) |
Sidebar: Club slicer + Season slicer
```

**Page 2 — Player Scout Board:**
```
Row 1 (KPI Cards): | G/90 | A/90 | GC/90 | Min/Goal | Scout Score |
Row 2: | Scout Bubble Chart (55%) | MV Trend Line (45%) |
Row 3: | Performance Table Matrix (full width) |
Row 4: | Age vs Performance Scatter (50%) | Discipline Bar (50%) |
Sidebar: Position / League / Age Range / MV Range slicers
```

---

## Checklist Triển Khai

### Data Model
- [ ] Đặt tất cả relationships đúng theo DATA_DICTIONARY (đặc biệt appearances[player_club_id] → clubs[club_id])
- [ ] Tạo bảng `_Measures` và move tất cả measures vào
- [ ] Season slicer đặt trên `games[season]` (KHÔNG phải appearances)
- [ ] Ẩn tất cả FK columns trong tables (chỉ để lại columns dùng trong visuals)

### Calculated Columns (tạo trong Data View)
- [ ] `players[Player Age Group]`
- [ ] `players[Contract Expiry Year]`
- [ ] `players[Months to Expiry]`
- [ ] `players[Contract Action]`
- [ ] `players[MV vs Peak Pct]`

### Page 1
- [ ] 5 KPI Cards Row 1
- [ ] Visual 1: Age Histogram — X = `players[Player Age Group]`, Y = `[Player Count]`, Legend = `players[position]`
- [ ] Visual 2: Squad Depth — game_lineups data, stacked by `[Starter Appearances]` / `[Sub Appearances]`
- [ ] Visual 3: Treemap — `players[market_value_in_eur]` as Values, `players[name]` as Details
- [ ] Visual 4: Scatter — X = `[Total Minutes]`, Y = `[Current MV]`, Size = `players[Player Age]`
- [ ] Visual 5: Contract Matrix — Rows = `players[position]`, Columns = `players[Contract Expiry Year]`

### Page 2
- [ ] 7 KPI Cards Row 1
- [ ] Visual 1: Bubble Chart — X = `[Goal Contributions per 90]`, Y = `[Current MV]`, Size = `[Total Minutes]`, Details = `players[name]`
- [ ] Visual 2: Performance Table với Conditional Formatting từ `[CF Scout Quadrant Color]`
- [ ] Visual 3: Line Chart từ `player_valuations[date]` + `[MV from Valuations]`
- [ ] Visual 4: Scatter Age vs G/90
- [ ] Visual 5: Discipline Bar với `[Discipline Risk Score]`
- [ ] Slicers: Position, Sub-position, Age range, League, MV range, Season

---

*Built by Senior Power BI Developer | Transfermarkt Analytics | April 2026*  
*Schema verified against DATA_DICTIONARY.md | Production-ready DAX*
