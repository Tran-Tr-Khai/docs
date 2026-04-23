# Transfermarkt Analytics – Mô tả Bộ Dữ Liệu

Bộ dữ liệu được thu thập từ [Transfermarkt](https://www.transfermarkt.co.uk), bao gồm thông tin về cầu thủ, câu lạc bộ, trận đấu và chuyển nhượng trong bóng đá. Tổng cộng **12 bảng**, hơn **7.2 triệu bản ghi**.

---

## Sơ đồ quan hệ (Entity Relationship)

```
countries ──< competitions ──< games >── club_games
                                  │
                                  ├──< game_events
                                  └──< game_lineups
clubs ──< players ──< appearances ──> games
              │
              ├──< player_valuations
              └──< transfers
national_teams ──── countries
```

---

## Mô tả từng bảng

### 1. `countries.csv` — 118 bản ghi
Danh sách các quốc gia.

| Cột | Mô tả |
|-----|-------|
| `country_id` | ID quốc gia (PK) |
| `country_name` | Tên quốc gia |
| `country_code` | Mã code giải VĐQG |
| `confederation` | Liên đoàn (UEFA, CONMEBOL, …) |
| `total_clubs` | Tổng số CLB |
| `total_players` | Tổng số cầu thủ |
| `average_age` | Tuổi trung bình cầu thủ |
| `url` | Đường dẫn Transfermarkt |

---

### 2. `competitions.csv` — 67 bản ghi
Thông tin các giải đấu (VĐQG, cúp châu lục, giải ĐTQG).

| Cột | Mô tả |
|-----|-------|
| `competition_id` | ID giải đấu (PK) |
| `competition_code` | Slug/code giải đấu |
| `name` | Tên giải đấu |
| `sub_type` | Phân loại (`first_tier`, `cup`, `afc_asian_cup`, …) |
| `type` | Loại (`domestic_league`, `national_team_competition`, …) |
| `country_id` | FK → `countries` |
| `country_name` | Tên quốc gia tổ chức |
| `domestic_league_code` | Mã giải VĐQG |
| `confederation` | Liên đoàn |
| `total_clubs` | Số CLB tham dự |
| `url` | Đường dẫn Transfermarkt |

---

### 3. `clubs.csv` — 796 bản ghi
Thông tin chi tiết các câu lạc bộ.

| Cột | Mô tả |
|-----|-------|
| `club_id` | ID câu lạc bộ (PK) |
| `club_code` | Slug CLB |
| `name` | Tên CLB |
| `domestic_competition_id` | FK → `competitions` |
| `total_market_value` | Tổng giá trị đội hình |
| `squad_size` | Số lượng cầu thủ |
| `average_age` | Tuổi trung bình đội hình |
| `foreigners_number` | Số cầu thủ ngoại |
| `foreigners_percentage` | Tỷ lệ cầu thủ ngoại (%) |
| `national_team_players` | Số cầu thủ được triệu tập ĐTQG |
| `stadium_name` | Tên sân vận động |
| `stadium_seats` | Sức chứa sân |
| `net_transfer_record` | Số dư ròng chuyển nhượng |
| `coach_name` | Tên HLV |
| `last_season` | Mùa giải gần nhất có dữ liệu |

---

### 4. `national_teams.csv` — 118 bản ghi
Thông tin các đội tuyển quốc gia.

| Cột | Mô tả |
|-----|-------|
| `national_team_id` | ID đội tuyển (PK) |
| `name` | Tên đội tuyển |
| `country_id` | FK → `countries` |
| `confederation` | Liên đoàn (UEFA, AFC, …) |
| `squad_size` | Số cầu thủ trong đội |
| `average_age` | Tuổi trung bình |
| `total_market_value` | Tổng giá trị đội hình |
| `coach_name` | Tên HLV |
| `fifa_ranking` | Xếp hạng FIFA |
| `last_season` | Mùa giải gần nhất |

---

### 5. `players.csv` — 47,702 bản ghi
Hồ sơ cầu thủ (thông tin cá nhân & nghề nghiệp).

| Cột | Mô tả |
|-----|-------|
| `player_id` | ID cầu thủ (PK) |
| `first_name` / `last_name` / `name` | Họ, tên, tên đầy đủ |
| `date_of_birth` | Ngày sinh |
| `country_of_birth` | Quốc gia sinh |
| `country_of_citizenship` | Quốc tịch |
| `position` | Vị trí chính (`Attack`, `Midfield`, `Defender`, `Goalkeeper`) |
| `sub_position` | Vị trí cụ thể (`Centre-Forward`, `Centre-Back`, …) |
| `foot` | Chân thuận |
| `height_in_cm` | Chiều cao (cm) |
| `current_club_id` | FK → `clubs` |
| `current_club_name` | Tên CLB hiện tại |
| `contract_expiration_date` | Ngày hết hạn hợp đồng |
| `agent_name` | Tên người đại diện |
| `international_caps` | Số lần khoác áo ĐTQG |
| `international_goals` | Số bàn thắng ĐTQG |
| `market_value_in_eur` | Giá trị thị trường hiện tại (EUR) |
| `highest_market_value_in_eur` | Giá trị thị trường cao nhất từng đạt |

---

### 6. `games.csv` — 86,983 bản ghi
Kết quả và thông tin chi tiết từng trận đấu.

| Cột | Mô tả |
|-----|-------|
| `game_id` | ID trận đấu (PK) |
| `competition_id` | FK → `competitions` |
| `season` | Mùa giải |
| `round` | Vòng đấu |
| `date` | Ngày thi đấu |
| `home_club_id` / `away_club_id` | FK → `clubs` |
| `home_club_goals` / `away_club_goals` | Số bàn thắng |
| `home_club_formation` / `away_club_formation` | Sơ đồ chiến thuật |
| `home_club_manager_name` / `away_club_manager_name` | HLV hai đội |
| `stadium` | Sân thi đấu |
| `attendance` | Khán giả |
| `referee` | Trọng tài |
| `competition_type` | Loại giải đấu |

---

### 7. `club_games.csv` — 173,966 bản ghi
Thống kê từng trận đấu theo góc nhìn của mỗi đội (mỗi trận tạo ra 2 bản ghi).

| Cột | Mô tả |
|-----|-------|
| `game_id` | FK → `games` |
| `club_id` | FK → `clubs` |
| `own_goals` | Bàn thắng của đội này |
| `own_position` | Xếp hạng đội này |
| `own_manager_name` | HLV đội này |
| `opponent_id` | FK → `clubs` (đối thủ) |
| `opponent_goals` | Bàn thắng đối thủ |
| `hosting` | Vai trò (`Home` / `Away`) |
| `is_win` | Kết quả thắng/thua (`1` / `0`) |

---

### 8. `game_events.csv` — 1,242,945 bản ghi
Các sự kiện trong trận đấu (bàn thắng, thẻ phạt, thay người).

| Cột | Mô tả |
|-----|-------|
| `game_event_id` | ID sự kiện (PK) |
| `game_id` | FK → `games` |
| `date` | Ngày diễn ra sự kiện |
| `minute` | Phút xảy ra |
| `type` | Loại sự kiện (`Goals`, `Cards`, `Substitutions`) |
| `club_id` | FK → `clubs` |
| `player_id` | Cầu thủ liên quan (FK → `players`) |
| `description` | Mô tả chi tiết |
| `player_in_id` | Cầu thủ vào sân (khi thay người) |
| `player_assist_id` | Cầu thủ kiến tạo (khi ghi bàn) |

---

### 9. `game_lineups.csv` — 3,049,833 bản ghi
Đội hình ra sân và cầu thủ dự bị cho từng trận.

| Cột | Mô tả |
|-----|-------|
| `game_lineups_id` | ID bản ghi (PK) |
| `game_id` | FK → `games` |
| `player_id` | FK → `players` |
| `club_id` | FK → `clubs` |
| `date` | Ngày trận đấu |
| `type` | Vai trò (`starting_lineup` / `substitutes`) |
| `position` | Vị trí thi đấu |
| `number` | Số áo |
| `team_captain` | Đội trưởng (`1` / `0`) |

---

### 10. `appearances.csv` — 1,862,208 bản ghi
Thống kê hiệu suất thi đấu của từng cầu thủ trong từng trận.

| Cột | Mô tả |
|-----|-------|
| `appearance_id` | ID lần thi đấu (PK, dạng `game_id_player_id`) |
| `game_id` | FK → `games` |
| `player_id` | FK → `players` |
| `player_club_id` | CLB của cầu thủ tại thời điểm thi đấu |
| `player_current_club_id` | CLB hiện tại của cầu thủ |
| `competition_id` | FK → `competitions` |
| `date` | Ngày thi đấu |
| `yellow_cards` | Số thẻ vàng |
| `red_cards` | Số thẻ đỏ |
| `goals` | Số bàn thắng |
| `assists` | Số kiến tạo |
| `minutes_played` | Số phút thi đấu |

---

### 11. `player_valuations.csv` — 616,377 bản ghi
Lịch sử định giá thị trường của cầu thủ theo thời gian.

| Cột | Mô tả |
|-----|-------|
| `player_id` | FK → `players` |
| `date` | Ngày định giá |
| `market_value_in_eur` | Giá trị thị trường tại thời điểm đó (EUR) |
| `current_club_name` | Tên CLB tại thời điểm định giá |
| `current_club_id` | FK → `clubs` |
| `player_club_domestic_competition_id` | FK → `competitions` |

---

### 12. `transfers.csv` — 157,186 bản ghi
Lịch sử chuyển nhượng cầu thủ.

| Cột | Mô tả |
|-----|-------|
| `player_id` | FK → `players` |
| `player_name` | Tên cầu thủ |
| `transfer_date` | Ngày chuyển nhượng |
| `transfer_season` | Mùa giải chuyển nhượng |
| `from_club_id` / `from_club_name` | CLB xuất phát |
| `to_club_id` / `to_club_name` | CLB đích đến |
| `transfer_fee` | Phí chuyển nhượng (EUR) |
| `market_value_in_eur` | Giá trị thị trường tại thời điểm chuyển nhượng |

---

## Tổng quan quy mô

| Bảng | Số bản ghi |
|------|------------|
| `game_lineups` | 3,049,833 |
| `appearances` | 1,862,208 |
| `game_events` | 1,242,945 |
| `player_valuations` | 616,377 |
| `transfers` | 157,186 |
| `club_games` | 173,966 |
| `games` | 86,983 |
| `players` | 47,702 |
| `clubs` | 796 |
| `countries` | 118 |
| `national_teams` | 118 |
| `competitions` | 67 |
| **Tổng** | **~7.2 triệu** |
