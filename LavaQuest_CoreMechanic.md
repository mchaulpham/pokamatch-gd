# 1. PHASE A

## 1.1. Khởi tạo

- Thời điểm: Ngay khi Cohort được tạo (Player bấm Start)

- Logic: Hệ thống random một giá trị $R$ trong $[0, 100]$ để chọn ra __Tier__ theo bảng config bên dưới để từ đó xác định được `Target_Total_Winners`.

| Tier | Weight (%) | Cumulative Range | Winner |
|:---|:---|:---|:---|
| Champion | 5% | $[1, 5]$ | 1 |
| Top | 15% | $[6, 20]$ | $[2, 3]$ |
| Mid | 60% | $[21, 80]$ | $[4, 8]$ |
| Crowd | 20% | $[81, 100]$ | $[9, 15]$ |

> **Weight (%)**: Là tỷ lệ phần trăm xác suất để hệ thống chọn một __Tier__ cụ thể ngay tại thời điểm khởi tạo Cohort. Tổng Weight của tất cả Tier phải luôn bằng 100%.

Nếu:
  - $R \in [1, 5] \rightarrow \text{ Champion}$
  - $R \in [6, 20] \rightarrow \text{ Top}$
  - $R \in [21, 80] \rightarrow \text{ Mid}$
  - $R \in [81, 100] \rightarrow \text{ Crowd}$

- Dựa vào Tier đã chọn, tiếp tục random một số nguyên $W$ trong $[Min_{Winners}, Max_{Winners}]$ của Tier đó.

> **W** là tổng số Winner còn lại sau 7 step.

- Tính số tổng 
