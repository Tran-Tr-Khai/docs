# Transfermarkt Analytics — Power BI Dashboard Design
**Audience:** Sporting Director · Head of Recruitment · Data Analyst  
**Dataset:** Transfermarkt (12 CSV files, ~7.2M records)  
**Tool:** Microsoft Power BI Desktop  
**Phiên bản tài liệu:** v1.0 — April 2026

---

## Tổng quan kiến trúc Dashboard

| # | Tên trang | Business domain | Người dùng chính |
|---|-----------|-----------------|-----------------|
| 1 | **Squad Health Check** | Club Performance | Sporting Director |
| 2 | **Player Scout Board** | Player Performance | Head of Recruitment |
| 3 | **Transfer Investment ROI** | Transfer Strategy | CEO / CFO |
| 4 | **Market Value Radar** | Market Value Analysis | Agent · Scout |
| 5 | **Match Intelligence** | Match Analysis | Head Coach · Analyst |
| 6 | **Contract Risk Monitor** | Squad Management | Sporting Director |
| 7 | **League Benchmarking** | Club Performance | Board / Investor |

> **Mô hình dữ liệu (Power BI Data Model):** Star schema — bảng `games`, `players`, `clubs`, `competitions` là dimension trung tâm. Các bảng fact: `appearances`, `game_events`, `club_games`, `player_valuations`, `transfers`.

---

## Data Model — Relationships

```
competitions (dim)
    └── games (fact) ──< club_games (fact)
                   ──< game_events (fact)
                   ──< game_lineups (fact)

clubs (dim)
    └── players (dim) ──< appearances (fact)
                      ──< player_valuations (fact)
                      ──< transfers (fact)

countries (dim) ──< competitions
                └── national_teams (dim)
```

**Key joins cần thiết trong Power BI:**
- `appearances[game_id]` → `games[game_id]`
- `appearances[player_id]` → `players[player_id]`
- `appearances[player_club_id]` → `clubs[club_id]`
- `game_events[game_id]` → `games[game_id]`
- `game_events[player_id]` → `players[player_id]`
- `club_games[club_id]` → `clubs[club_id]`
- `transfers[player_id]` → `players[player_id]`
- `player_valuations[player_id]` → `players[player_id]`
- `games[competition_id]` → `competitions[competition_id]`
- `competitions[country_id]` → `countries[country_id]`

---

---

## TRANG 1 — Squad Health Check

### Business Question
> *"Đội hình của CLB hiện tại mạnh ở đâu, yếu ở đâu, và nguy cơ rủi ro nhân sự nào cần giải quyết ngay trong kỳ chuyển nhượng tới?"*

### Tại sao quan trọng (Decision Impact)
Một Sporting Director cần nhìn toàn diện đội hình — không chỉ về chất lượng cầu thủ mà còn về phân bổ độ tuổi, giá trị đội hình, tỷ lệ cầu thủ nội/ngoại, và số cầu thủ ĐTQG. Đây là trang mở đầu để ra quyết định "mua hay giữ" trong mùa hè.

### Bảng dữ liệu sử dụng
| Bảng | Vai trò | Join key |
|------|---------|----------|
| `clubs` | Thông tin tổng quan CLB | — |
| `players` | Danh sách cầu thủ | `players.current_club_id = clubs.club_id` |
| `appearances` | Số phút thi đấu mùa này | `appearances.player_id = players.player_id` |
| `player_valuations` | Giá trị thị trường mới nhất | `player_valuations.player_id = players.player_id` |

### KPI Cards (Row 1)
| KPI | Công thức DAX | Ý nghĩa |
|-----|--------------|---------|
| **Total Squad Market Value** | `SUM(player_valuations[market_value_in_eur])` — lấy giá trị mới nhất per player | Tổng giá trị đội hình |
| **Average Player Age** | `AVERAGE(players[date_of_birth])` → tính tuổi | Đội hình trẻ hay già? |
| **Foreign Player %** | `clubs[foreigners_percentage]` | Tuân thủ quota nội binh |
| **National Team Players** | `clubs[national_team_players]` | Số cầu thủ được triệu tập |
| **Players with Contract < 1 Year** | `COUNTROWS(FILTER(players, DATEDIFF(TODAY(), players[contract_expiration_date], YEAR) < 1))` | Rủi ro mất cầu thủ miễn phí |

### Visuals

**Visual 1 — Age Distribution Histogram (Bar Chart)**
- Trục X: Nhóm tuổi (U21, 21–25, 26–29, 30+)
- Trục Y: Số cầu thủ
- Color: Position (Attack / Midfield / Defender / GK)
- Insight: Xác định khoảng tuổi thiếu hụt (recruitment gap)

**Visual 2 — Squad Depth by Position (Stacked Bar)**
- Trục X: Position (GK, Defender, Midfield, Attack)
- Trục Y: Số cầu thủ
- Legend: Type (starting lineup frequency vs. substitute)
- Insight: Vị trí nào đang thiếu chiều sâu đội hình (depth issue)

**Visual 3 — Market Value by Player (Treemap)**
- Kích thước ô: `market_value_in_eur`
- Label: Tên cầu thủ + Vị trí
- Color: Position
- Insight: Top 5 cầu thủ chiếm bao nhiêu % tổng giá trị?

**Visual 4 — Minutes Played Distribution (Scatter Plot)**
- Trục X: Minutes played (mùa hiện tại, từ `appearances`)
- Trục Y: `market_value_in_eur`
- Size: `age`
- Filter: Chọn CLB / Mùa giải
- Insight: Cầu thủ đắt tiền nhưng ít ra sân = lãng phí nguồn lực

**Visual 5 — Contract Expiry Timeline (Gantt-style Bar)**
- Dùng bảng matrix: Cột = Năm hết hạn, Hàng = Position
- Insight: Bao nhiêu cầu thủ hết hạn năm tới theo vị trí?

### Insight kỳ vọng
- CLB có nhiều cầu thủ U21 nhưng thiếu cầu thủ 26–29 tuổi ở vị trí trung vệ → cần mua ngay
- 3–4 cầu thủ key có giá trị > 10M EUR sắp hết hạn hợp đồng năm tới → ưu tiên gia hạn

### Actionable Insight
- **Mua:** Xác định 1–2 vị trí cần bổ sung ngay (thiếu depth + cao tuổi)
- **Gia hạn:** List cầu thủ giá trị cao + hợp đồng ngắn
- **Thanh lý:** Cầu thủ ít phút thi đấu + hợp đồng còn dài = chi phí thừa

---

---

## TRANG 2 — Player Scout Board

### Business Question
> *"Cầu thủ nào trên thị trường (hoặc đang thi đấu ở các giải đất trong dataset) có hiệu suất cao, đang ở độ tuổi chín nhất, nhưng giá trị thị trường vẫn còn hợp lý — tức là cơ hội mua undervalued?"*

### Tại sao quan trọng (Decision Impact)
Đây là trang quan trọng nhất với Head of Recruitment. Mục tiêu: tìm cầu thủ "value for money" — hiệu suất tốt nhưng chưa bị định giá cao, hoặc giá trị đang tăng mạnh. Đây là nền tảng cho scouting report.

### Bảng dữ liệu sử dụng
| Bảng | Vai trò | Join key |
|------|---------|----------|
| `players` | Thông tin cơ bản | — |
| `appearances` | Thống kê hiệu suất mùa | `player_id` |
| `player_valuations` | Xu hướng giá trị | `player_id` |
| `game_events` | Bàn thắng, kiến tạo, thẻ | `player_id` |
| `clubs` | CLB hiện tại của cầu thủ | `current_club_id` |
| `competitions` | Giải đấu đang thi đấu | qua `appearances.competition_id` |

### KPI Cards (Row 1) — Context cho cầu thủ đang lọc
| KPI | Công thức |
|-----|-----------|
| **Goals per 90 min** | `DIVIDE([Total Goals], [Total Minutes], 0) * 90` |
| **Assists per 90 min** | `DIVIDE([Total Assists], [Total Minutes], 0) * 90` |
| **Goal Contributions per 90** | Goals/90 + Assists/90 |
| **Minutes per Goal** | `DIVIDE([Total Minutes], [Total Goals], 0)` |
| **Current Market Value** | `players[market_value_in_eur]` |
| **Value Growth (YoY)** | `(Latest MV - MV 1 year ago) / MV 1 year ago` — từ `player_valuations` |

### Visuals

**Visual 1 — Scout Radar / Bubble Chart (Main)**
- Trục X: Goal Contributions per 90
- Trục Y: `market_value_in_eur` (logarithmic scale)
- Size: `minutes_played` (tổng mùa)
- Color: `position`
- Quadrant lines: Median X và Median Y
- **Quadrant diễn giải:**
  - Bottom-right: Hiệu suất cao, giá thấp = **BUY TARGET**
  - Top-right: Hiệu suất cao, giá cao = Star players
  - Bottom-left: Hiệu suất thấp, giá thấp = Cần theo dõi thêm
  - Top-left: Hiệu suất thấp, giá cao = **Overvalued**

**Visual 2 — Performance Table (Matrix)**
Cột: Player | Club | League | Age | Pos | Goals | Assists | Mins | G/90 | A/90 | MV (EUR)  
Có thể sort và filter linh hoạt

**Visual 3 — Market Value Trend (Line Chart)**
- Trục X: Date (từ `player_valuations`)
- Trục Y: `market_value_in_eur`
- Legend: Top 10 players được chọn
- Insight: Cầu thủ nào đang trending up mạnh?

**Visual 4 — Age vs. Performance (Scatter)**
- Trục X: Age
- Trục Y: Goals per 90
- Color: Position
- Highlight: Age 22–26 = prime scouting window
- Insight: Tìm cầu thủ đang ở peak age với hiệu suất tốt

**Visual 5 — Discipline Filter (Card + Bar)**
- Yellow cards per game, Red cards per game
- Dùng để lọc ra cầu thủ kỷ luật kém (high risk)

### Filters / Slicers
- Position (Attack / Midfield / Defender / GK)
- Sub-position (Centre-Forward, Left Winger, …)
- Age range (slider)
- League / Competition
- Market value range (slider)
- Nationality / Confederation
- Season

### Insight kỳ vọng
- Phát hiện 5–10 tiền đạo tuổi 22–25, Goals/90 > 0.5, giá trị < 5M EUR → undervalued targets
- Cầu thủ đang thi đấu ở giải hạng 2 nhưng hiệu suất ngang cầu thủ giải top → cơ hội mua rẻ

### Actionable Insight
- Export shortlist từ bảng Performance Table → gửi scout đi xem trực tiếp
- So sánh cầu thủ target với cầu thủ hiện tại của CLB (dùng bookmark/compare mode)

---

---

## TRANG 3 — Transfer Investment ROI

### Business Question
> *"CLB đã chi bao nhiêu tiền chuyển nhượng, và khoản đầu tư đó có sinh ra giá trị (value creation) hay không? Mùa nào CLB mua/bán hiệu quả nhất?"*

### Tại sao quan trọng (Decision Impact)
CEO và CFO cần biết Transfer Strategy có hiệu quả về mặt tài chính không. Mua cầu thủ đắt nhưng bán rẻ hơn là đang đốt tiền. Trang này cũng giúp so sánh CLB với đối thủ cạnh tranh.

### Bảng dữ liệu sử dụng
| Bảng | Vai trò | Join key |
|------|---------|----------|
| `transfers` | Lịch sử mua/bán | — |
| `players` | Thông tin cầu thủ | `transfers.player_id = players.player_id` |
| `player_valuations` | MV tại thời điểm chuyển nhượng | `player_id` + match theo date |
| `clubs` | Thông tin CLB mua/bán | `from_club_id`, `to_club_id` |
| `appearances` | Hiệu suất sau khi mua | `player_id` |

### Measures DAX quan trọng

```dax
-- Tổng tiền chi ra (mua cầu thủ vào)
Transfer Spend =
CALCULATE(
    SUM(transfers[transfer_fee]),
    transfers[to_club_id] = SELECTEDVALUE(clubs[club_id])
)

-- Tổng tiền thu về (bán cầu thủ ra)
Transfer Income =
CALCULATE(
    SUM(transfers[transfer_fee]),
    transfers[from_club_id] = SELECTEDVALUE(clubs[club_id])
)

-- Net Transfer Balance
Net Transfer Balance = [Transfer Income] - [Transfer Spend]

-- Transfer ROI (giá trị tạo ra so với chi phí)
-- So sánh MV hiện tại của cầu thủ mua vào với giá mua
Transfer ROI % =
DIVIDE(
    SUM(players[market_value_in_eur]) - [Transfer Spend],
    [Transfer Spend],
    0
) * 100

-- Mua hớ / mua hời
Value vs Fee Gap = transfers[market_value_in_eur] - transfers[transfer_fee]
-- Dương = mua rẻ hơn giá trị thị trường (smart buy)
-- Âm = trả premium
```

### KPI Cards
| KPI | Giải thích |
|-----|------------|
| **Net Transfer Balance** | Dương = CLB thu nhiều hơn chi |
| **Avg Fee vs Avg MV** | Trung bình CLB trả bao nhiêu % so với MV |
| **Best Deal (highest ROI buy)** | Cầu thủ được mua có giá trị tăng nhiều nhất |
| **Worst Deal (biggest loss)** | Cầu thủ bán rẻ hơn hoặc mua quá đắt |
| **Total Transactions (season)** | Số lượng giao dịch mùa được chọn |

### Visuals

**Visual 1 — Transfer Balance by Season (Waterfall Chart)**
- Trục X: Mùa giải (transfer_season)
- Trục Y: Net balance (EUR)
- Màu xanh = net dương, đỏ = net âm
- Insight: Mùa nào CLB đầu tư nhiều? Mùa nào thu hồi vốn?

**Visual 2 — Buy vs Sell Scatter (Scatter Plot)**
- Trục X: `transfer_fee` (phí thực tế)
- Trục Y: `market_value_in_eur` (tại thời điểm chuyển nhượng)
- Color: Mua (xanh) vs Bán (đỏ)
- Đường 45° = mua/bán đúng giá trị
- Điểm trên đường = bán đắt hơn MV / mua rẻ hơn MV (tốt)
- Insight: CLB có xu hướng mua premium hay mua smart?

**Visual 3 — Top Transfers by Fee (Horizontal Bar)**
- Top 10 thương vụ mua đắt nhất của CLB được chọn
- Label: Tên cầu thủ + Mùa + Fee
- Color: Position

**Visual 4 — League Spending Comparison (Bar Chart)**
- Trục X: CLB (top 20 theo spending)
- Trục Y: Net transfer spend
- Color: Confederation / League
- Insight: CLB đang ở đâu trong bức tranh tổng thể thị trường?

**Visual 5 — Transfer Fee vs. Performance Gained (Scatter)**
- Trục X: Transfer fee
- Trục Y: Goals + Assists per 90 sau khi mua (join `appearances`)
- Label: Tên cầu thủ
- Insight: Cầu thủ đắt có thực sự perform không?

**Visual 6 — Transfer Activity Timeline (Line Chart)**
- Trục X: Năm chuyển nhượng
- Trục Y: Tổng giá trị giao dịch
- Legend: Transfer In (mua) vs Transfer Out (bán)
- Insight: CLB đang trong giai đoạn xây dựng hay tái cơ cấu?

### Insight kỳ vọng
- 60% giao dịch mua của CLB là trả premium so với MV → thiếu thương lượng
- Một cầu thủ được mua 2M EUR, hiện MV = 15M EUR → tạo ra 13M EUR giá trị tiềm năng
- 3 mùa gần nhất Net Balance âm → CLB cần tìm kiếm bán cầu thủ để tái cân bằng

### Actionable Insight
- Xác định cầu thủ "ready to sell" — MV cao, phút thi đấu ít, hợp đồng còn dài → bán trước khi hết phong độ
- Tìm CLB bán rẻ (distress selling) để mua smart

---

---

## TRANG 4 — Market Value Radar

### Business Question
> *"Giá trị thị trường của cầu thủ và đội hình đang thay đổi như thế nào theo thời gian? CLB nào đang tăng giá trị nhanh nhất? Cầu thủ nào đang ở đỉnh giá trị và sắp giảm?"*

### Tại sao quan trọng (Decision Impact)
Market value là proxy quan trọng cho chất lượng cầu thủ và sức mạnh CLB. Theo dõi trend giúp ra quyết định đúng thời điểm: bán cầu thủ ở đỉnh, mua cầu thủ đang tăng trước khi giá vọt.

### Bảng dữ liệu sử dụng
| Bảng | Vai trò | Join key |
|------|---------|----------|
| `player_valuations` | Lịch sử giá trị theo ngày | — |
| `players` | Thông tin cầu thủ | `player_id` |
| `clubs` | CLB cầu thủ đang thuộc | `current_club_id` |
| `competitions` | Giải đấu | qua `player_valuations.player_club_domestic_competition_id` |

### Measures DAX quan trọng

```dax
-- MV mới nhất của mỗi cầu thủ
Latest MV =
CALCULATE(
    LASTNONBLANK(player_valuations[market_value_in_eur], 1),
    ALLEXCEPT(player_valuations, player_valuations[player_id])
)

-- Thay đổi MV so với 1 năm trước
MV Change YoY =
VAR CurrentMV = [Latest MV]
VAR MV1YearAgo =
    CALCULATE(
        LASTNONBLANK(player_valuations[market_value_in_eur], 1),
        DATEADD(player_valuations[date], -365, DAY)
    )
RETURN DIVIDE(CurrentMV - MV1YearAgo, MV1YearAgo, 0) * 100

-- Peak MV của cầu thủ
Peak MV = MAX(player_valuations[market_value_in_eur])

-- % còn lại so với đỉnh (decay metric)
MV vs Peak % = DIVIDE([Latest MV], [Peak MV], 0) * 100

-- Tổng giá trị squad của CLB
Squad Total MV =
SUMX(
    VALUES(players[player_id]),
    [Latest MV]
)
```

### KPI Cards
| KPI | Ý nghĩa |
|-----|---------|
| **Most Valuable Player (selected club)** | Top cầu thủ giá trị nhất |
| **Avg MV Change YoY (squad)** | Đội hình đang tăng hay giảm giá trị? |
| **Players at/near Peak MV** | Cơ hội bán (sell high) |
| **Players with MV < 50% of Peak** | Cầu thủ đang đi xuống rõ rệt |
| **League Total Market Value** | So sánh sức mạnh tài chính giữa các giải |

### Visuals

**Visual 1 — Squad Market Value Timeline (Area Chart)**
- Trục X: Date (năm/tháng)
- Trục Y: Tổng MV đội hình
- Filter: Chọn CLB
- Legend: Có thể chồng nhiều CLB để so sánh
- Insight: CLB nào đang phát triển nhanh nhất về giá trị?

**Visual 2 — Player Value Lifecycle (Line Chart + Reference Line)**
- Trục X: Date
- Trục Y: `market_value_in_eur`
- Line: Từng cầu thủ được chọn
- Reference line: Peak value
- Insight: Cầu thủ đang ở giai đoạn nào của vòng đời giá trị?

**Visual 3 — MV Growth Leaderboard (Bar Chart — Top 20)**
- Cầu thủ có % tăng MV cao nhất trong 12 tháng qua
- Color: Position / League
- Insight: Ai đang "hot" nhất trên thị trường

**Visual 4 — MV vs Peak Heatmap (Matrix)**
- Hàng: Vị trí
- Cột: CLB hoặc Giải đấu
- Giá trị: Trung bình `MV vs Peak %`
- Màu: Gradient xanh (cao) → đỏ (thấp)
- Insight: Giải đấu nào đang có nhiều cầu thủ ở đỉnh form?

**Visual 5 — League Value Comparison (Stacked Bar)**
- Trục X: Giải đấu (competition)
- Trục Y: Tổng Market Value
- Stack: Theo Position
- Insight: Premier League vs Bundesliga vs La Liga về sức mạnh tài chính

**Visual 6 — Sell Window Alert Table**
- Cầu thủ có `MV vs Peak % > 95%` + Age > 27 → Nên cân nhắc bán
- Cột: Tên, Vị trí, CLB, Tuổi, MV hiện tại, Peak MV, % so với đỉnh

### Insight kỳ vọng
- Premier League có tổng MV gấp 2.5x Bundesliga → khoảng cách tài chính ngày càng lớn
- 30% cầu thủ trên 29 tuổi trong dataset đang ở mức MV < 50% so với đỉnh → đang trong giai đoạn decline
- Một số tiền vệ từ giải Eredivisie tăng MV > 200% trong 2 năm → scouting hotspot

### Actionable Insight
- **Sell High:** List cầu thủ đang ở peak, tuổi > 27 → bán mùa hè này
- **Buy Rising:** Đầu tư vào cầu thủ MV đang tăng mạnh nhưng chưa bị để ý bởi big clubs
- **Scouting Market:** Xác định giải đấu có nhiều "emerging talent" (giải Eredivisie, Liga NOS)

---

---

## TRANG 5 — Match Intelligence

### Business Question
> *"CLB đang thể hiện như thế nào trong các trận đấu? Các mẫu sự kiện trận đấu (bàn thắng muộn, thẻ đỏ, thay người) ảnh hưởng thế nào đến kết quả? Đâu là điểm yếu chiến thuật cần khắc phục?"*

### Tại sao quan trọng (Decision Impact)
Head Coach và Data Analyst cần hiểu pattern hiệu suất theo thời gian thực, theo sân, theo đối thủ, và theo giai đoạn mùa giải để điều chỉnh chiến thuật.

### Bảng dữ liệu sử dụng
| Bảng | Vai trò | Join key |
|------|---------|----------|
| `games` | Kết quả và thông tin trận | — |
| `club_games` | Thống kê theo góc nhìn CLB | `game_id` |
| `game_events` | Sự kiện (bàn thắng, thẻ, thay người) | `game_id` |
| `game_lineups` | Đội hình ra sân | `game_id` |
| `clubs` | Thông tin CLB | `club_id` |
| `competitions` | Giải đấu | `competition_id` |

### Measures DAX quan trọng

```dax
-- Tỷ lệ thắng
Win Rate =
DIVIDE(
    CALCULATE(COUNTROWS(club_games), club_games[is_win] = 1),
    COUNTROWS(club_games),
    0
) * 100

-- Bàn thắng trung bình mỗi trận
Avg Goals Scored = AVERAGE(club_games[own_goals])

-- Bàn thua trung bình mỗi trận
Avg Goals Conceded = AVERAGE(club_games[opponent_goals])

-- Clean sheets
Clean Sheets =
CALCULATE(
    COUNTROWS(club_games),
    club_games[opponent_goals] = 0
)

-- Late goals (phút 75+)
Late Goals =
CALCULATE(
    COUNTROWS(game_events),
    game_events[type] = "Goals",
    game_events[minute] >= 75
)

-- Red cards impact on result
Red Card Games Win Rate =
CALCULATE(
    [Win Rate],
    FILTER(game_events, game_events[type] = "Cards"
        && SEARCH("Red", game_events[description], 1, 0) > 0)
)
```

### KPI Cards
| KPI | Ý nghĩa |
|-----|---------|
| **Win Rate (Home vs Away)** | Sức mạnh sân nhà |
| **Avg Goals For / Against** | Tấn công vs Phòng thủ |
| **Clean Sheets %** | Độ vững chắc hàng thủ |
| **Points Per Game** | Form tổng thể |
| **Goals in Last 15 min %** | Tỷ lệ bàn thắng giai đoạn cuối |

### Visuals

**Visual 1 — Win/Draw/Loss Breakdown (Donut Chart)**
- Filter: CLB + Mùa giải + Home/Away
- Instant overview kết quả tổng thể

**Visual 2 — Goal Timeline Distribution (Bar Chart)**
- Trục X: Phút trong trận (bins: 1-15, 16-30, 31-45, 46-60, 61-75, 76-90, 90+)
- Trục Y: Số bàn thắng
- Legend: Ghi (For) vs Thủng lưới (Against)
- Insight: CLB mạnh/yếu giai đoạn nào trong trận?

**Visual 3 — Formation Performance Matrix (Matrix)**
- Hàng: `home_club_formation`
- Cột: Đối thủ formation
- Giá trị: Win rate, goals scored
- Insight: Sơ đồ nào hiệu quả nhất chống lại sơ đồ cụ thể?

**Visual 4 — Form Trend (Line Chart)**
- Trục X: Vòng đấu (round) / Ngày
- Trục Y: Rolling 5-game points (3 thắng, 1 hòa, 0 thua)
- Legend: CLB được chọn vs Top 3 đối thủ
- Insight: CLB đang trong giai đoạn form tốt hay sa sút?

**Visual 5 — Home vs Away Performance (Grouped Bar)**
- Trục X: Metric (Win Rate, Goals For, Goals Against, Clean Sheets)
- Legend: Home / Away
- Insight: Khoảng cách sân nhà vs sân khách

**Visual 6 — Event Impact Analysis (Scatter)**
- Trục X: Số thẻ vàng mỗi trận
- Trục Y: Điểm mỗi trận (outcome score)
- Size: Khán giả (attendance)
- Insight: Kỷ luật có ảnh hưởng đến kết quả không?

**Visual 7 — Top Performers per Game (Table)**
- Cầu thủ với Goals + Assists cao nhất mùa này
- Join từ `game_events` + `appearances`

### Insight kỳ vọng
- CLB ghi 40% bàn thắng trong giai đoạn 61–90 phút → lợi thế về thể lực
- Win rate sân khách chỉ 30% → cần điều chỉnh chiến thuật khi đá sân đối thủ
- Sơ đồ 4-3-3 của CLB thua nhiều trước đội dùng 3-5-2 → cần có plan B

### Actionable Insight
- **Chiến thuật:** Cải thiện thể lực giai đoạn đầu trận (nhiều bàn thua 0–30 phút)
- **Phòng thủ:** Bổ sung mua trung vệ nếu Clean Sheet Rate < 25%
- **Sân nhà:** Xây dựng đội hình khác biệt cho trận sân khách quan trọng

---

---

## TRANG 6 — Contract Risk Monitor

### Business Question
> *"CLB đang đối mặt với rủi ro mất bao nhiêu giá trị đội hình (triệu EUR) do hợp đồng sắp hết hạn? Cần ưu tiên gia hạn hoặc bán cầu thủ nào trước khi mất họ miễn phí?"*

### Tại sao quan trọng (Decision Impact)
Mất cầu thủ Bosman (hết hợp đồng không thu phí chuyển nhượng) là thất thoát tài chính khổng lồ. Trang này giúp Sporting Director có kế hoạch chủ động, tránh kịch bản "bán rẻ hoặc cho đi miễn phí".

### Bảng dữ liệu sử dụng
| Bảng | Vai trò | Join key |
|------|---------|----------|
| `players` | Hợp đồng + giá trị | — |
| `appearances` | Phút thi đấu (tầm quan trọng) | `player_id` |
| `player_valuations` | MV mới nhất | `player_id` |
| `clubs` | Thông tin CLB | `current_club_id` |

### KPI Cards
| KPI | Công thức |
|-----|-----------|
| **Value at Risk (Bosman next 12M)** | SUM(MV) của cầu thủ hết hạn < 1 năm |
| **Value at Risk (Bosman next 24M)** | SUM(MV) của cầu thủ hết hạn < 2 năm |
| **Players Expiring This Summer** | Count |
| **Avg MV of Expiring Players** | Để biết "tầm cỡ" tài sản đang bị đe dọa |
| **% Squad Value at Risk** | Value at Risk / Total Squad Value |

### Visuals

**Visual 1 — Contract Expiry Heatmap (Matrix)**
- Hàng: Position (GK, Defender, Midfield, Attack)
- Cột: Năm hết hạn hợp đồng (2025, 2026, 2027, 2028+)
- Giá trị: SUM(market_value_in_eur)
- Màu: Đỏ đậm = giá trị cao + hết hạn sớm → cực kỳ nguy hiểm
- Insight: Vị trí nào và thời điểm nào nguy hiểm nhất?

**Visual 2 — Priority Action Table**
- Lọc: Cầu thủ hết hạn < 18 tháng
- Cột: Tên | Vị trí | Tuổi | Hết hạn | MV | Phút thi đấu mùa này | Recommendation
- Recommendation logic (calculated column):
  - MV > 5M + Minutes > 1000 → "PRIORITY RENEW"
  - MV > 5M + Minutes < 500 → "CONSIDER SELLING NOW"
  - MV < 2M + Minutes > 1500 → "RENEW OR LET GO"
  - MV < 2M + Minutes < 500 → "LET GO"

**Visual 3 — Contract Length Distribution (Bar)**
- Trục X: Thời gian còn lại (bins: <6M, 6–12M, 1–2Y, 2–3Y, 3Y+)
- Trục Y: Số cầu thủ + Tổng MV (dual axis)
- Insight: Profile rủi ro hợp đồng tổng thể của đội hình

**Visual 4 — Historical Bosman Losses (Table)**
- Cầu thủ đã rời CLB với `transfer_fee = 0` (free transfer) trong quá khứ
- Cột: Tên | Năm | MV tại thời điểm ra đi | Phí thu về (0) | Value Lost
- Insight: CLB đã "tặng" bao nhiêu triệu EUR vì không gia hạn kịp?

### Insight kỳ vọng
- CLB có 4 cầu thủ tổng MV = 45M EUR hết hạn trong 12 tháng → "Value at Risk" tương đương ngân sách mua cầu thủ 1 mùa
- Trong 5 năm qua, CLB đã để mất 3 cầu thủ miễn phí với tổng MV lúc ra đi là 18M EUR

### Actionable Insight
- **Q1 Action:** Khởi động đàm phán gia hạn với tất cả cầu thủ trong vùng đỏ ngay
- **Q2 Action:** Nếu đàm phán thất bại → đưa ra thị trường chuyển nhượng hè trước khi mất miễn phí
- **Policy:** Thiết lập quy tắc "gia hạn hợp đồng khi còn 2 năm" cho cầu thủ MV > 3M EUR

---

---

## TRANG 7 — League Benchmarking

### Business Question
> *"CLB của chúng ta đang đứng ở đâu so với các đối thủ cạnh tranh trong cùng giải đấu — về sức mạnh đội hình, đầu tư chuyển nhượng, hiệu suất thi đấu và phát triển tài năng trẻ?"*

### Tại sao quan trọng (Decision Impact)
Board of Directors và Investors cần tầm nhìn comparative. Không phải chỉ "CLB có tốt không" mà là "CLB tốt hơn hay kém hơn đối thủ, và khoảng cách là bao nhiêu?"

### Bảng dữ liệu sử dụng
| Bảng | Vai trò | Join key |
|------|---------|----------|
| `clubs` | Tổng quan CLB | — |
| `players` | Đội hình và MV | `current_club_id` |
| `club_games` | Kết quả thi đấu | `club_id` |
| `transfers` | Hoạt động chuyển nhượng | `from/to_club_id` |
| `competitions` | Filter theo giải | `domestic_competition_id` |
| `player_valuations` | MV theo thời gian | `player_id` |

### KPI Cards — Benchmarking Summary
| KPI | Ý nghĩa |
|-----|---------|
| **CLB Rank by Squad MV** | Xếp hạng sức mạnh tài chính trong giải |
| **CLB Win Rate vs League Avg** | Chênh lệch hiệu suất |
| **Net Spend vs League Avg** | Chiến lược đầu tư so với đối thủ |
| **U23 Players % vs League Avg** | Định hướng trẻ hóa so với thị trường |
| **Foreign % vs League Quota** | Tuân thủ quy định |

### Visuals

**Visual 1 — League Ranking Table (Matrix — Main Visual)**
- Hàng: Từng CLB trong giải được chọn
- Cột: Squad MV | Win Rate | Goals For | Goals Against | Net Transfer Spend | Avg Age | U23 Count
- Sort theo bất kỳ cột
- Highlight hàng CLB của mình
- Insight: One-stop-shop cho benchmarking toàn diện

**Visual 2 — Radar Chart — Club vs. League Average**
- Các trục: MV Rank | Win Rate | Attack (Goals) | Defense (Clean Sheets) | Youth % | Net Spend
- Đường 1: CLB được chọn
- Đường 2: League average
- Insight: Hình thù radar nói lên strategy của CLB (tấn công vs phòng thủ vs đầu tư tài chính)

**Visual 3 — Quadrant — Spend vs Performance (Scatter)**
- Trục X: Net transfer spend (mùa hiện tại)
- Trục Y: Win Rate
- Size: Squad MV
- Label: Tên CLB
- Quadrant:
  - Top-left: Win nhiều nhưng chi ít = **Efficient clubs** (mẫu hình)
  - Top-right: Win nhiều, chi nhiều = Big spenders
  - Bottom-right: Chi nhiều nhưng thắng ít = **Wasteful investment**
  - Bottom-left: Chi ít, thắng ít = Struggling clubs
- Insight: CLB nào đang over/under-performing so với đầu tư?

**Visual 4 — Youth Development Index (Bar Chart)**
- Trục X: CLB
- Trục Y: Số cầu thủ U21 + Tổng MV cầu thủ U21
- Sort: Từ cao xuống thấp
- Insight: CLB nào đang đầu tư vào lò đào tạo vs mua đắt?

**Visual 5 — Transfer Activity Heatmap**
- Hàng: CLB, Cột: Mùa giải
- Giá trị: Net transfer spend
- Màu: Đỏ = chi nhiều, xanh = thu nhiều
- Insight: Pattern đầu tư theo mùa của từng CLB

### Insight kỳ vọng
- 2–3 CLB nằm ở góc "Efficient" (chi ít nhưng thắng nhiều) → học hỏi chiến lược của họ
- CLB của mình nằm ở "Wasteful" quadrant → cần tái cơ cấu transfer strategy
- Các CLB top đầu giải về U23 MV thường là "selling clubs" tiềm năng cho big clubs

### Actionable Insight
- **Benchmark:** Đặt mục tiêu KPI cụ thể dựa trên CLB efficient nhất trong giải
- **Recruitment model:** Học theo CLB có tỷ lệ Win Rate / Spend tốt nhất
- **Investment pitch:** Dùng trang này để thuyết phục investor về tiềm năng phát triển

---

---

## Phụ lục A — DAX Measures tổng hợp quan trọng

```dax
-- ===== PLAYER MEASURES =====

-- Age
Player Age = DATEDIFF(players[date_of_birth], TODAY(), YEAR)

-- Goals per 90
Goals per 90 =
DIVIDE(SUM(appearances[goals]), SUM(appearances[minutes_played]), 0) * 90

-- Assists per 90
Assists per 90 =
DIVIDE(SUM(appearances[assists]), SUM(appearances[minutes_played]), 0) * 90

-- Goal Contributions per 90
Goal Contributions p90 = [Goals per 90] + [Assists per 90]

-- Cards per game
Yellow Cards per Game =
DIVIDE(SUM(appearances[yellow_cards]), COUNTROWS(appearances), 0)


-- ===== CLUB MEASURES =====

-- Squad market value (latest)
Squad MV =
SUMX(
    VALUES(players[player_id]),
    CALCULATE(
        LASTNONBLANK(player_valuations[market_value_in_eur], 1)
    )
)

-- Win Rate
Win Rate % =
DIVIDE(
    CALCULATE(COUNTROWS(club_games), club_games[is_win] = 1),
    COUNTROWS(club_games),
    0
) * 100

-- Home Win Rate
Home Win Rate % =
CALCULATE([Win Rate %], club_games[hosting] = "Home")

-- Away Win Rate
Away Win Rate % =
CALCULATE([Win Rate %], club_games[hosting] = "Away")


-- ===== TRANSFER MEASURES =====

-- Net Transfer Balance
Net Spend =
CALCULATE(SUM(transfers[transfer_fee]), transfers[to_club_id] = SELECTEDVALUE(clubs[club_id]))
- CALCULATE(SUM(transfers[transfer_fee]), transfers[from_club_id] = SELECTEDVALUE(clubs[club_id]))

-- Smart Buy Index (MV tại thời điểm mua so với fee)
Buy Premium % =
DIVIDE(
    SUM(transfers[transfer_fee]) - SUM(transfers[market_value_in_eur]),
    SUM(transfers[market_value_in_eur]),
    0
) * 100


-- ===== MARKET VALUE MEASURES =====

-- MV Change last 12 months
MV Growth YoY % =
VAR LatestDate = MAX(player_valuations[date])
VAR LatestMV = CALCULATE(SUM(player_valuations[market_value_in_eur]),
    player_valuations[date] = LatestDate)
VAR Date1YrAgo = DATEADD(LatestDate, -1, YEAR)
VAR MV1YrAgo = CALCULATE(SUM(player_valuations[market_value_in_eur]),
    player_valuations[date] <= Date1YrAgo,
    TOPN(1, player_valuations, player_valuations[date], DESC))
RETURN DIVIDE(LatestMV - MV1YrAgo, MV1YrAgo, 0) * 100

-- Value at Risk (Bosman expiry)
Value at Risk 12M =
CALCULATE(
    [Squad MV],
    FILTER(players,
        NOT(ISBLANK(players[contract_expiration_date]))
        && DATEDIFF(TODAY(), players[contract_expiration_date], MONTH) <= 12
    )
)
```

---

## Phụ lục B — Filters / Slicers toàn Dashboard

Đặt ở thanh sidebar hoặc dùng Power BI Sync Slicers:

| Slicer | Scope | Ghi chú |
|--------|-------|---------|
| **Mùa giải** (season) | Toàn dashboard | Key filter |
| **Giải đấu** (competition) | Toàn dashboard | Multi-select |
| **CLB** | Theo trang | Sync trang 1, 5, 6, 7 |
| **Quốc gia** | Trang 2, 4 | Filter cầu thủ |
| **Vị trí** | Trang 2 | Attack/Mid/Def/GK |
| **Tuổi** (range slider) | Trang 2, 6 | |
| **Confederation** | Trang 7 | UEFA, CONMEBOL,... |

---

## Phụ lục C — Thiết kế màu sắc & UX

| Yếu tố | Recommendation |
|--------|---------------|
| **Background** | Dark navy (#0D1B2A) — professional sports feel |
| **Accent 1** | Gold (#F4B942) — highlights, KPI cards |
| **Accent 2** | Teal (#4ECDC4) — positive metrics (win, gain) |
| **Danger** | Red (#E63946) — risk indicators (loss, expiry) |
| **Neutral** | Gray (#8B97A2) — secondary info |
| **Font** | Segoe UI — native Power BI |
| **KPI cards** | Luôn có arrow indicator (▲▼) + % change |
| **Mobile layout** | Thiết kế phiên bản mobile cho trang 1 và 6 |
| **Tooltips** | Mỗi visual nên có custom tooltip với thông tin phụ thêm |
| **Bookmarks** | Lưu bookmark "Mùa hiện tại" và "5 năm gần nhất" |

---

## Phụ lục D — Load & Performance Optimization

| Bảng | Kích thước | Gợi ý |
|------|-----------|-------|
| `game_lineups` | 3M rows | Import mode — filter theo season range, ẩn các cột không dùng |
| `appearances` | 1.86M rows | Import mode — tạo aggregation table theo player+season |
| `game_events` | 1.24M rows | Import mode — tạo summary table theo minute_bin + type |
| `player_valuations` | 616K rows | Import mode — chỉ giữ latest per player nếu không cần timeline |
| `transfers` | 157K rows | Import mode — giữ nguyên |

**Gợi ý aggregation table cho appearances:**
```dax
AppearancesSummary =
SUMMARIZE(
    appearances,
    appearances[player_id],
    appearances[competition_id],
    "Season", RELATED(games[season]),
    "Total Games", COUNTROWS(appearances),
    "Total Goals", SUM(appearances[goals]),
    "Total Assists", SUM(appearances[assists]),
    "Total Minutes", SUM(appearances[minutes_played]),
    "Yellow Cards", SUM(appearances[yellow_cards]),
    "Red Cards", SUM(appearances[red_cards])
)
```

---

*Tài liệu này phục vụ như Dashboard Specification cho team Power BI developer. Mỗi trang có thể phát triển thành một sprint riêng. Ưu tiên phát triển: Trang 1 → Trang 2 → Trang 3 → Trang 6 → Trang 4 → Trang 5 → Trang 7.*
