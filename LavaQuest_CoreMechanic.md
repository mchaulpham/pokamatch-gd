# 1. PHASE A

- Thời điểm: Ngay khi Cohort được tạo (Player bấm Start)

- Logic: Hệ thống random một giá trị $R$ trong $[0, 100]$ để chọn ra __Tier__ theo bảng config bên dưới để từ đó xác định được **tổng số Winner**.

| Tier | Cumulative Range | Winner | Weight |
|:---|:---|:---|:---|
| Champion | $[1, 5]$ | 1 | 0.05 |
| Top | $[6, 20]$ | $[2, 3]$ | 0.15 |
| Mid | $[21, 80]$ | $[4, 8]$ | 0.6 |
| Crowd | $[81, 100]$ | $[9, 15]$ | 0.2 |

> **Weight**: Là xác suất để hệ thống chọn một __Tier__ cụ thể ngay tại thời điểm khởi tạo Cohort. Tổng Weight của tất cả Tier phải luôn bằng 100%.

- Nếu:
  - $R \in [1, 5] \rightarrow \text{ Champion}$
  - $R \in [6, 20] \rightarrow \text{ Top}$
  - $R \in [21, 80] \rightarrow \text{ Mid}$
  - $R \in [81, 100] \rightarrow \text{ Crowd}$

- Dựa vào Tier đã chọn, tiếp tục random một số nguyên $W \in [Min_{Winners}, Max_{Winners}]$ của Tier đó.

> **W** là tổng số Winner còn lại sau 7 step.

- Tính tổng số bot cần loại là $T$:

$$
T = 100 - W
$$

- **Output**: Một số nguyên $T$ là tổng số bot cần loại của 7 step.

---

# 2. PHASE B

## Bước 1

- **Input**:

  - Tổng số Bot cần loại $T$
 
  - Bảng config để phân phối $T$ vào mỗi step bằng trọng số `Pacing Weight`
 
| Step | Pacing Weight |
|:---|:---|
| 1 | $W_1$ |
| 2 | $W_2$ |
| 3 | $W_3$ |
| 4 | $W_4$ |
| 5 | $W_5$ |
| 6 | $W_6$ |
| 7 | $W_7$ |

- **Ràng buộc**:

$$
\sum_{i = 1}^{7} W_i = 1
$$

- Tính giá trị phân phối cơ bản (Base) cho $step_i$:

$$
Base_i = T \times W_i
$$

### **Vấn đề**

$Basi_i$ sẽ có trường hợp là số thập phân, nên cần dùng phương pháp __phần dư lớn nhất (Largest Remainder)__.

1. Lấy phần nguyên của $Base_i$

$$
\lfloor Base_i \rfloor
$$

2. Tính phần dư của $Base_i$

$$
Remainder_i = Base_i - \lfloor Base_i \rfloor
$$

3. Sắp xếp và tìm kiếm các $step_i$ theo $Remainder_i$ giảm dần.

4. Công thêm 1 đơn vị vào $\lfloor Base_i \rfloor$ của các $step$ có $Remainder$ lớn nhất cho đến khi:

$$
\sum_{i = 1}^{7} Base_i = T
$$

> **Lưu ý**: Khi cộng thêm 1 đơn vị vào $\lfloor Base_i \rfloor$, nếu các $step_i$ có $Remainder$ bằng nhau, thì ưu tiên chọn $i$ nhỏ hơn.

### Output

Một mảng số nguyên $Base_i$ với $i = \{1, 2, 3, 4, 5, 6, 7\}$ sao cho:

$$
\sum_{i = 1}^{7} Base_i = T
$$

## Bước 2

Để phá vỡ pattern, mỗi $step_i$ được làm cho dao động lên hoặc xuống so với $Base$ của nó. Độ dao động này được gọi là $d$ với mỗi $d_i$ tương ứng với $step_i$.

$$
d_i = \lfloor Base_i \times JITTER \rfloor
$$

> $JITTER = 0.25$ (được set mặc định) và có thể tùy chỉnh.

**Ràng buộc**: Tổng độ lệch tối đa (Drift Budget) cho toàn bộ 7 step phải bằng $d_7$ của $step_7$.

## Bước 3

Hệ thống random $r_i$ (độ lệch thực tế) tương ứng với $step_i$ theo công thức:

$$
r_i \in [Min_{allowed}, Max_{allowed}]
$$

Với

$$
Min_{allowed} = \max(-d_i, -d_7 - \text{Drift}_\text{pre})
$$

$$
Max_{allowed} = \min(d_i, d_7 - \text{Drift}_\text{pre})
$$

> $Drift_{pre}$ là __TỔNG__ độ lệch tích lũy từ $step_1$ đến $step_{i - 1}$.

$$
Drift_{pre} = \sum_{k = 1}^{i - 1} r_k = r_1 + r_2 + r_3 + \ldots + r_{i - 1}
$$

### Output

Số bot bị loại thực tế ở $step_i$ là $Final_i$

$$
Final_i = Base_i + r_i
$$

## Bước 4

Số bot bị loại ở $step_7$

$$
Final_7 = T - \sum_{k = 1}^{6} Final_k
$$

## Bước 5

Phân phối phần thưởng $P$ là tổng số Gold Coins mà local player sẽ nhận được khi hoàn thành 7 step:

$$
P = \lfloor \frac{10000}{W} \rfloor
$$

---

# SIMULATION

Kịch bản mô phỏng

**Thông số đầu vào (config)**

- Cohort size = 100
- Prize = 10000 Gold Coins

| Tier | Cumulative Range | Winner | Weight |
|:---|:---|:---|:---|
| Champion | $[1, 5]$ | 1 | 0.5 |
| Top | $[6, 20]$ | $[2, 3]$ | 0.15 |
| Mid | $[21, 80]$ | $[4, 8]$ | 0.6 |
| Crowd | $[81, 100]$ | $[9, 15]$ | 0.2 |

| Step | Pacing Weight |
|:---|:---|
| 1 | $0.2$ |
| 2 | $0.15$ |
| 3 | $0.15$ |
| 4 | $0.15$ |
| 5 | $0.15$ |
| 6 | $0.1$ |
| 7 | $0.1$ |

- $JITTER = 0.25$

## CHẠY MÔ PHỎNG

1. Random $R = 45 \rightarrow \text{Mid Tier}$
2. Random $W = 6$ (tổng số winner = 6)
3. Tính tổng số bot cần loại: $T = 100 - 6 = 94$

4. Phân bổ $T = 94$ vào bảng config `Pacing Weight`

| Step | Weight | Base | Floor | Remainder |
|:---|:---|:---|:---|:---|
| 1 | 0.2 | 18.8 | 18 | 0.8 |
| 2 | 0.15 | 14.1 | 14 | 0.1 |
| 3 | 0.15 | 14.1 | 14 | 0.1 |
| 4 | 0.15 | 14.1 | 14 | 0.1 |
| 5 | 0.15 | 14.1 | 14 | 0.1 |
| 6 | 0.1 | 9.4 | 9 | 0.4 |
| 7 | 0.1 | 9.4 | 9 | 0.4 |

$\sum_{i = 1}^{7} Remainder_i = 0.8 + 0.1 + 0.1 + 0.1 + 0.1 + 0.4 + 0.4 = 2$

Cần phân bổ $Remainder = 2$ vào $step_1$ và $step_6$.

| Step | Base |
|:---|:---|
| 1 | 18 + 1 |
| 2 | 14 |
| 3 | 14 |
| 4 | 14 |
| 5 | 14 |
| 6 | 9 + 1 |
| 7 | 9 |

5. Tính __độ dao động $(d)$__ và __tổng độ lệch tối đa $(d_7)$__

| Step | Base | $d = \lfloor Base \times 0.25 \rfloor$ |
|:---|:---|:---|
| 1 | 19 | 4 |
| 2 | 14 | 3 |
| 3 | 14 | 3 |
| 4 | 14 | 3 |
| 5 | 14 | 3 |
| 6 | 10 | 2 |
| 7 | 9 | 2 (tổng độ lệch tối đa) |

6. Loại trừ Bot qua từng step

$$
Min_{allowed} = \max(-d_i, -d_7 - \text{Drift}_\text{pre})
$$

$$
Max_{allowed} = \min(d_i, d_7 - \text{Drift}_\text{pre})
$$

| Step | Base | $d = \lfloor Base \times 0.25 \rfloor$ | $Drift_{pre}$ | $[Min, Max]$ | $r$ (random) | $Final$ | $Drift_{post}$ |
|:---|:---|:---|:---|:---|:---|:---|:---|
| 1 | 19 | 4 | 0 | $[-2, 2]$ | $+1$ | 20 | $+1$ |
| 2 | 14 | 3 | $+1$ | $[-3, 1]$ | $-2$ | 12 | $-1$ |
| 3 | 14 | 3 | $-1$ | $[-1, 3]$ | $+2$ | 16 | $+1$ |
| 4 | 14 | 3 | $+1$ | $[-3, 1]$ | $0$ | 14 | $+1$ |
| 5 | 14 | 3 | $+1$ | $[-3, 1]$ | $-1$ | 13 | $0$ |
| 6 | 10 | 2 | $0$ | $[-2, 2]$ | $1$ | 11 | $+1$ |
| 7 | 9 | 2 | $+1$ | - | $-1$ (tự động) | 8 | $0$ |

$Final_7 = 94 - (20 + 12 + 16 + 14 + 13 + 11) = 8$

7. Phân phối phần thưởng cho Player

$P = \lfloor \frac{10000}{6} = 1666 \rfloor$
