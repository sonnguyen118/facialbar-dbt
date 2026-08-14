# Bus Matrix

Ma trận fact × dimension. Artifact trung tâm của lớp thiết kế logic: xác định conformed dimension, thứ tự triển khai, và những so sánh chéo nào là hợp lệ.



Bus Matrix là bảng có **hàng là các fact table** (business process) và **cột là các dimension**; ô được đánh dấu nghĩa là fact đó dùng dim đó.

Bus Matrix quyết định ba việc:
1. Cột xuất hiện ở nhiều hàng → đó là **conformed dimension**, phải dựng **một bản dùng chung**. Mỗi mart tự dựng một `dim_customer` riêng là con đường chắc chắn dẫn tới "mỗi phòng ban một con số".
2. Nó cho **thứ tự triển khai**: dim nào được dùng nhiều nhất thì làm trước.
3. Nó cho biết **so sánh chéo nào là hợp lệ** — hai fact chỉ so được với nhau qua những dim mà **cả hai** đều có.

| Fact / Business Process | Grain | date | time | customer | salon | employee | room | service | product | promotion | payment_method | campaign | junk |
|---|---|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| `fact_booking_line` | 1 dịch vụ được đặt | ✓ | ✓ | ✓ | ✓ | | | ✓ | | ✓ | | ✓ | ✓ |
| `fact_appointment` | 1 lịch hẹn | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | | | | | | ✓ |
| `fact_treatment` | 1 dịch vụ đã làm | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | | ✓ | | | ✓ |
| `fact_sales_line` | 1 dòng hoá đơn | ✓ | ✓ | ✓ | ✓ | ✓ | | ✓ | ✓ | ✓ | | ✓ | ✓ |
| `fact_payment` | 1 lần chuyển tiền | ✓ | ✓ | ✓ | ✓ | | | | | | ✓ | | |
| `fact_loyalty_txn` | 1 lần điểm biến động | ✓ | | ✓ | ✓ | | | | | | | | |
| `fact_feedback` | 1 phiếu đánh giá | ✓ | | ✓ | ✓ | ✓ | | ✓ | | | | | |
| `fact_ad_spend` | 1 ngày × chiến dịch × nền tảng | ✓ | | | | | | | | ✓ | | ✓ | |
| `fact_booking_lifecycle` | 1 booking (cập nhật dần) | ✓ | | ✓ | ✓ | ✓ | | ✓ | | ✓ | | ✓ | ✓ |
| `fact_customer_monthly_snapshot` | 1 khách × 1 tháng | ✓ | | ✓ | ✓ | | | | | | | | |

> **Ghi chú thiết kế — không có cột `membership_tier` trong matrix là có chủ đích.**
> Hạng thành viên được lưu **làm thuộc tính bên trong `dim_customer` và được SCD2 theo dõi**, không đặt FK riêng trên fact. Nhờ vậy, join fact với `dim_customer` theo thời điểm giao dịch là **tự động** ra đúng hạng thẻ tại thời điểm đó. Bảng `dim_membership_tier` vẫn tồn tại nhưng chỉ là **bảng tham chiếu quy tắc** (`discount_pct`, `point_multiplier`) cho ETL dùng, không phải chiều phân tích.

### Ba kết luận thiết kế đọc trực tiếp từ Bus Matrix

**1. `dim_date`, `dim_customer`, `dim_salon` có mặt ở gần như mọi fact** → ba dim này phải dựng đầu tiên (Sprint 1) và tuyệt đối không được tồn tại hai bản.

**2. `fact_ad_spend` có grain thô hơn hẳn** (ngày × chiến dịch, không có customer) → **cấm join trực tiếp** với `fact_sales_line`. Muốn tính ROAS phải tổng hợp `fact_sales_line` lên mức ngày × campaign **trước**, rồi mới join hai bảng cùng grain — đây chính là kỹ thuật *drilling across*:

```sql
WITH rev AS (   -- đưa fact chi tiết LÊN đúng grain của fact kia
    SELECT date_key, campaign_sk, SUM(net_amount) AS net_revenue
    FROM   dm.fact_sales_line
    WHERE  campaign_sk <> -1
    GROUP  BY date_key, campaign_sk
),
spd AS (
    SELECT date_key, campaign_sk, SUM(spend_amount) AS spend
    FROM   dm.fact_ad_spend
    GROUP  BY date_key, campaign_sk
)
SELECT COALESCE(r.date_key, s.date_key)       AS date_key,
       COALESCE(r.campaign_sk, s.campaign_sk) AS campaign_sk,
       ISNULL(r.net_revenue, 0)               AS net_revenue,
       ISNULL(s.spend, 0)                     AS spend,
       ISNULL(r.net_revenue,0) / NULLIF(s.spend, 0) AS roas
FROM   rev r
FULL OUTER JOIN spd s ON r.date_key = s.date_key AND r.campaign_sk = s.campaign_sk;
```
`FULL OUTER JOIN` là cố ý: giữ được cả chiến dịch tốn tiền mà không ra doanh thu (ROAS = 0) và doanh thu không gắn chiến dịch nào.

**3. `fact_payment` không có `dim_service`** → bảng này **không trả lời được** câu "dịch vụ nào thu được nhiều tiền nhất". Câu đó phải hỏi `fact_sales_line`. Ngược lại `fact_sales_line` không có `dim_payment_method` → câu "khách trả bằng gì nhiều nhất" phải hỏi `fact_payment`.
Đây không phải thiếu sót mà là **hệ quả tất yếu của grain**: hình thức thanh toán gắn với **lần chuyển tiền**, không gắn với **dòng hoá đơn** — một hoá đơn trả bằng 2 thẻ thì không có cách nào gán "hình thức thanh toán" cho từng dòng hoá đơn mà không bịa dữ liệu.

---
---
