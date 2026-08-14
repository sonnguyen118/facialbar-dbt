# DDL — Schema `svg_bi` (Serving / Consumption)

Bảng tổng hợp sẵn để dashboard mở dưới 2 giây. Dashboard mở 500 lượt/ngày, mỗi lượt quét fact chi tiết là lãng phí — tính một lần lúc 06:40, đọc 500 lần.

**Ranh giới bắt buộc:** Superset và Power BI **chỉ** được đọc `dm` và `svg_bi`. Cấm truy cập `lnd`, `crt`, `ctl` — những tầng đó chưa qua cổng kiểm tra chất lượng.

Quy trình làm mới: [04-etl/load-fact.md](../04-etl/load-fact.md#5-làm-mới-bảng-tổng-hợp-svg_bi).

---

## 1. Bảng cầu nối `dm.bridge_sales_promotion`

Bảng này thuộc schema **`dm`**, không thuộc `svg_bi`. DDL đặt tại đây vì nó được tạo ở bước 08 cùng với các bảng tổng hợp. Vì vậy con số "6 bảng `svg_bi`" ở mục Danh mục bảng không tính bảng này. Sáu bảng tổng hợp bắt đầu từ mục 2.

```sql
CREATE TABLE dm.bridge_sales_promotion (
    -- service_date_key có mặt để xoá-nạp theo ngày chạy được. Không có cột ngày thì
    -- không có gì để `DELETE ... WHERE` theo, và nạp lại một ngày buộc phải xoá cả bảng.
    service_date_key          INT          NOT NULL,
    invoice_line_id           BIGINT       NOT NULL,
    promotion_sk              INT          NOT NULL,
    allocation_factor         DECIMAL(9,6) NOT NULL,   -- tổng theo invoice_line_id = 1.000000
    allocated_discount_amount DECIMAL(18,2) NOT NULL,
    _run_id                   UNIQUEIDENTIFIER NOT NULL,
    CONSTRAINT PK_bridge_sales_promotion PRIMARY KEY CLUSTERED (invoice_line_id, promotion_sk),
    CONSTRAINT CK_bridge_sales_promotion_factor
        CHECK (allocation_factor > 0 AND allocation_factor <= 1),
    CONSTRAINT FK_bridge_sales_promotion_dim_promotion
        FOREIGN KEY (promotion_sk) REFERENCES dm.dim_promotion(promotion_sk),
    CONSTRAINT FK_bridge_sales_promotion_dim_date
        FOREIGN KEY (service_date_key) REFERENCES dm.dim_date(date_key)
);

CREATE NONCLUSTERED INDEX IX_bridge_sales_promotion_date
    ON dm.bridge_sales_promotion (service_date_key);
```

DQ rule đi kèm — không có nó thì bridge table vô dụng:

```sql
-- DQ-ALLOC-001: hệ số phân bổ của mỗi dòng hoá đơn phải cộng lại đúng 1
SELECT invoice_line_id, SUM(allocation_factor) AS total_factor
FROM   dm.bridge_sales_promotion
GROUP  BY invoice_line_id
HAVING ABS(SUM(allocation_factor) - 1.0) > 0.000001;
```

> ⚠️ **Quy tắc dùng bridge — bắt buộc phải viết vào Data Catalog:**
> Join `fact_sales_line` với `bridge_sales_promotion` **sẽ nhân số dòng fact lên** theo số promotion. Sau khi join, **chỉ được** dùng `allocated_discount_amount`; **cấm** dùng `net_amount` (sẽ bị nhân đôi). Đây chính là cái giá phải trả để phân tích được quan hệ nhiều-nhiều, và nó phải được ghi ra thành văn — nếu không, sẽ có người viết `SUM(net_amount)` sau khi join bridge.

## 2. `agg_revenue_daily_salon` và `agg_customer_360`

```sql
CREATE TABLE svg_bi.agg_revenue_daily_salon (
    date_key            INT      NOT NULL,
    salon_sk            INT      NOT NULL,
    -- Doanh thu: chốt sẵn cả 4 định nghĩa để không ai phải tự tính (mục 2.4)
    gross_amount        DECIMAL(18,2) NOT NULL,
    discount_amount     DECIMAL(18,2) NOT NULL,
    net_amount          DECIMAL(18,2) NOT NULL,
    net_excl_tax_amount DECIMAL(18,2) NOT NULL,
    cash_collected_amount DECIMAL(18,2) NOT NULL,
    cogs_amount         DECIMAL(18,2) NOT NULL,
    gross_margin_amount DECIMAL(18,2) NOT NULL,
    -- Tử số / mẫu số để BI tính lại tỷ lệ ở mọi mức tổng hợp
    invoice_count       INT NOT NULL,
    line_count          INT NOT NULL,
    treatment_count     INT NOT NULL,
    unique_customer_count INT NOT NULL,          -- ĐÃ tính DISTINCT ở mức ngày × salon
    new_customer_count  INT NOT NULL,
    appointment_count   INT NOT NULL,
    no_show_count       INT NOT NULL,
    busy_minutes        INT NOT NULL,
    available_minutes   INT NOT NULL,
    -- CSAT ở mức ngày × chi nhánh: lưu tử số và mẫu số, không lưu tỷ lệ (mục 2.1)
    rating_sum          INT NOT NULL,
    response_count      INT NOT NULL,
    _run_id             UNIQUEIDENTIFIER NOT NULL,
    _refreshed_at       DATETIME2(3) NOT NULL,
    CONSTRAINT PK_agg_revenue_daily_salon PRIMARY KEY CLUSTERED (date_key, salon_sk)
);
```

> ⚠️ **`unique_customer_count` là cột KHÔNG cộng được** — nó đã bị `DISTINCT` ở mức ngày × salon. Khách đến cả 3 ngày sẽ được đếm 3 lần nếu cộng qua tháng. Đếm số khách duy nhất trong tháng **buộc phải** quay lại `fact_sales_line`.
> Đây là hạn chế cố hữu của mọi bảng tổng hợp: **`COUNT(DISTINCT ...)` không tổng hợp trước được.** Cách xử lý đúng: đặt tên cột tường minh (`..._count` ở mức grain của bảng), ghi rõ vào catalog, và tạo thêm bảng `agg_revenue_monthly_salon` có cột `unique_customer_count` tính riêng ở mức tháng.

```sql
-- Bảng chân dung khách hàng: 1 dòng = 1 khách. Dùng cho CRM và làm feature store cho ML.
CREATE TABLE svg_bi.agg_customer_360 (
    customer_sk           BIGINT NOT NULL CONSTRAINT PK_agg_customer_360 PRIMARY KEY CLUSTERED,
    customer_id           BIGINT NOT NULL,
    -- RFM
    last_visit_date       DATE   NULL,
    days_since_last_visit SMALLINT NULL,
    visit_count_lifetime  INT    NOT NULL,
    visit_count_l90d      SMALLINT NOT NULL,
    net_amount_lifetime   DECIMAL(18,2) NOT NULL,
    net_amount_l365d      DECIMAL(18,2) NOT NULL,
    avg_ticket_amount     DECIMAL(18,2) NOT NULL,
    rfm_segment           VARCHAR(30) NOT NULL,
    -- Hành vi
    favourite_service_sk  INT    NOT NULL,
    favourite_salon_sk    INT    NOT NULL,
    favourite_employee_sk INT    NOT NULL,
    avg_rating            DECIMAL(3,2) NULL,
    cancel_count_lifetime SMALLINT NOT NULL,
    no_show_count_lifetime SMALLINT NOT NULL,
    -- Thành viên & điểm
    membership_tier       VARCHAR(20) NOT NULL,
    point_balance         INT    NOT NULL,
    -- Nhãn / dự đoán của ML
    is_churned            BIT    NOT NULL,
    churn_probability     DECIMAL(5,4) NULL,
    predicted_clv_amount  DECIMAL(18,2) NULL,
    -- MỐC CHỐT FEATURE: bắt buộc, để chống data leakage (mục 6.3)
    feature_cutoff_date   DATE   NOT NULL,
    _run_id               UNIQUEIDENTIFIER NOT NULL,
    _refreshed_at         DATETIME2(3) NOT NULL
);
```

Cột `feature_cutoff_date` là điểm nối trực tiếp với nguyên tắc chống **data leakage** ở mục 6.3: mọi feature trong dòng này chỉ được tính từ dữ liệu có `occurred_at <= feature_cutoff_date`. Không có cột này thì không thể chứng minh model được huấn luyện sạch.

---

---

## 3. `agg_funnel_daily` — Phễu chuyển đổi

Grain: 1 dòng = 1 ngày × 1 salon. Nguồn: `fact_booking_lifecycle` + `crt.service_view`.

```sql
CREATE TABLE svg_bi.agg_funnel_daily (
    date_key           INT NOT NULL,
    salon_sk           INT NOT NULL,
    -- Các bậc phễu: đều là số đếm nên additive, cộng lên tháng/quý được
    session_cnt        INT NOT NULL,   -- phiên có xem trang dịch vụ
    service_view_cnt   INT NOT NULL,
    booking_cnt        INT NOT NULL,
    confirmed_cnt      INT NOT NULL,
    checkin_cnt        INT NOT NULL,
    treatment_cnt      INT NOT NULL,
    paid_cnt           INT NOT NULL,
    feedback_cnt       INT NOT NULL,
    -- Rơi khỏi phễu
    cancelled_cnt      INT NOT NULL,
    no_show_cnt        INT NOT NULL,
    -- Thời gian mỗi chặng: lưu TỔNG và MẪU SỐ, không lưu trung bình (non-additive)
    sum_hours_book_to_confirm DECIMAL(18,2) NOT NULL,
    cnt_hours_book_to_confirm INT           NOT NULL,
    sum_hours_book_to_pay     DECIMAL(18,2) NOT NULL,
    cnt_hours_book_to_pay     INT           NOT NULL,
    sum_wait_minutes          INT           NOT NULL,
    cnt_wait_minutes          INT           NOT NULL,
    _run_id            UNIQUEIDENTIFIER NOT NULL,
    _refreshed_at      DATETIME2(3) NOT NULL,
    CONSTRAINT PK_agg_funnel_daily PRIMARY KEY CLUSTERED (date_key, salon_sk),
    CONSTRAINT CK_agg_funnel_nonneg CHECK (session_cnt >= 0 AND booking_cnt >= 0),
    -- Phễu phải thu hẹp dần
    CONSTRAINT CK_agg_funnel_shape CHECK (paid_cnt <= treatment_cnt
                                      AND treatment_cnt <= checkin_cnt
                                      AND checkin_cnt <= confirmed_cnt
                                      AND confirmed_cnt <= booking_cnt),
    CONSTRAINT FK_agg_funnel_dim_date  FOREIGN KEY (date_key) REFERENCES dm.dim_date(date_key),
    CONSTRAINT FK_agg_funnel_dim_salon FOREIGN KEY (salon_sk) REFERENCES dm.dim_salon(salon_sk)
);
```

`CK_agg_funnel_shape` là hàng rào phát hiện lỗi logic nạp: phễu không bao giờ mở rộng ra ở bậc sau.

Bảng này giải bài toán **Booking Conversion** đúng cách: cả tử số và mẫu số đã được đưa về **cùng grain `ngày × salon`**, nên chia trực tiếp là hợp lệ — không còn tình huống chia giữa hai fact khác grain.

```sql
-- Booking Conversion toàn chuỗi trong tháng
SELECT SUM(booking_cnt) * 1.0 / NULLIF(SUM(session_cnt), 0) AS booking_conversion,
       SUM(paid_cnt)    * 1.0 / NULLIF(SUM(booking_cnt), 0) AS booking_to_paid,
       SUM(sum_wait_minutes) * 1.0 / NULLIF(SUM(cnt_wait_minutes), 0) AS avg_wait_min
FROM   svg_bi.agg_funnel_daily f
JOIN   dm.dim_date d ON d.date_key = f.date_key
WHERE  d.year_month = 202608;
```

---

## 4. `agg_service_perf_monthly` — Hiệu quả dịch vụ

Grain: 1 dòng = 1 tháng × 1 salon × 1 dịch vụ.

```sql
CREATE TABLE svg_bi.agg_service_perf_monthly (
    year_month           INT NOT NULL,
    salon_sk             INT NOT NULL,
    service_sk           INT NOT NULL,
    -- Doanh thu và lãi
    gross_amount         DECIMAL(18,2) NOT NULL,
    discount_amount      DECIMAL(18,2) NOT NULL,
    net_amount           DECIMAL(18,2) NOT NULL,
    cogs_amount          DECIMAL(18,2) NOT NULL,
    gross_margin_amount  DECIMAL(18,2) NOT NULL,
    -- Sản lượng
    line_cnt             INT NOT NULL,
    treatment_cnt        INT NOT NULL,
    quantity             DECIMAL(18,2) NOT NULL,
    upsell_treatment_cnt INT NOT NULL,
    -- Thời lượng: tử số và mẫu số
    sum_busy_minutes     INT NOT NULL,
    sum_standard_minutes INT NOT NULL,
    sum_overrun_minutes  INT NOT NULL,
    -- Hài lòng: tử số và mẫu số, KHÔNG lưu trung bình
    sum_rating           INT NOT NULL,
    response_cnt         INT NOT NULL,
    satisfied_cnt        INT NOT NULL,
    -- Không cộng được qua tháng: đã DISTINCT ở grain này
    unique_customer_cnt  INT NOT NULL,
    _run_id              UNIQUEIDENTIFIER NOT NULL,
    _refreshed_at        DATETIME2(3) NOT NULL,
    CONSTRAINT PK_agg_service_perf_monthly PRIMARY KEY CLUSTERED (year_month, salon_sk, service_sk),
    CONSTRAINT FK_agg_svcperf_dim_salon   FOREIGN KEY (salon_sk)   REFERENCES dm.dim_salon(salon_sk),
    CONSTRAINT FK_agg_svcperf_dim_service FOREIGN KEY (service_sk) REFERENCES dm.dim_service(service_sk)
);
```

> `unique_customer_cnt` **không cộng được** qua tháng hoặc qua salon — khách dùng dịch vụ ở 2 salon sẽ bị đếm 2 lần. Muốn số khách duy nhất ở mức cao hơn phải quay lại `dm.fact_sales_line`. Ghi rõ điều này vào `ctl.metric_definition`.

---

## 5. `agg_therapist_utilization_daily` — Năng suất kỹ thuật viên

Grain: 1 dòng = 1 ngày × 1 kỹ thuật viên.

```sql
CREATE TABLE svg_bi.agg_therapist_utilization_daily (
    date_key            INT NOT NULL,
    employee_sk         INT NOT NULL,
    salon_sk            INT NOT NULL,
    -- Tử số / mẫu số của Utilization. KHÔNG lưu utilization_pct.
    busy_minutes        INT NOT NULL,
    available_minutes   INT NOT NULL,
    overrun_minutes     INT NOT NULL,
    -- Sản lượng và doanh thu quy về KTV
    treatment_cnt       INT NOT NULL,
    upsell_cnt          INT NOT NULL,
    net_amount          DECIMAL(18,2) NOT NULL,
    gross_margin_amount DECIMAL(18,2) NOT NULL,
    unique_customer_cnt INT NOT NULL,
    -- Chất lượng phục vụ
    sum_rating          INT NOT NULL,
    response_cnt        INT NOT NULL,
    satisfied_cnt       INT NOT NULL,
    -- Chất lượng dữ liệu: 1 = ngày này không có lịch làm việc nên mẫu số không tin được
    is_shift_missing    BIT NOT NULL,
    _run_id             UNIQUEIDENTIFIER NOT NULL,
    _refreshed_at       DATETIME2(3) NOT NULL,
    CONSTRAINT PK_agg_therapist_util_daily PRIMARY KEY CLUSTERED (date_key, employee_sk),
    CONSTRAINT CK_agg_util_nonneg CHECK (busy_minutes >= 0 AND available_minutes >= 0),
    -- Làm quá giờ ca là hợp lệ nhưng phải trong giới hạn hợp lý
    CONSTRAINT CK_agg_util_ratio CHECK (available_minutes = 0 OR busy_minutes <= available_minutes * 2),
    CONSTRAINT FK_agg_util_dim_date     FOREIGN KEY (date_key)    REFERENCES dm.dim_date(date_key),
    CONSTRAINT FK_agg_util_dim_employee FOREIGN KEY (employee_sk) REFERENCES dm.dim_employee(employee_sk),
    CONSTRAINT FK_agg_util_dim_salon    FOREIGN KEY (salon_sk)    REFERENCES dm.dim_salon(salon_sk)
);
```

> `is_shift_missing` là cột **trung thực về chất lượng dữ liệu**. `available_minutes` lấy từ `lnd.hr_shift` — dữ liệu [chưa có nguồn](../02-mapping/source-to-target.md#phụ-thuộc-bên-ngoài-chưa-có). Ngày nào thiếu lịch làm việc thì mẫu số bằng 0 và Utilization không tính được. Dashboard phải loại các dòng `is_shift_missing = 1` khỏi phép tính và hiển thị rõ số ngày bị loại, thay vì âm thầm cho ra tỷ lệ sai.

---

## 6. `agg_cohort_retention` — Giữ chân khách theo nhóm

Grain: 1 dòng = 1 cohort (tháng khách đến lần đầu) × 1 tháng thứ N kể từ đó.

```sql
CREATE TABLE svg_bi.agg_cohort_retention (
    cohort_year_month   INT NOT NULL,   -- 202601 = nhóm khách đến lần đầu tháng 01/2026
    month_offset        SMALLINT NOT NULL,  -- 0 = chính tháng đó, 1 = tháng kế tiếp...
    -- Mẫu số: cố định theo cohort, không đổi qua các month_offset
    cohort_size         INT NOT NULL,
    -- Tử số
    active_customer_cnt INT NOT NULL,
    net_amount          DECIMAL(18,2) NOT NULL,
    treatment_cnt       INT NOT NULL,
    _run_id             UNIQUEIDENTIFIER NOT NULL,
    _refreshed_at       DATETIME2(3) NOT NULL,
    CONSTRAINT PK_agg_cohort_retention PRIMARY KEY CLUSTERED (cohort_year_month, month_offset),
    CONSTRAINT CK_agg_cohort_offset CHECK (month_offset BETWEEN 0 AND 60),
    CONSTRAINT CK_agg_cohort_size   CHECK (cohort_size > 0),
    -- Số khách còn hoạt động không thể vượt quy mô cohort
    CONSTRAINT CK_agg_cohort_active CHECK (active_customer_cnt <= cohort_size)
);
```

```sql
-- Ma trận cohort: tỷ lệ giữ chân theo tháng thứ N
SELECT cohort_year_month, cohort_size,
       MAX(CASE WHEN month_offset = 1 THEN active_customer_cnt END) * 1.0 / cohort_size AS m1,
       MAX(CASE WHEN month_offset = 3 THEN active_customer_cnt END) * 1.0 / cohort_size AS m3,
       MAX(CASE WHEN month_offset = 6 THEN active_customer_cnt END) * 1.0 / cohort_size AS m6,
       MAX(CASE WHEN month_offset =12 THEN active_customer_cnt END) * 1.0 / cohort_size AS m12
FROM   svg_bi.agg_cohort_retention
GROUP  BY cohort_year_month, cohort_size
ORDER  BY cohort_year_month;
```

> **`cohort_size` cố định theo cohort là điểm dễ làm sai nhất.** Mẫu số phải là số khách của tháng gốc, giữ nguyên qua mọi `month_offset`. Nếu lấy mẫu số là số khách còn hoạt động ở tháng trước thì ra chỉ số khác hoàn toàn (retention theo chuỗi, không phải theo cohort) và không so được với số liệu ngành.
>
> Cohort chưa đủ tuổi không được nạp dòng: cohort tháng 08/2026 chưa có `month_offset = 3` vào tháng 09/2026. Nạp dòng với `active_customer_cnt = 0` sẽ làm biểu đồ tụt xuống 0 một cách sai lệch.

---

## Danh mục bảng

| # | Bảng | Grain | Dòng/năm (20 salon) | Nguồn |
|---|---|---|---|---|
| 1 | `agg_revenue_daily_salon` | ngày × salon | ~7.000 | `fact_sales_line`, `fact_payment`, `fact_appointment`, `fact_feedback` |
| 2 | `agg_customer_360` | 1 khách | ~35.000 | Nhiều fact + ML |
| 3 | `agg_funnel_daily` | ngày × salon | ~7.000 | `fact_booking_lifecycle`, `crt.service_view` |
| 4 | `agg_service_perf_monthly` | tháng × salon × dịch vụ | ~36.000 | `fact_sales_line`, `fact_treatment`, `fact_feedback` |
| 5 | `agg_therapist_utilization_daily` | ngày × KTV | ~140.000 | `fact_treatment`, `lnd.hr_shift` |
| 6 | `agg_cohort_retention` | cohort × tháng thứ N | ~2.000 | `fact_sales_line` |

**Tổng: 6 bảng `svg_bi`**, chưa tính `dm.bridge_sales_promotion` ở mục 1. Toàn bộ dùng rowstore clustered — bảng nhỏ, truy vấn theo khoảng ngày, không cần columnstore.

## Nguyên tắc thiết kế bảng tổng hợp

| # | Nguyên tắc | Lý do |
|---|---|---|
| 1 | **Không lưu tỷ lệ.** Chỉ lưu tử số và mẫu số | Tỷ lệ là non-additive; `AVG` của tỷ lệ cho kết quả sai lệch nhiều lần |
| 2 | Cột `COUNT(DISTINCT ...)` đặt tên rõ grain và ghi cảnh báo vào catalog | Không tổng hợp tiếp lên mức cao hơn được |
| 3 | Có `CHECK` kiểm tra hình dạng dữ liệu (phễu thu hẹp, tỷ lệ trong biên) | Bắt lỗi logic nạp ngay tại database |
| 4 | Có cột cờ trung thực về chất lượng dữ liệu (`is_shift_missing`) | Thà báo "không tính được" hơn là cho ra số sai |
| 5 | Đối chiếu tổng với fact nguồn sau mỗi lần làm mới | `DQ-RECON-004` |
