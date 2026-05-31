TÀI LIỆU ĐẶC TẢ
===

- **Phiên bản:** `1.0`
- **Ngày soạn thảo:** 31/05/2026
- **Người tạo:** Minh-Chau L. Pham
- **Reviewer:**
- **Mục đích:** Hướng dẫn chi tiết cách thu thập, tính toán và sử dựng các chỉ số (Metrics) liên quan đến Core Gameplay (Match-3) và Light Ball Tracking cho 10 level đầu tiên.

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

- `attempt_id:` ID duy nhất của lần chơi này

- `attempt_number:` Số thứ tự lần thử (1 = lần đầu; 2, 3, ... = retry)

- `session_id:` ID phiên chơi (tùy chọn nhưng khuyến khích)

- `timestamp:` Thời gian hệ thống (tùy chọn)

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

### 3.1.6. Avg. Power-ups Created

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

  - `powerup_type` (Propeller, Rocket, TNT, Light Ball)

**Benchmark:**

- Tính trên tất cả attempt (win và fail)

- Nếu $APC$
