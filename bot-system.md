HỆ THỐNG BOT CHO LEADERBOARD
===

# 1. Tổng quan

Leaderboard là bảng xếp hạng cạnh tranh theo chu kỳ **7 ngày**.

Mỗi nhóm Leaderboard gồm **10 người tham gia**. Thứ hạng được xác định dựa trên tổng số Level đã hoàn thành trong thời gian diễn ra sự kiện.

- Hạng 1, 2 và 3 nhận phần thưởng.
- Hệ thống ưu tiên ghép người chơi thật với nhau.
- Khi không đủ 10 người chơi thật, Bot được sử dụng để bổ sung các vị trí còn thiếu.
- Sau khi nhóm đủ 10 người, danh sách người tham gia được khóa trong toàn bộ chu kỳ 7 ngày.

Bot đóng vai trò như một người tham gia trong bảng xếp hạng và có khả năng tự tăng số Level hoàn thành theo thời gian.

---

# 2. Mục tiêu thiết kế

## 2.1. Mục tiêu chính

Đảm bảo Leaderboard luôn có đủ đối thủ để tạo ra một môi trường cạnh tranh đang hoạt động, kể cả khi lượng người chơi thật chưa đủ.

Bot cần tạo ra progression hợp lý trong suốt 7 ngày để:

- Leaderboard không xuất hiện nhiều vị trí trống.
- Người chơi không cảm thấy game thiếu người tham gia.
- Thứ hạng có khả năng thay đổi trong quá trình diễn ra sự kiện.
- Người chơi có động lực tiếp tục hoàn thành Level để cải thiện vị trí.

Bot **không tồn tại để đảm bảo người chơi thắng hoặc thua**.

Nguyên tắc cốt lõi:

> **Bot tạo ra môi trường cạnh tranh, không tạo sẵn kết quả cạnh tranh.**

## 2.2. Mục tiêu phụ

Hệ thống cần đủ khả năng cấu hình để Game Designer có thể:

- Tạo nhiều nhóm Bot với mức progression khác nhau.
- Mô phỏng các tình huống Leaderboard khác nhau.
- Điều chỉnh mức độ cạnh tranh mà không phải sửa logic cốt lõi.
- Thay các giả định ban đầu bằng dữ liệu người chơi thật khi game có đủ dữ liệu.

## 2.3. Ngoài phạm vi MVP

MVP (_Minimum Viable Product_), tức phiên bản tối thiểu khả dụng, chưa triển khai các cơ chế điều khiển thích ứng trực tiếp theo người chơi như:

- Bot tăng tốc khi người chơi sắp vượt.
- Bot giảm tốc để người chơi bắt kịp.
- Bot cố định khoảng cách với người chơi.
- Bot được ép thắng hoặc thua.
- Hệ thống chủ động ép người chơi về một hạng cụ thể.
- Rubber Banding, cơ chế tự điều chỉnh Bot dựa trực tiếp vào khoảng cách với người chơi.

Các cơ chế trên có thể được nghiên cứu ở phiên bản sau nếu dữ liệu thực tế cho thấy cần thiết.

---

# 3. Thuật ngữ chính

| Thuật ngữ              | Định nghĩa                                                                              |
| :--------------------- | --------------------------------------------------------------------------------------- |
| `CompletedLevels`      | Tổng số Level đã hoàn thành trong Leaderboard hiện tại. Đây là chỉ số dùng để xếp hạng. |
| `ProfileType`          | Hồ sơ hành vi tổng quát của Bot.                                                        |
| `BaseProgressionRate`  | Tốc độ progression nền của Bot, tính bằng Level/giờ thời gian sự kiện.                  |
| `CurrentEffectiveRate` | Tốc độ progression thực tế Bot đang sử dụng trong một khoảng thời gian hiện tại.        |
| `SimulationTick`       | Bước thời gian dùng để mô phỏng progression.                                            |
| `VarianceInterval`     | Khoảng thời gian Bot giữ cùng một `CurrentEffectiveRate` trước khi lấy giá trị mới.     |
| `ProgressAccumulator`  | Phần progression thập phân chưa đủ tạo thành một Level hoàn chỉnh.                      |
| `RateAnchor`           | Mốc tốc độ trung tâm mà một hồ sơ Bot đại diện.                                         |
| `RateRangePct`         | Biên độ cho phép `BaseProgressionRate` lệch khỏi `RateAnchor`.                          |
| `ProgressionVariance`  | Biên độ thay đổi của `CurrentEffectiveRate` quanh `BaseProgressionRate`                 |

---

# 4. Luồng tổng thể của hệ thống

1. Người chơi tham gia Leaderboard.
2. Hệ thống mở quá trình ghép nhóm.
3. Ưu tiên tìm người chơi thật phù hợp.
4. Quá trình ghép nhóm kết thúc.
5. Đếm số người chơi thật đã ghép được.
6. Tính số vị trí Bot còn thiếu.
7. Chọn cơ cấu hồ sơ Bot tương ứng.
8. Khởi tạo Bot.
9. Đủ 10 người tham gia.
10. Khóa danh sách người tham gia.
11. Bắt đầu Leaderboard 7 ngày.
12. Bot tăng tiến trình theo thời gian.
13. Khi sự kiện kết thúc, hệ thống tính bù Bot tới đúng thời điểm kết thúc.
14. Chốt `CompletedLevels`.
15. Áp dụng luật xử lý nếu trùng điểm.
16. Khóa thứ hạng cuối.
17. Xác định Top 1, 2, 3 và phát thưởng.

---

# 5. Luật hình thành nhóm Leaderboard

## 5.1. Quy tắc chung

- Kích thước Leaderboard: $$LeaderboardSize = 10$$
- Số Bot cần tạo: $$BotSlots = 10 - RealPlayerCount$$
- Người chơi thật luôn được ưu tiên hơn Bot.

Ví dụ:

| Người chơi thật | Bot | Tổng |
| :-------------- | --- | ---- |
| 10              | 0   | 10   |
| 7               | 3   | 10   |
| 3               | 7   | 10   |
| 1               | 9   | 10   |

## 5.2. Thời gian chờ ghép nhóm

`MatchingWindow = TBD`

`MatchingWindow` là khoảng thời gian hệ thống chờ tìm thêm người chơi thật trước khi dùng Bot để bổ sung.

Tham số này cần được tinh chỉnh sau dựa trên:

- Lượng người chơi đồng thời.
- Thời gian chờ có thể chấp nhận được.
- Khả năng tìm đủ người chơi thật.

## 5.3. Khóa danh sách người tham gia

Sau khi nhóm đủ 10 người:

- Danh sách được khóa
- Không thêm người chơi mới.
- Không thay Bot.
- Không thay người tham gia trong suốt 7 ngày.

---

# 6. Cấu trúc dữ liệu Bot

| Field                    | Nhóm               | Ý nghĩa                                                                        |
| :----------------------- | ------------------ | ------------------------------------------------------------------------------ |
| `BotID`                  | Identity           | Mã định danh duy nhất của một Bot instance trong hệ thống.                     |
| `DisplayName`            | Identity           | Tên Bot hiển thị trên Leaderboard.                                             |
| `Avatar`                 | Identity           | Ảnh đại diện hiển thị.                                                         |
| `CompletedLevels`        | Progression State  | Tổng số Level Bot đã hoàn thành trong Leaderboard hiện tại.                    |
| `LastUpdateTime`         | Progression State  | Thời điểm gần nhất tiến trình Bot đã được tính đến.                            |
| `ProgressAccumulator`    | Progression State  | Phần progression thập phân chưa đủ chuyển thành một Level hoàn chỉnh.          |
| `LastProgressTime`       | Progression State  | Thời điểm gần nhất `CompletedLevels` tăng.                                     |
| `ProfileType`            | Bot Profile        | Hồ sơ hành vi mà Bot thuộc về.                                                 |
| `BaseProgressionRate`    | Bot Behavior State | Tốc độ progression nền, tính bằng Level/giờ và giữ cố định trong suốt sự kiện. |
| `CurrentEffectiveRate`   | Bot Behavior State | Tốc độ progression thực tế đang dùng trong `VarianceInterval` hiện tại.        |
| `VarianceTicksRemaining` | Bot Behavior State | Số bước mô phỏng còn lại trước khi phải lấy `CurrentEffectiveRate` mới.        |

---

# 7. Giả định progression của người chơi

Do chưa có data thực tế, hệ thống sử dụng một bộ giả định được __thiết kế tạm thời__ để phục vụ cân bằng và mô phỏng.

Các giá trị này không phải là __dữ liệu thực tế của người chơi__.

## 7.1. Người chơi điển hình

Giả định rằng mốc Typical (điển hình) của người chơi:

- 3 phiên chơi/ngày.
- Hoàn thành 4 Level/phiên.

Vậy 1 ngày chơi được: 

$$3 \times 4 = 12\text{ (Levels)}$$

7 ngày chơi được: 

$$12 \times 7 = 84\text{ (Levels)}$$

- Thời lượng sự kiện:

$$7 \times 24 = 168\text{ (Giờ)}$$

- Tốc độ:

$$\frac{84}{168} = \frac{1}{2}\text{ (Level/Giờ)}$$

## 7.2. Các mốc progression giả định

| Mốc       | Ý nghĩa                   | Hệ số nhân | Level / 7 ngày |
| :-------- | ------------------------- | ---------- | -------------- |
| Very Low  | Gần như không progression | 0.25       | 21             |
| Low       | Progression thấp          | 0.50       | 42             |
| Typical   | Người chơi điển hình      | 1.00       | 84             |
| High      | Progression cao           | 1.50       | 126            |
| Very High | Progression rất cao       | 2.00       | 168            |

> **Trạng thái:** Các giả định này phải được thay bằng dữ liệu thực khi Analytics đủ mẫu.

Khi có dữ liệu thật, cần tính percentile (_phân vị của tập người chơi_): `P10`, `P25`, `P50`, `P75`, `P90`.

---

# 8. Hồ sơ hành vi Bot

MVP hiện tại có 4 `ProfileType`.

Các `ProfileType` này được nạp vào các mốc tham chiếu ở _Mục 7_.

| Hồ sơ         | Mốc tham chiếu | Vai trò                                                          |
| :------------ | -------------- | ---------------------------------------------------------------- |
| `Inactive`    | Very Low       | Gần như không cạnh tranh, tạo phần đáy tự nhiên cho Leaderboard. |
| `Casual`      | Low            | Progression thấp, tương đối dễ vượt nếu người chơi có hoạt động. |
| `Regular`     | Typical        | Đại diện cho đối thủ có progression tiêu chuẩn.                  |
| `Competitive` | High           | Đối thủ mạnh, có khả năng cạnh tranh Top 3.                      |

> `Very High` chưa được tạo thành hồ sơ riêng trong MVP. Đây là mốc tham chiếu biên để hỗ trợ cân bằng.

Hồ sơ Bot chỉ tạo __xu hướng hành vi__, không quyết định cứng thứ hạng.

---

# 9. Cấu hình tốc độ progression

$$RateAnchor = \frac{CompletedLevelsPer7Days}{168}$$

| Hồ sơ         | $RateAnchor$ |
| :------------ | ------------ |
| `Inactive`    | 0.125        |
| `Casual`      | 0.250        |
| `Regular`     | 0.500        |
| `Competitive` | 0.750        |

- Đơn vị: `Level/hour`.

> $RateAnchor$ là mức progression trung tâm mà hồ sơ đại diện.

---

# 10. Cấu hình toàn cục

| Config                | Giá trị | Ý nghĩa                                                                   |
| :-------------------- | ------- | ------------------------------------------------------------------------- |
| `SimulationTick`      | 1 giờ   | Mỗi bước mô phỏng tương đương 1 giờ thời gian sự kiện.                    |
| `VarianceInterval`    | 4 giờ   | Bot giữ cùng `CurrentEffectiveRate` trong tối đa 4 bước mô phỏng (4 giờ). |
| `RateRangePct`        | ±25%    | Biên độ khác biệt `BaseProgressionRate`giữa các Bot cùng hồ sơ.           |
| `ProgressionVariance` | ±20%    | Biên độ thay đổi progression của cùng một Bot theo thời gian.             |

> Tất cả các giá trị trên là tham số cân bằng, chưa được chứng minh bằng dữ liệu thực tế.

---

# 11. Sinh $BaseProgressionRate$

$BaseProgressionRate$ tạo sự khác biệt giữa các Bot của cùng một hồ sơ.

$$BaseProgressionRate \in [RateAnchor \times (1 - RateRangePct), RateAnchor \times (1 + RateRangePct)]$$

Với: 

$$RateRangePct = 0.25$$

| Hồ sơ         | $RateAnchor$ | Min     | Max     |
| :------------ | ------------ | ------- | ------- |
| `Inactive`    | 0.125        | 0.09375 | 0.15625 |
| `Casual`      | 0.250        | 0.18750 | 0.31250 |
| `Regular`     | 0.500        | 0.37500 | 0.62500 |
| `Competitive` | 0.750        | 0.56250 | 0.93750 |

Khi Bot được tạo: 

$$BaseProgressionRate \sim Uniform(\min, \max)$$

> `Uniform` nghĩa là lấy ngẫu nhiên theo __phân bố đều__, tức các giá trị trong khoảng có cơ hội được chọn tương đương nhau.

$BaseProgressionRate$
- Chỉ được lấy ngẫu nhiên một lần khi Bot được tạo.
- Giữa cố định trong toàn bộ chu kỳ Leaderboard.

Khoảng progression cơ sở dự kiến trong 7 ngày, chưa áp dụng biến thiên progression theo thời gian:

| Hồ sơ         | Level tối thiểu / 7 ngày | Level tối đa / 7 ngày |
| :------------ | ------------------------ | --------------------- |
| `Inactive`    | 15.75                    | 26.25                 |
| `Casual`      | 31.50                    | 52.50                 |
| `Regular`     | 63.00                    | 105.00                |
| `Competitive` | 94.50                    | 157.50                |

---

# 12. Biến thiên progression theo thời gian

Nếu Bot luôn chạy đúng $BaseProgressionRate$, progression sẽ quá tuyến tính.

Do đó, $CurrentEffectiveRate$ được thay đổi theo từng khoảng 4 giờ.

## 12.1. Khoảng $EffectiveRate$

$$EffectiveRate = [BaseProgressionRate \times (1 - ProgressionVariance), BaseProgressionRate \times (1 + ProgressionVariance)]$$

Với: 

$$ProgressionVariance = 0.20$$

Vậy: 

$$CurrentEffectiveRate \sim Uniform(EffectiveRate)$$

Ví dụ: $BaseProgressionRate = 0.20$

$$EffectiveRate = [0.20 \times (1 - 0.2), 0.20 \times (1 + 0.2)]$$

$$CurrentEffectiveRate \sim Uniform(0.16, 0.24)$$

## 12.2. Thời gian giữ $CurrentEffectiveRate$

$$VarianceTicksRemaining = \frac{VarianceInterval}{SimulationTick}$$

Với cấu hình hiện tại là: 

$$VarianceTicksRemaining = \frac{4}{1} = 4$$

Nghĩa là, Bot được giữ $CurrentEffectiveRate$ trong 4 bước mô phỏng.

- Sau mỗi bước:

$$VarianceTicksRemaining = VarianceTicksRemaining - 1$$

- Khi $VarianceTicksRemaining = 0$, hệ thống:

	- Lấy một $CurrentEffectiveRate$ mới.
	- Đặt lại $VarianceTicksRemaining = 4$.

---

# 13. Cách cập nhật $CompletedLevels$

Trong mỗi lần mô phỏng:

- Bước 1: Chuẩn bị $CurrentEffectiveRate$

	Nếu:
	$$VarianceTicksRemaining = 0$$

	Thì:
	$$CurrentEffectiveRate \sim Uniform(EffectiveRate)$$

	Và:
	$$VarianceTicksRemaining = \frac{VarianceInterval}{SimulationTick}$$

- Bước 2: Tính progression của Tick

	$$ProgressionGain = CurrentEffectiveRate \times SimulationTick$$

	Với:
	$$SimulationTick = 1 \text{ (hour)}$$

	Thì:
	$$ProgressionGain = CurrentEffectiveRate$$

- Bước 3: Cập nhật $CompletedLevels$ 

	$$TotalProgress = ProgressAccumulator + ProgressGain$$

	$$AddedLevels = \lfloor TotalProgress \rfloor$$

	$$CompletedLevels = CompletedLevels + AddedLevels$$

	$$ProgressAccumulator = TotalProgress - AddedLevels$$

  	> Hàm `Floor` trả về phần nguyên lớn nhất nhỏ hơn hoặc bằng $TotalProgress$.

- Bước 4: Cập nhật $LastProgressTime$

	$$LastProgressTime = (AddedLevels > 0) \text{ ?} \text{ Update :} \text{ Keep} $$

  	> $LastProgressTime$ được ghi nhận tại thời điểm kết thúc bước mô phỏng làm $CompletedLevels$ tăng.

- Bước 5: Giảm thời gian còn lại của $CurrentEffectiveRate$

	$$VarianceTicksRemaining = VarianceTicksRemaining - 1$$

---

# 14. Tính bù progression khi người chơi offline

Bot không bắt buộc phải được server cập nhật liên tục mỗi giờ.

Khi cần cập nhật trạng thái Leaderboard:

- Bước 1: Tính thời gian đã trôi qua
	$$ElapsedTime = CurrentTime - LastUpdateTime$$

- Bước 2: Tính số bước đầy đủ

	$$ElapsedTicks = \left\lfloor \frac{ElapsedTime}{SimulationTick} \right\rfloor$$

- Bước 3: Hệ thống chạy progression cho mỗi $SimulationTick$ của $ElapsedTicks$ đã tính.

- Bước 4: Với mỗi $ElapsedTick$, hệ thống thực hiện logic progression được định nghĩa tại __Mục 13__

- Bước 5:

	$$LastUpdateTime = LastUpdateTime + ElapsedTicks \times SimulationTick$$

  	>  	Phần thời gian chưa đủ một $SimulationTick$ được giữ lại cho lần cập nhật sau.
	

---

# 15. Cơ cấu hồ sơ Bot

Số lượng Bot phụ thuộc số người chơi thật đã ghép được.

| BotSlots | `Inactive` | `Casual` | `Regular` | `Competitive` |
| :------- | ---------- | -------- | --------- | ------------- |
| 0        | 0          | 0        | 0         | 0             |
| 1        | 0          | 0        | 1         | 0             |
| 2        | 0          | 1        | 1         | 0             |
| 3        | 0          | 1        | 1         | 1             |
| 4        | 0          | 1        | 2         | 1             |
| 5        | 1          | 1        | 2         | 1             |
| 6        | 1          | 2        | 2         | 1             |
| 7        | 1          | 2        | 2         | 2             |
| 8        | 1          | 3        | 2         | 2             |
| 9        | 1          | 3        | 3         | 2             |

> Đây là cấu hình giả định để simulation, cần điều chỉnh khi có dữ liệu người chơi thực.

---

# 16. Khởi tạo Bot

Khi cần tạo Bot:

1. Xác định `ProfileType` từ bảng cơ cấu Bot.
2. Chọn một Identity chưa được sử dụng trong Leaderboard hiện tại.
3. Tạo `BotID`
4. Lấy ngẫu nhiên $BaseProgressionRate$ trong khoảng Min–Max của hồ sơ.
5. Tìm tập $EffectiveRate$.
6. Lấy ngẫu nhiên $CurrentEffectiveRate$ từ tập $EffectiveRate$.
7. Khởi tạo các tham số:

| Parameter                | Value                     | Ý nghĩa |
| :----------------------- | ------------------------- | ------- |
| `CompletedLevels`        | 0                         |         |
| `ProgressAccumulator`    | 0                         |         |
| `VarianceTicksRemaining` | 4                         |         |
| `LastUpdateTime`         | LeaderboardGroupStartTime |         |
| `LastProgressTime`       | null                      |         |

8. Dùng `ParticipantID` để sắp xếp vị trí của Bot trên Leaderboard theo thứ tự tăng dần.
9. Thêm Bot vào danh sách người tham gia.

---

# 17. Kho danh tính Bot

Mỗi danh tính Bot gồm:

| Field         | Ý nghĩa                       |
| :------------ | ----------------------------- |
| `IdentityID`  | ID của bộ danh tính hiển thị. |
| `DisplayName` | Tên hiển thị.                 |
| `AvatarID`    | ID ảnh đại diện               |

Quy tắc:

1. Không sử dụng cùng `IdentityID` cho hai Bot trong cùng một Leaderboard.
2. Không có hai Bot cùng `DisplayName` trong một Leaderboard.
3. Danh tính có thể được tái sử dụng ở các Leaderboard khác nhau.
4. Danh tính không gắn cố định với `ProfileType`.
5. Danh tính không ảnh hưởng đến progression Bot.

Cách sản xuất nội dung tên/avatar nằm ngoài phạm vi logic Bot.

---

# 18. Luật xếp hạng

- **Điều kiện 1:** $CompletedLevels$ cao hơn đứng trên.

- **Điều kiện 2:** Nếu $CompletedLevels$ bằng nhau thì xét $LastProgressTime$ nào sớm hơn thì đứng trên.

- **Điều kiện 3:** Nếu $CompletedLevels$ và $LastProgressTime$ bằng nhau thì xét $ParticipantID$ theo thứ tự tăng dần để đảm bảo kết quả xác định, không phụ thuộc random.

> Các field phục vụ xếp hạng (`CompletedLevels`, `LastProgressTime`, `ParticipantID`) phải áp dụng thống nhất cho cả người chơi thật và Bot.

---

# 19. Kết thúc sự kiện

1. Tính bù tất cả Bot đến đúng `EventEndTime`.
2. Không tính progression sau `EventEndTime`.
3. Cập nhật $CompletedLevels$.
4. Áp dụng luật xếp hạng.
5. Khóa thứ hạng cuối cùng.
6. Xác định Top 1, 2, 3.
7. Phát thưởng

Kết quả cuối cùng phải độc lập với thời điểm người chơi mở game lại.

Ví dụ: Sự kiện kết thúc lúc 20:00 nhưng người chơi mở game lúc 23:00 thì Bot chỉ được tính progression tới 20:00.

---

# 20. Dữ liệu Tracking

| Metric                | Group                   | Ý nghĩa                                  |
| :-------------------- | ----------------------- | ---------------------------------------- |
| CompletedLevels7D     | Real Player Progression | Tổng level player hoàn thành trong event |
| FinalRank             | Real Player Progression | Rank cuối cùng                           |
| Top3Result            | Real Player Progression | Có vào Top 3 hay không                   |
| ProgressionByDay      | Real Player Progression | Số Level tích lũy theo từng ngày         |
| ProfileType           | Bot Behavior Validation | Bot thuộc profile nào                    |
| BaseProgressionRate   | Bot Behavior Validation | Rate được random lúc spawn               |
| FinalCompletedLevels  | Bot Behavior Validation | Progress cuối event                      |
| FinalRank             | Bot Behavior Validation | Rank cuối                                |
| Top3Result            | Bot Behavior Validation | Bot có vào Top3 hay không                |
| RealPlayerCount       | Leaderboard Composition | Số người chơi thật trong leaderboard     |
| BotCount              | Leaderboard Composition | Số Bot trong leaderboard                 |
| BotProfileComposition | Leaderboard Composition | Số Bot từng profile                      |
| Top3Composition       | Leaderboard Composition | Top 3 gồm bao nhiêu Real/Bot             |

---
