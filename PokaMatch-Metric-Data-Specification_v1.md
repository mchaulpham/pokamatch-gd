TÀI LIỆU ĐẶC TẢ
===

- **Phiên bản:** `1.0`
- **Ngày soạn thảo:** 31/05/2026
- **Người tạo:** Minh-Chau L. Pham
- **Reviewer:**
- **Mục đích:** Hướng dẫn chi tiết cách thu thập, tính toán và sử dụng các chỉ số (Metrics) liên quan đến Core Gameplay (Match-3) và Light Ball Tracking cho 10 level đầu tiên.

---

# 1. Giới thiệu

## 1.1. Mục đích của tài liệu

Tài liệu này được xây dựng nhằm:

- Xác định rõ ràng định nghĩa, công thức tính toán và cách thu thập dữ liệu cho tất cả các chỉ số (Metrics) quan trọng trong giai đoạn Audit & Re-design Core Gameplay (10 level đầu).

- Đảm bảo sự thống nhất giữa Designer, Developer và QA khi implement tracking và phân tích dữ liệu.

- Cung cấp cơ sở data-driven để đánh giá độ khó, độ thỏa mãn và mức độ frustrate của từng level, từ đó hỗ trợ tối ưu Level Progression và Core Loop cho Target Audience (Nữ 25-44+, Thinker archetype).

- Làm tài liệu tham khảo chính thức cho các giai đoạn sau (level 11-30, live-ops, A/B testing).

## 1.2. Phạm vi áp dụng

- **Phạm vi:** Chỉ áp dụng cho **10 level đầu tiên** của Core Gameplay (Swapping Match-3).

- **Không nằm trong phạm vi:** Meta Layer (Map), Live-ops events, Monetization metrics, Retention D1/D7/D30.

- **Phiên bản sau:** Tài liệu sẽ được cập nhật khi mở rộng audit sang level 11-30 hoặc thêm mechanic mới.

## 1.3. Đối tượng sử dụng:

- **Game Designer:** Phân tích data, đề xuất cải tiến level.

- **Developer:** Implement event logging và telemetry.

- **QA / Data Analyst:** Kiểm tra tính chính xác của data và tạo report.

- **Team Leader / Project Manager:** Review tiến độ audit và quyết định ưu tiên cải tiến.

## 1.4. Giả định & Yêu cầu kỹ thuật

- Game sử dụng Firebase (hoặc Google Analytics for Firebase) để thu thập event.

- Mỗi lần chơi level đều được ghi nhận với `attempt_number` (1 = lần đầu; 2, 3, ... = retry).

- Dev đã implement đầy đủ __7 event__ chính theo bảng __Tracking Specification__.

- Tất cả event đều chứa các tham số chung: `level_id`, `attempt_id`, `session_id`.

---

# 2. Tổng quan về Event Logging

## 2.1. Mục tiêu của hệ thống tracking

Hệ thống tracking được thiết kế tối giản chỉ với __7 vent chính__, nhằm:

- Thu thập đầy đủ _raw data_ cho tất cả metric trong Level Audit và Light Ball Tracking.

- Giảm thiểu số lượng event để tránh spam log và tối ưu hiệu năng game.

- Đảm bảo data dễ tổng hợp (aggregate) trong Firebase Analytics / Google Analytics.

## 2.2. Danh sách 7 Event chính

| Event Name | Mô tả | Thời điểm log chính | Số lượng parameters chính |
|:---|:---|:---|:---|
| `Level_Start` | Bắt đầu một lần chơi level | Ngay khi vào level | 3 |
| `Level_End` | Kết thúc level<br>(win hoặc fail) | Khi level kết thúc | 6 |
| `PreLevel_Booster_Selected` | Người chơi chọn Pre-Level Booster | Sau màn chọn booster, trước khi vào level | 3 |
| `PowerUp_Created` | Tạo Power-up bằng match 4+ | Khi Power-up xuất hiện | 4 |
| `PowerUp_Combo_Activated` | Match 2 Power-up với nhau | Khi combo xảy ra | 3 |
| `InGame_Booster_Activated` | Sử dụng In-Game Booster | Khi activate booster trong level | 3 |
| `LightBall_Created` | Light Ball xuất hiện trong level (tất cả nguồn) | Khi Light Ball spawn trên board | 4 |
| `LightBall_Activated` | Người chơi activate Light Ball | Khi tap hoặc swipe Light Ball | 6 |

## 2.3. Tham số chung (Common Parameters)

Mọi event đều phải chứa các tham số sau:

- `level_id`: Số level (1-10)

- `attempt_id`: ID duy nhất của lần chơi này

- `attempt_number`: Số thứ tự lần thử (1 = lần đầu; 2, 3, ... = retry)

- `session_id`: ID phiên chơi (tùy chọn nhưng khuyến khích)

- `timestamp`: Thời gian hệ thống (tùy chọn)

## 2.4. Cách triển khai (implement)

- Log event ngay tại thời điểm xảy ra (real-time).

- Không log event khi người chơi pause hoặc thoát game giữa chừng.

- Kiểm tra event trong Firebase DebugView trước khi push build production.

- Đặt tên event theo quy tắc `PascalCase` và parameter theo `camelCase`

- Thêm comment rõ ràng trong code để dễ bảo trì sau này.

---

# 3. Chi tiết từng Metric

Phần này liệt kê tất cả các chỉ số (Metrics) trong Level Audit và Light Ball Tracking.

## 3.1. Core Level Metrics

### 3.1.1. First Attempt Win Rate (%)

**Định nghĩa:** First Attempt Win Rate (%) là tỷ lệ phần trăm (%) người chơi win ngay ở lần chơi đầu tiên (attempt_number = 1) của một level, không tính các lần retry.

**Ý nghĩa:** Đây là một trong những metric __quan trọng nhất__ trong giai đoạn onboarding và 10 level đầu.

- Phản ánh trực tiếp mức độ __dễ tiếp cận__ và __thỏa mãn ngay lập tức__ của level.

- Người chơi nữ 25-44+ (Thinker Archetype) rất nhạy cảm với cảm giác "mình giỏi" ngay từ lần chơi đầu. Nếu chỉ số này thấp, họ dễ cảm thấy level khó và drop sớm.

- Giúp Designer xác định nhanh level nào cần giảm độ khó, tăng cơ hội tạo Power-up/Light Ball.

**Công thức tính:**

$$FAWR (\\%) = \left( \dfrac{FAW}{TFA} \right) \times 100$$

**Đặc tả ký hiệu:**

- $FAWR$: First Attempt Win Rate

- $FAW$: First Attempt Wins (tổng số người chơi win ngay ở attempt đầu tiên của level)

- $TFA$: Total First Attempts (tổng số người chơi bắt đầu level ở attempt đầu tiên)

**Cách thu thập dữ liệu:**

- **Event chính:** `Level_End`

- **Parameters:**

  - `level_id`

  - `win` (true/false)

  - `attempt_number` (= 1)

- **Benchmark:**

- **Ghi chú:**

  - Chỉ tính những người chơi bắt đầu level lần đầu tiên.

  - Nếu chỉ track Overall Win Rate mà bỏ qua chỉ số này thì sẽ dẫn đến sai lầm trong thiết kế level progression.

---

### 3.1.2. Avg. Attempts to Win

**Định nghĩa:** Avg. Attemptsto Win là trung bình số lần thử (lần đầu + tất cả các lần retry) cho đến khi người chơi win một level.

**Ý nghĩa:** Đây là metric cực kỳ quan trọng để do lường __mức độ frustrate__ (stress) của level.

- Phản ánh trực tiếp cảm giác mệt mỏi của người chơi khi phải chơi đi chơi lại nhiều lần.

- Người chơi nữ 25-44+ (Thinker Archetype) rất nhạy cảm với chỉ số này. Nếu trung bình phải retry quá nhiều lần, họ dễ cảm thấy level _"không công bằng"_ và drop game.

- Giúp Designer xác định chính xác level nào cần giảm độ khó, tăng cơ hội tạo Power-up/Light Ball.

**Công thức tính:**

$$AATW = \dfrac{\sum(A_w)}{N_w}$$

**Đặc tả ký hiệu:**

- $AATW$: Avg. Attempts to Win

- $A_w$: attempt_number của các lần người chơi win level

- $N_w$: Total Wins (tổng số lần người chơi win level, không phân biệt attempt)

**Cách thu thập dữ liệu:**

- **Event chính:** `Level_End`

- **Parameters:**

  - `level_id`

  - `win` (true)

  - `attempt_number`

- **Benchmark:**

- **Ghi chú:**

  - Chỉ tính những lần win level

  - Nếu bỏ qua $AATW$ thì sẽ không phát hiện được level gây stress nặng cho người chơi.

---

### 3.1.3. Avg. Moves Used

**Định nghĩa:** Avg. Moves Used là số move trung bình người chơi sử dụng để hoàn thành (hoặc fail) một level.

> Với lần fail, `moves_used` được tính bằng __tổng số move có sẵn của level__.

**Ý nghĩa:** Đây là metric quan trọng để đánh giá __độ khó thực tế__ về mặt gameplay của level.

- Nếu quá cao: level gây cảm giác mệt mỏi, người chơi dễ chán và drop.

- Nếu quá thấp: level quá dễ, người chơi không cảm thấy _"thách thức"_ và mất hứng thú.

- Với target audience nữ 25-44+ (Thinker archetype), chỉ số này ảnh hưởng trực tiếp đến cảm giác _"thư giãn"_ và _"thỏa mãn khi giải đố"_.

**Công thức tính:**

$$AMU = \dfrac{\sum(M)}{N}$$

**Đặc tả ký hiệu:**

- $AMU$: Avg. Moves Used

- $M$: `moves_used` của từng attempt

- $N$: Tổng số attempt (cả win và fail)

**Cách thu thập dữ liệu:**

- **Event chính:** `Level_End`

- **Parameters:**

  - `level_id`

  - `moves_used`

- **Benchmark:**

- **Ghi chú:**

  - Tính trên __tất cả attempt__ (bao gồm cả lần fail).

  - Nếu $AMU$ cao bất thường ở những level đầu: cần tăng cơ hội tạo power-up hoặc giảm obstacel/goal

---

### 3.1.4. Churn Rate (%)

**Định nghĩa:** Churn Rate (%) là tỷ lệ phần trăm (%) người chơi drop (fail) ngay tại level này, không hoàn thành level.

**Ý nghĩa:** Đây là metric quan trong nhất để phát hiện level gây drop mạnh.

- Phản ánh trực tiếp mức độ khó/frustrate của level.

- Nếu Churn Rate cao ở 10 level đầu, người chơi (đặc biệt là nữ 25-44+, Thinker archetype) sẽ dễ bỏ game ngay từ giai đoạn onboarding.

- Giúp Designer ưu tiên fix level nào trước để nâng D1 retention.

**Công thức tính:**

$$CR(\\%) = \left( \dfrac{F}{T} \right) \times 100$$

**Đặc tả ký hiệu:**

- $CR$: Churn Rate

- $F$: Number of Fails (Số lần fail tại level)

- $T$: Total Attempts (tổng số lần attempt level, win và fail)

**Cách thu thập dữ liệu:**

- **Event chính:** `Level_End`

- **Parameters:**

  - `level_id`

  - `win` (false = fail)

  **Benchmark**

  **Ghi chú:**

  - Churn Rate cao ở level 1-3 là dấu hiệu FTUE (First-Time User Experience) kém.

  - Nên kết hợp xem với First Attempt Win Rate và Avg. Attempts to Win để chẩn đoán nguyên nhân.

---

### 3.1.5. Avg. Session Length (giây)

**Định nghĩa:** Avg. Session Length (giây) là thời gian trung bình người chơi dành cho một level (tính từ lúc vào level đến khi win hoặc fail).

**Ý nghĩa:** Đây là metric quan trọng để đo lường độ hấp dẫn và mức độ thư giãn của level.

- Người chơi nữ 25-44+ (Thinker archetype) thích session ngắn, dễ chơi (thường 60–120 giây mỗi level).
  
- Nếu quá dài: người chơi cảm thấy mệt mỏi, dễ drop.
  
- Nếu quá ngắn: level thiếu chiều sâu và không tạo được cảm giác thành tựu.
  
- Giúp Designer cân bằng giữa độ khó và trải nghiệm thư giãn.

**Công thức tính:**

$$ASL = \dfrac{\sum(S)}{N}$$

**Đặc tả ký hiệu:**

- $ASL$: Avg. Session Length (giây)

- $S$: `time_spent_seconds` của từng attempt

- $N$: Tổng số attempt (win và fail)

**Cách thu thập dữ liệu:**

- **Event chính:** `Level_End`

- **Parameters:**

  - `level_id`

  - `time_spent_seconds`

**Benchmark:**

**Ghi chú**

- Tính tên __tất cả attempt__ (win và fail).

----

## 3.2. Power-up & Booster Metrics

### 3.2.1. Avg. Power-ups Created

**Định nghĩa:** Avg. Power-ups Created là số Power-up trung bình được tạo ra bằng cách match 4+ items trong một level.

**Ý nghĩa:** Đây là metric quan trọng để đo lường tần suất người chơi được tạo Power-up (Rocket, TNT, Propeller, Light Ball).

- Power-up là nguồn chính tạo cảm giác “thỏa mãn” và “mạnh mẽ”.

- Với target audience nữ 25-44+ (Thinker archetype), chỉ số này cao giúp người chơi cảm thấy mình đang “giải đố thông minh” và tiến bộ rõ rệt.

- Nếu Avg. Power-ups Created thấp: level thiếu chiều sâu, người chơi dễ chán và drop sớm.

**Công thức:**

$$APC = \dfrac{\sum(P)}{N}$$

**Đặc tả ký hiệu:**

- $APC$: Avg. Power-ups Created

- $P$: Số Power-up được tạo trong từng attempt

- $N$: Tổng số attempt (win và fail)

**Cách thu thập dữ liệu:**

- **Event chính:** `PowerUp_Created`

- **Parameters:**

  - `level_id`

  - `powerup_type` (Propeller, Rocket, TNT, LightBall)

**Benchmark:**

- Tính trên tất cả attempt (win và fail)

---

### 3.2.2. Avg. Power-up Combos

**Định nghĩa:** Avg. Power-up Combos là số lần trung bình người chơi match 2 Power-up (Propeller, Rocket, TNT, Light Ball) với nhau trong một level.

**Ý nghĩa:** Đây là metric quan trọng để đo lường __cảm giác "wow"__ và __thỏa mãn cao nhất__ trong core gameplay.

- Power-up Combo tạo hiệu ứng nổ lớn, cascade đẹp mắt và giúp người chơi cảm thấy _"mình thông minh"_.

- Với target audience nữ 25-44+ (Thinker archetype), chỉ số này cao sẽ tăng đáng kể cảm giác Escapism & Mastery.

**Công thức tính:**

$$APCb = \dfrac{\sum(C)}{N}$$

**Đặc tả ký hiệu:**

- $APC$: Avg. Power-up Combos

- $C$: Số lần combo Power-up trong từng attempt

- $N$: Tổng số attempt (win và fail)

**Cách thu thập dữ liệu:**

- **Event chính:** `PowerUp_Combo_Activated`

- **Parameters:**

  - `level_id`
 
  - `combo_type` (ví dụ: Light Ball+Rocket, TNT+Propeller, ...)
 
**Benchmark:**

**Ghi chú:**

- Tính trên tất cả **attempt** (win và fail)

---

### 3.2.3. Pre-Level Boosters Usage Rate (%)

**Định nghĩa:** Pre-Level Boosters Usage Rate (%) là tỷ lệ phần trăm (%) người chơi chọn Pre-Level Boosters (Rocket, TNT, Light Ball) trước khi bắt đầu một level.

**Ý nghĩa:**

**Công thức tính:**

$$PBR(\\%) = \left( \dfrac{N_{pre}}{T} \right) \times 100$$

**Đặc tả ký hiệu:**

- $PBR$: Pre-Level Boosters Usage Rate

- $N_{pre}$: Số lần chọn Pre-Level Boosters

- $T$: Tổng số lần bắt đầu level

**Cách thu thập dữ liệu:**

- **Event chính:** `PreLevel_Booster_Selected`

- **Parameters:**

  - `level_id`
 
  - `booster_type` (Propeller / Rocket / TNT / LightBall / None)
 
**Benchmark:**

**Ghi chú:**

- Tính trên __tất cả lần bắt đầu level__.

---

### 3.2.4. In-Game Boosters Used

**Định nghĩa:** In-Game Boosters Used là số lần trung bình người chơi sử dụng In-Game Boosters (Royal Hammer, Arrow, Cannon, Jester Hat) trong một level.

**Ý nghĩa:** Đây là metric quan trọng để đo lường __tần suất người chơi sử dụng boosters trong lúc chơi__.

  - In-game Boosters giúp người chơi thoát khỏi tình huống khó, tăng cảm giác kiểm soát và thỏa mãn.

  - Nếu chỉ số này thấp: In-Game Boosters chưa đủ hấp dẫn hoặc khó sử dụng.

**Công thức tính:**

$$IGBU = \dfrac{\sum(B)}{N}$$

**Đặc tả ký hiệu:**

- $IGBU$: In-Game Boosters Used

- $B$: Số lần active In-Game Boosters trong từng attempt

- $N$: Tổng số attempt (win và fail)

**Cách thu thập dữ liệu:**

- **Event chính:** `InGame_Booster_Activated`

- **Parameters:**

  - `level_id`
 
  - `booster_type` (RoyalHammer / Arrow / Cannon / JesterHat)
 
**Benchmark:**

**Ghi chú:**

- Tính trên __tất cả attempt__ (win và fail)

---

## 3.3. Light Ball Metrics

### 3.3.1. Avg. Light Balls Created (Total)

**Định nghĩa:** Avg. Light Balls Created (Total) là tổng số Light Ball trung bình xuất hiện trong một level attempt (tính từ tất cả các nguồn; Player Match-5, Pre-Level Booster và Board Auto-Spawn)

**Ý nghĩa**: Đây là metric cốt lõi để đo lường __tần suất xuất hiện của Power-up mạnh nhất__ trong game.

- Light Ball là yếu tố quyết định cảm giác _"thỏa mãn cao nhất"_ và khả năng hoàn thành goal nhanh.

**Công thức tính:**

$$ALBC = \dfrac{\sum(L)}{N}$$

**Đặc tả ký hiệu:**

- $$ALBC$$: Avg. Light Balls Created (Total)

- $L$: Số Light Ball xuất hiện trong từng attempt

- $N$: Tổng số attempt (win và fail)

**Cách thu thập dữ liệu:**

- **Event chính:** `LightBall_Created`

- **Parameters:**

  - `level_id`
 
  - `source` (PlayerMatch5 / PreLevel / BoardAutoSpawn)
 
**Benchmark**

**Ghi chú:**

- Tính trên tất cả attempt (win và fail).

---

### 3.3.2. Pre-Level Light Ball Usage Rate (%)

**Định nghĩa:** Pre-Level Light Ball Usage Rate (%) là tỷ lệ phần trăm (%) người chơi chọn Light Ball từ Pre-Level Boosters trước khi bắt đầu một level.

**Ý nghĩa:** Đây là metric quan trọng để đo lường __sức hấp dẫn__ và __mức độ sử dụng Light Ball Pre-Level Booster__.

- Light Ball là Power-up mạnh nhất, chọn nó trước level giúp người chơi có lợi thế lớn ngay từ đầu.

**Công thức tính:**

$$PLBUR(\\%) = \left(  \dfrac{N_{preLB}}{T} \right) \times 100$$

**Đặc tả ký hiệu:**

- **Event chính:** `PreLevel_Booster_Selected`

- **Parameters:**

  - `level_id`
 
  - `booster_type` = "LightBall"
 
**Benchmark:**

**Ghi chú:**

- Tính trên __tất cả lần bắt đầu level__.

- Vì Pre-Level Light Ball chỉ spawn tối đa 1 quả, chỉ số này phản ánh trực tiếp quyết định chọn booster mạnh nhất của người chơi.

----

### 3.3.3. Avg. Player-Created Light Balls (Match-5)

**Định nghĩa:** Avg. Player-Created Light Balls (Match-5) là số Light Ball trung bình được tạo ra bởi người chơi thông qua match 5 tiles cùng màu (không tính Light Ball từ Pre-Level Booster hoặc Board Auto-Spawn).

**Ý nghĩa:** Đây là metric quan trọng để đo lường __kỹ năng__ và __sự chủ động__ của người chơi trong việc tạo Power-up mạnh nhất.

**Công thức tính:**

$$APCLB = \dfrac{\sum(L_p)}{N}$$

**Đặc tả ký hiệu:**

- $APCLB$: Avg. Player-Created Light Balls (Match-5)

- $L_p$: Số Light Ball được tạo bởi Player Match-5 trong từng attempt

- $N$: Tổng số attempt (win và fail)

**Cách thu thập dữ liệu:**

- **Event chính:** `LightBall_Created`

- **Parameters:**

  - `level_id`
 
  - `source` = "PlayerMatch5"
 
**Benchmark:**

**Ghi chú:**

- Chỉ tính Light Ball do người chơi tự match-5 (không tính Pre-Level và Board Auto-Spawn).
 
---

### 3.3.4. Avg. Board Auto-Spawn Light Balls

**Định nghĩa:** Avg. Board Auto-Spawn Light Balls là số Light Ball trung bình được board tự spawn khi tiles rơi xuống trong một level attempt.

**Ý nghĩa:** Đây là metric quan trọng để đo lường __yếu tố ngẫu nhiên__ và __may mắn__ từ thiết kế level.

**Công thức tính:**

$$ABASLB = \dfrac{\sum(L_b)}{N}$$

**Cách thu thập dữ liêu:**

- **event chính:** `LightBall_Created`

- **Parameters:**

  - `level_id`
 
  - `source` = "BoardAutoSpawn"
 
**Benchmark**

**Ghi chú:**

- Tính trên tất cả attempt (win và fail)

---

### 3.3.5. Avg. Light Ball Effectiveness (%)

**Định nghĩa:** Avg. Light Ball Effectiveness (%) là tỷ lệ phần trăm (%) các lần active Light Ball đạt hiệu quả (clear >= 25 tiles hoặc hoàn thành >= 50% goal còn lại).

**Ý nghĩa:** Đây là metric quan trọng nhất để đo lường __chất lượng sử dụng__ của Light Ball (Power-up mạnh nhất trong game).

- Không chỉ đém số lượng Light Ball xuất hiện, mà còn đánh giá người chơi có tận dụng được nó hiệu quả hay không.

- Nếu chỉ số thấp: Người chơi đang active Light Ball sai thời điểm hoặc board không hỗ trợ tốt.

**Công thức tính:**

$$LBE (\\%) = \left( \dfrac{E}{A} \right) \times 100$$

**Đặc tả ký hiệu:**

- $LBE$: Light Ball Effectiveness

- $E$: Số lần active Light Ball đạt hiệu quả

- $A$: Tổng số lần activate Light Ball

**Cách thu thập dữ liệu:**

- **Event chính:** `LightBall_Activated`

- **Parameters:**

  - `level_id`
 
  - `tiles_cleared`
 
  - `goal_progress_before`
 
  - `goal_progress_after`
 
**Benchmark:**

**Ghi chú:**

- Hiệu quả được định nghĩa là clear >= 25 tiles __hoặc__ hoàn thành >= 50% goal còn lại.

- Nếu $LBE$ thấp: cần cải thiện hoặc điều chỉnh thời điểm và vị trí spawn để người chơi dễ sử dụng hơn.

---

### 3.3.6. Avg. Light Ball Combinations

**Định nghĩa:** Avg. Light Ball Combinations là số lần trung bình người chơi kết hợp Light Ball với các Power-up khác (Propeller, Rocket, TNT, Light Ball) trong một level.

**Ý nghĩa:** Đây là metric đo lường __peak moment__ (khoảng khắc thỏa mãn nhất) trong core gameplay.

- Light Ball kết hợp với Power-up khác tạo hiệu ứng nổ cực lớn, cascade mạnh mẽ và clear hàng loạt tiles.

**Công thức tính:**

$$ALBC_{mb} = \dfrac{\sum(C_{mb})}{N}$$

**Đặc tả ký hiệu:**

- $ALBC_{mb}$: Avg. Light Ball Combinations

- $C_{mb}$: Số lần kết hợp Light Ball với Power-up khác trong từng attempt

- $N$: Tổng số attempt (win và fail).

**Cách thu thập dữ liệu:**

- **Event chính:** `LightBall_Activated`

- **Parameters:**

  - `level_id`
 
  - `combined_with` (Propeller / Rocket / TNT / LightBall)

 **Benchmark:**

 **Ghi chú:**

- Chỉ tính những lần Light Ball được kết hợp với các Power-up kacs hoặc với chính nó (không tính activate đơn thuần).

- Nếu $ALBC_{mb}$ thấp: cần tăng cơ hội match Light Ball gần Power-up hoặc điều chỉnh vị trí spawn.

---

## 3.4. Bảng Tổng hợp

| STT | Metric | Viết tắt | Định nghĩa | Event | Parameter | Benchmark |
|:---|:---|:---|:---|:---|:---|:---|
| 1 | First Attempt Win Rate (%) | FAWR | % win ngay ở attempt đầu tiên của level | `Level_End` | `level_id`,<br>`attempt_number` (=1) |  |
| 2 | Avg. Attempts to Win | AATW | Trung bình số lần thử đến khi win level | `Level_End` | `level_id`,<br>`win`,<br>`attempt_number` |  |
| 3 | Avg. Moves Used | AMU | Số move trung bình sử dụng trong attempt | `Level_End` | `level_id`,<br>`moves_used` |  |
| 4 | Churn Rate (%) | CR | % fail/drop ngay tại level | `Level_End` | `level_id`,<br>`win` (false) |  |
| 5 | Avg. Session Length (giây) | ASL | Thời gian trung bình chơi một level | `Level_End` | `level_id`,<br>`time_spent_seconds` |  |
| 6 | Avg. Power-ups Created | APC | Số Power-up trung bình được tạo bằng match 4+ | `PowerUp_Created` | `level_id`,<br>`powerup_type` |  |
| 7 | Avg. Power-up Combos | APCb | Số lần trung bình match 2 Power-up | `PowerUp_Combo_Activated` | `level_id`,<br>`combo_type` |  |
| 8 | Pre-Level Boosters Usage Rate (%) | PBR | % người chơi chọn Pre-Level Boosters | `PreLevel_Booster_Selected` | `level_id`,<br>`booster_type` |  |
| 9 | In-Game Boosters Used | IGBU | Số lần trung bình sử dụng In-Game Boosters | `InGame_Booster_Activated` | `level_id`,<br>`booster_type` |  |
| 10 | Avg. Light Balls Created (Total) | ALBC | Tổng số Light Ball trung bình xuất hiện trong level | `LightBall_Created` | `level_id`,<br>`source` |  |
| 11 | Pre-Level Light Ball Usage Rate (%) | PLBUR | % người chơi chọn Light Ball từ Pre-Level | `PreLevel_Booster_Selected` | `level_id`,<br>`booster_type` = "LightBall" |  |
| 12 | Avg. Player-Created Light Balls (Match-5) | APCLB | % người chơi chọn Light Ball từ Pre-Level | `PreLevel_Booster_Selected` | `level_id`,<br>`booster_type` = "LightBall" |  |
| 13 | Avg. Board Auto-Spawn Light Balls | ABASLB | Số Light Ball trung bình do Board Auto-Spawn | `LightBall_Created` | `level_id`,<br>`source` = "BoardAutoSpawn" |  |
| 14 | Avg. Light Ball Effectiveness (%) | LBE | % lần activate Light Ball đạt hiệu quả |  |
| 15 | Avg. Light Ball Combinations | ALBCmb | Số lần trung bình kết hợp Light Ball với Power-up khác | `LightBall_Activated` | `level_id`,<br>`combined_with` |  |




