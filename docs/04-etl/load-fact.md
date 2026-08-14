# ETL — Nạp Fact và bảng tổng hợp

10 fact table, 1 bridge, 6 bảng tổng hợp. Chạy **sau** khi toàn bộ dimension đã nạp xong và ba quy tắc SCD đã pass.

| Loại fact | Bảng | Cơ chế nạp |
|---|---|---|
| Transaction | `fact_sales_line`, `fact_payment`, `fact_booking_line`, `fact_appointment`, `fact_treatment`, `fact_loyalty_txn`, `fact_feedback` | Delete-insert theo phân vùng |
| Transaction (nạp lại 7 ngày) | `fact_ad_spend` | Delete-insert 7 phân vùng gần nhất |
| Accumulating snapshot | `fact_booking_lifecycle` | `MERGE` — **bảng duy nhất được UPDATE** |
| Periodic snapshot | `fact_customer_monthly_snapshot` | Delete-insert theo `year_month` |
| Bridge | `bridge_sales_promotion` | Xoá-nạp theo `service_date_key` |

---

## 1. Ba nguyên tắc chung

**Idempotent.** Chạy lại với cùng dữ liệu nguồn phải ra cùng kết quả. Sự cố hạ tầng dẫn tới chạy lại là tình huống thường trực trong vận hành; không lũy đẳng thì mỗi lần chạy lại cộng thêm một bản doanh thu.

**Tra khoá dimension bằng `LEFT JOIN` + `ISNULL(..., -1)`.** `INNER JOIN` làm mất dòng fact khi thiếu dimension — mất doanh thu không có thông báo lỗi.

**Tra SCD2 theo thời điểm nghiệp vụ.** Dùng `is_current = 1` cho fact lịch sử sẽ gán thuộc tính hiện tại vào giao dịch quá khứ.

```sql
-- Khuôn tra khoá SCD2 chuẩn, dùng lại cho mọi fact
LEFT JOIN dm.dim_customer c
       ON  c.customer_id = src.customer_id
       AND src.<thời điểm nghiệp vụ> >= c.valid_from
       AND src.<thời điểm nghiệp vụ> <  c.valid_to
```

---

## 2. Transaction fact — `fact_sales_line`

Bảng trung tâm. Các transaction fact khác theo đúng khuôn này.

```sql
CREATE OR ALTER PROCEDURE dm.usp_load_fact_sales_line
    @business_date DATE,
    @run_id        UNIQUEIDENTIFIER
AS
BEGIN
    SET NOCOUNT, XACT_ABORT ON;

    DECLARE @date_key INT = YEAR(@business_date)*10000 + MONTH(@business_date)*100 + DAY(@business_date);
    DECLARE @partition_number INT =
        $PARTITION.pf_date_key_month(@date_key);

    BEGIN TRAN;

    -- Xoá sạch phân vùng của ngày nghiệp vụ này.
    -- TRUNCATE PARTITION nhanh hơn DELETE rất nhiều và không làm tăng transaction log.
    -- Lưu ý: phân vùng theo THÁNG nên chỉ dùng TRUNCATE khi nạp lại cả tháng;
    -- nạp lại một ngày thì dùng DELETE có điều kiện.
    DELETE FROM dm.fact_sales_line WHERE service_date_key = @date_key;

    INSERT INTO dm.fact_sales_line (
        service_date_key, invoice_date_key, service_time_key,
        customer_sk, salon_sk, employee_sk, service_sk, product_sk,
        promotion_sk, campaign_sk, booking_junk_sk,
        invoice_line_id, invoice_no, invoice_line_no, treatment_id, booking_id,
        quantity, unit_price_amount, gross_amount,
        promo_discount_amount, member_discount_amount, discount_amount,
        net_amount, tax_amount, net_excl_tax_amount,
        cogs_amount, gross_margin_amount, line_count, _run_id)
    SELECT
        @date_key,
        YEAR(i.invoiced_at)*10000 + MONTH(i.invoiced_at)*100 + DAY(i.invoiced_at),
        DATEPART(HOUR, i.service_at)*60 + DATEPART(MINUTE, i.service_at),
        ISNULL(c.customer_sk,  -1),
        ISNULL(s.salon_sk,     -1),
        ISNULL(e.employee_sk,  -1),
        ISNULL(sv.service_sk,  -1),
        ISNULL(p.product_sk,   -1),
        ISNULL(pr.promotion_sk,-1),
        ISNULL(cp.campaign_sk, -1),
        ISNULL(j.booking_junk_sk, -1),
        l.invoice_line_id, i.invoice_no, l.invoice_line_no, l.treatment_id, ap.booking_id,
        l.quantity, l.unit_price_amount, l.gross_amount,
        l.promo_discount_amount, l.member_discount_amount, l.discount_amount,
        l.net_amount, l.tax_amount, l.net_excl_tax_amount,
        l.cogs_amount, l.gross_margin_amount, 1, @run_id
    FROM       crt.invoice_line l
    JOIN       crt.invoice      i  ON i.invoice_id = l.invoice_id
    LEFT JOIN  crt.treatment    t  ON t.treatment_id = l.treatment_id
    -- SCD2: tra theo service_at, KHÔNG dùng is_current
    LEFT JOIN  dm.dim_customer  c  ON c.customer_id = i.customer_id
                                  AND i.service_at >= c.valid_from AND i.service_at < c.valid_to
    LEFT JOIN  dm.dim_salon     s  ON s.salon_id    = i.salon_id
                                  AND i.service_at >= s.valid_from AND i.service_at < s.valid_to
    LEFT JOIN  dm.dim_employee  e  ON e.employee_id = l.employee_id
                                  AND i.service_at >= e.valid_from AND i.service_at < e.valid_to
    LEFT JOIN  dm.dim_service   sv ON sv.service_id = l.service_id
                                  AND i.service_at >= sv.valid_from AND i.service_at < sv.valid_to
    -- SCD1: tra trực tiếp
    LEFT JOIN  dm.dim_product   p  ON p.product_id   = l.product_id
    LEFT JOIN  dm.dim_promotion pr ON pr.promotion_id = l.promotion_id
    LEFT JOIN  dm.dim_campaign  cp ON cp.campaign_id  = i.campaign_id
    -- ap, b, fv phải khai TRƯỚC join vào junk dim: mệnh đề ON chỉ tham chiếu được
    -- các nguồn đã xuất hiện phía trên, đặt sau sẽ lỗi Msg 4104 could not be bound.
    LEFT JOIN  crt.appointment  ap ON ap.appointment_id = t.appointment_id
    LEFT JOIN  crt.booking      b  ON b.booking_id = ap.booking_id
    -- is_first_visit: có hoá đơn nào TRƯỚC service_at của cùng khách hay không
    OUTER APPLY (SELECT TOP (1) i2.customer_id
                 FROM   crt.invoice i2
                 WHERE  i2.customer_id = i.customer_id
                   AND  i2.service_at  < i.service_at
                   AND  i2._is_deleted = 0
                   AND  i2.invoice_status <> 'void') fv
    -- Junk: khớp đồng thời cả 5 cờ
    LEFT JOIN  dm.dim_booking_junk j
           ON  j.booking_channel      = ISNULL(b.booking_channel, 'unknown')
           AND j.is_first_visit       = CASE WHEN fv.customer_id IS NULL THEN 1 ELSE 0 END
           AND j.is_promotion_applied = CASE WHEN l.promotion_id IS NOT NULL THEN 1 ELSE 0 END
           AND j.is_member            = CASE WHEN l.member_discount_amount > 0 THEN 1 ELSE 0 END
           AND j.is_rescheduled       = CASE WHEN b.booking_status = 'rescheduled' THEN 1 ELSE 0 END
    WHERE  CAST(i.service_at AS DATE) = @business_date
      AND  i.invoice_status <> 'void'
      AND  i._is_deleted = 0
      AND  l._is_deleted = 0;

    -- Chỉ tiến, không lùi: nếu không có điều kiện này thì một lần backfill ngày cũ
    -- sẽ đẩy watermark về quá khứ và lần thu nạp incremental kế tiếp đọc lại từ đó.
    -- Dòng watermark của bước nạp dm tách riêng khỏi dòng của bước thu nạp.
    UPDATE ctl.watermark
       SET last_value = CONVERT(VARCHAR(10), @business_date, 23),
           last_run_id = @run_id, updated_at = SYSUTCDATETIME()
     WHERE source_name = 'dm' AND entity_name = 'fact_sales_line'
       AND last_value < CONVERT(VARCHAR(10), @business_date, 23);

    COMMIT;
END
```

### Các transaction fact còn lại

Cùng khuôn, khác nguồn và cột chốt kỳ:

| Fact | Nguồn `crt` | Cột chốt kỳ | Khoá grain |
|---|---|---|---|
| `fact_payment` | `payment` | `paid_at` | `payment_id` |
| `fact_booking_line` | `booking_item` + `booking` | `booked_at` | `booking_item_id` |
| `fact_appointment` | `appointment` | `slot_at` | `appointment_id` |
| `fact_treatment` | `treatment` | `started_at` | `treatment_id` |
| `fact_loyalty_txn` | `loyalty_transaction` | `txn_at` | `loyalty_txn_id` |
| `fact_feedback` | `feedback` | `feedback_at` | `feedback_id` |

### `fact_ad_spend` — nạp lại 7 ngày

Nền tảng quảng cáo điều chỉnh lại số liệu trong 7 ngày, nên nạp lại cửa sổ 7 ngày mỗi lần chạy:

```sql
CREATE OR ALTER PROCEDURE dm.usp_load_fact_ad_spend
    @business_date DATE, @run_id UNIQUEIDENTIFIER
AS
BEGIN
    SET NOCOUNT, XACT_ABORT ON;
    DECLARE @from DATE = DATEADD(DAY, -6, @business_date);

    BEGIN TRAN;
    DELETE f FROM dm.fact_ad_spend f
    JOIN dm.dim_date d ON d.date_key = f.spend_date_key
    WHERE d.full_date BETWEEN @from AND @business_date;

    INSERT INTO dm.fact_ad_spend (spend_date_key, campaign_sk, promotion_sk, platform,
        spend_amount, impression_count, click_count, lead_count, _run_id)
    SELECT YEAR(a.spend_date)*10000 + MONTH(a.spend_date)*100 + DAY(a.spend_date),
           ISNULL(c.campaign_sk, -1), -1, a.platform,
           a.spend_amount, a.impression_count, a.click_count, a.lead_count, @run_id
    FROM   crt.ad_spend a
    LEFT JOIN dm.dim_campaign c ON c.campaign_id = a.campaign_id
    WHERE  a.spend_date BETWEEN @from AND @business_date;
    COMMIT;
END
```

> Không nạp lại 7 ngày thì số chi phí quảng cáo của tuần trước sẽ **cố định ở giá trị tạm** mà nền tảng đã điều chỉnh lại — làm ROAS và CAC sai lệch một cách hệ thống.

---

## 3. Accumulating snapshot — `fact_booking_lifecycle`

Bảng **duy nhất** trong `dm` được `UPDATE`. Mỗi booking là một dòng, cập nhật dần qua từng mốc.

```sql
CREATE OR ALTER PROCEDURE dm.usp_load_fact_booking_lifecycle
    @business_date DATE, @run_id UNIQUEIDENTIFIER
AS
BEGIN
    SET NOCOUNT, XACT_ABORT ON;
    DECLARE @now DATETIME2(3) = SYSUTCDATETIME();

    -- Chỉ xử lý booking CÓ BIẾN ĐỘNG: mới tạo, hoặc có mốc mới trong ngày.
    -- Quét lại toàn bộ lịch sử mỗi đêm là không cần thiết và rất chậm.
    SELECT b.booking_id,
           b.customer_id, b.salon_id, b.booked_at, b.booking_status, b.promotion_id, b.campaign_id,
           ap.appointment_id, ap.checkin_at, ap.is_no_show,
           t.treatment_id, t.employee_id, t.service_id, t.started_at,
           pay.paid_at, amt.net_amount,
           fb.feedback_at,
           b.cancelled_at
    INTO   #chg
    FROM   crt.booking b
    LEFT JOIN crt.appointment ap ON ap.booking_id = b.booking_id
    OUTER APPLY (SELECT TOP (1) t.treatment_id, t.employee_id, t.service_id, t.started_at
                 FROM crt.treatment t WHERE t.appointment_id = ap.appointment_id
                 ORDER BY t.started_at) t
    -- Tách làm hai: nối invoice_line với payment trên cùng invoice_id sinh tích
    -- Descartes, nên hoá đơn trả 2 lần sẽ làm net_amount gấp đôi.
    OUTER APPLY (SELECT SUM(l.net_amount) AS net_amount
                 FROM crt.invoice_line l
                 WHERE l.treatment_id = t.treatment_id AND l._is_deleted = 0) amt
    OUTER APPLY (SELECT MIN(p.paid_at) AS paid_at
                 FROM crt.invoice i
                 JOIN crt.payment p ON p.invoice_id = i.invoice_id
                 WHERE p.payment_status = 'completed'
                   AND EXISTS (SELECT 1 FROM crt.invoice_line l2
                               WHERE l2.invoice_id = i.invoice_id
                                 AND l2.treatment_id = t.treatment_id)) pay
    OUTER APPLY (SELECT MIN(f.feedback_at) AS feedback_at
                 FROM crt.feedback f WHERE f.treatment_id = t.treatment_id) fb
    WHERE  b._is_deleted = 0
      AND (CAST(b.booked_at     AS DATE) = @business_date
        OR CAST(b._updated_at   AS DATE) = @business_date
        OR CAST(ap.checkin_at   AS DATE) = @business_date
        OR CAST(t.started_at    AS DATE) = @business_date
        OR CAST(pay.paid_at     AS DATE) = @business_date
        OR CAST(fb.feedback_at  AS DATE) = @business_date
        OR CAST(b.cancelled_at  AS DATE) = @business_date);

    BEGIN TRAN;

    -- crt.appointment.booking_id KHÔNG unique (khách đổi lịch sinh nhiều appointment),
    -- nên #chg có thể chứa hai dòng cùng booking_id. MERGE gặp khoá trùng sẽ lỗi
    -- Msg 8672 và XACT_ABORT làm rollback cả job. Giữ lại appointment sớm nhất.
    WITH dup AS (
        SELECT ROW_NUMBER() OVER (PARTITION BY booking_id
                                  ORDER BY checkin_at, appointment_id) AS rn
        FROM #chg)
    DELETE FROM dup WHERE rn > 1;

    MERGE dm.fact_booking_lifecycle AS tgt
    USING (
        SELECT s.booking_id,
               ISNULL(c.customer_sk, -1)  AS customer_sk,
               ISNULL(sl.salon_sk,   -1)  AS salon_sk,
               ISNULL(e.employee_sk, -1)  AS employee_sk,
               ISNULL(sv.service_sk, -1)  AS service_sk,
               ISNULL(pr.promotion_sk,-1) AS promotion_sk,
               ISNULL(cp.campaign_sk, -1) AS campaign_sk,
               -1                         AS booking_junk_sk,
               dbk.date_key AS booked_date_key,
               dcf.date_key AS confirmed_date_key,
               dci.date_key AS checkin_date_key,
               dtr.date_key AS treatment_date_key,
               dpy.date_key AS paid_date_key,
               dfb.date_key AS feedback_date_key,
               dcx.date_key AS cancelled_date_key,
               DATEDIFF(MINUTE, s.booked_at,  s.checkin_at)  / 60.0 AS hours_confirm_to_visit,
               DATEDIFF(MINUTE, s.checkin_at, s.started_at)  / 60.0 AS hours_checkin_to_start,
               DATEDIFF(MINUTE, s.started_at, s.paid_at)     / 60.0 AS hours_treat_to_pay,
               DATEDIFF(MINUTE, s.booked_at,  s.paid_at)     / 60.0 AS hours_book_to_pay,
               CASE WHEN s.booking_status IN ('confirmed','completed') THEN 1 ELSE 0 END AS reached_confirmed,
               CASE WHEN s.checkin_at   IS NOT NULL THEN 1 ELSE 0 END AS reached_checkin,
               CASE WHEN s.treatment_id IS NOT NULL THEN 1 ELSE 0 END AS reached_treatment,
               CASE WHEN s.paid_at      IS NOT NULL THEN 1 ELSE 0 END AS reached_payment,
               CASE WHEN s.feedback_at  IS NOT NULL THEN 1 ELSE 0 END AS reached_feedback,
               CASE WHEN s.cancelled_at IS NOT NULL THEN 1 ELSE 0 END AS is_cancelled,
               ISNULL(s.is_no_show, 0)                                AS is_no_show,
               ISNULL(s.net_amount, 0)                                AS net_amount
        FROM   #chg s
        LEFT JOIN dm.dim_customer  c  ON c.customer_id = s.customer_id
                                     AND s.booked_at >= c.valid_from AND s.booked_at < c.valid_to
        LEFT JOIN dm.dim_salon     sl ON sl.salon_id = s.salon_id
                                     AND s.booked_at >= sl.valid_from AND s.booked_at < sl.valid_to
        -- Temporal join, KHÔNG dùng is_current: MERGE chạy lại mỗi đêm nên
        -- is_current sẽ trỏ booking cũ về phiên bản hiện tại của KTV/dịch vụ,
        -- làm phễu của quá khứ tự thay đổi sau mỗi lần chạy.
        LEFT JOIN dm.dim_employee  e  ON e.employee_id = s.employee_id
                                     AND s.booked_at >= e.valid_from AND s.booked_at < e.valid_to
        LEFT JOIN dm.dim_service   sv ON sv.service_id = s.service_id
                                     AND s.booked_at >= sv.valid_from AND s.booked_at < sv.valid_to
        LEFT JOIN dm.dim_promotion pr ON pr.promotion_id = s.promotion_id
        LEFT JOIN dm.dim_campaign  cp ON cp.campaign_id  = s.campaign_id
        LEFT JOIN dm.dim_date dbk ON dbk.full_date = CAST(s.booked_at    AS DATE)
        -- crt.booking CHƯA có cột confirmed_at, nên mốc xác nhận không lấy được.
        -- Để NULL thay vì gán bằng booked_at: gán bằng booked_at sẽ cho ra
        -- "100% booking xác nhận ngay trong ngày đặt" và vi phạm CK_fbc_flag_confirm.
        -- Bổ sung crt.booking.confirmed_at là điều kiện để tính hours_book_to_confirm.
        LEFT JOIN dm.dim_date dcf ON 1 = 0
        LEFT JOIN dm.dim_date dci ON dci.full_date = CAST(s.checkin_at   AS DATE)
        LEFT JOIN dm.dim_date dtr ON dtr.full_date = CAST(s.started_at   AS DATE)
        LEFT JOIN dm.dim_date dpy ON dpy.full_date = CAST(s.paid_at      AS DATE)
        LEFT JOIN dm.dim_date dfb ON dfb.full_date = CAST(s.feedback_at  AS DATE)
        LEFT JOIN dm.dim_date dcx ON dcx.full_date = CAST(s.cancelled_at AS DATE)
    ) AS src
       ON tgt.booking_id = src.booking_id
    WHEN MATCHED THEN UPDATE SET
        tgt.confirmed_date_key = src.confirmed_date_key,
        tgt.checkin_date_key   = src.checkin_date_key,
        tgt.treatment_date_key = src.treatment_date_key,
        tgt.paid_date_key      = src.paid_date_key,
        tgt.feedback_date_key  = src.feedback_date_key,
        tgt.cancelled_date_key = src.cancelled_date_key,
        tgt.employee_sk        = src.employee_sk,
        tgt.service_sk         = src.service_sk,
        tgt.hours_confirm_to_visit = src.hours_confirm_to_visit,
        tgt.hours_checkin_to_start = src.hours_checkin_to_start,
        tgt.hours_treat_to_pay     = src.hours_treat_to_pay,
        tgt.hours_book_to_pay      = src.hours_book_to_pay,
        tgt.reached_confirmed  = src.reached_confirmed,
        tgt.reached_checkin    = src.reached_checkin,
        tgt.reached_treatment  = src.reached_treatment,
        tgt.reached_payment    = src.reached_payment,
        tgt.reached_feedback   = src.reached_feedback,
        tgt.is_cancelled       = src.is_cancelled,
        tgt.is_no_show         = src.is_no_show,
        tgt.net_amount         = src.net_amount,
        tgt._run_id            = @run_id,
        tgt._updated_at        = @now
    WHEN NOT MATCHED BY TARGET THEN INSERT (
        booking_id, customer_sk, salon_sk, employee_sk, service_sk, promotion_sk,
        campaign_sk, booking_junk_sk,
        booked_date_key, confirmed_date_key, checkin_date_key, treatment_date_key,
        paid_date_key, feedback_date_key, cancelled_date_key,
        hours_confirm_to_visit, hours_checkin_to_start, hours_treat_to_pay, hours_book_to_pay,
        reached_confirmed, reached_checkin, reached_treatment, reached_payment, reached_feedback,
        is_cancelled, is_no_show, net_amount, _run_id, _updated_at)
    VALUES (
        src.booking_id, src.customer_sk, src.salon_sk, src.employee_sk, src.service_sk,
        src.promotion_sk, src.campaign_sk, src.booking_junk_sk,
        src.booked_date_key, src.confirmed_date_key, src.checkin_date_key, src.treatment_date_key,
        src.paid_date_key, src.feedback_date_key, src.cancelled_date_key,
        src.hours_confirm_to_visit, src.hours_checkin_to_start, src.hours_treat_to_pay,
        src.hours_book_to_pay,
        src.reached_confirmed, src.reached_checkin, src.reached_treatment, src.reached_payment,
        src.reached_feedback, src.is_cancelled, src.is_no_show, src.net_amount, @run_id, @now);

    COMMIT;
END
```

**Ba điểm khác biệt so với transaction fact:**

| # | Điểm | Lý do |
|---|---|---|
| 1 | Dùng `MERGE`, không delete-insert | Một booking có thể nhận mốc mới nhiều tháng sau ngày tạo. Delete-insert theo ngày sẽ không tìm thấy dòng cần cập nhật |
| 2 | Chỉ xử lý booking **có biến động** trong ngày | Quét lại toàn bộ lịch sử mỗi đêm rất chậm và không cần thiết |
| 3 | Dùng **rowstore**, không columnstore | Columnstore rất kém với khối lượng `UPDATE` lớn |

> `booking_junk_sk` để `-1` trong procedure này là **có chủ đích**: các cờ junk gắn với dòng hoá đơn, không gắn với vòng đời booking. Muốn phân tích phễu theo kênh đặt lịch thì dùng `dim_booking_junk` qua `fact_booking_line`.

---

## 4. Periodic snapshot — `fact_customer_monthly_snapshot`

Chốt trạng thái cuối tháng. Giải bài toán semi-additive của số dư điểm.

```sql
CREATE OR ALTER PROCEDURE dm.usp_load_fact_customer_monthly_snapshot
    @year_month INT, @run_id UNIQUEIDENTIFIER
AS
BEGIN
    SET NOCOUNT, XACT_ABORT ON;

    DECLARE @month_end DATE = (SELECT MAX(full_date) FROM dm.dim_date WHERE year_month = @year_month);
    DECLARE @month_end_key INT = (SELECT date_key FROM dm.dim_date WHERE full_date = @month_end);
    DECLARE @churn_days INT = 90;   -- ngưỡng hiệu chỉnh, xem ghi chú

    BEGIN TRAN;
    DELETE FROM dm.fact_customer_monthly_snapshot WHERE year_month = @year_month;

    INSERT INTO dm.fact_customer_monthly_snapshot (
        year_month, month_end_date_key, customer_sk, salon_sk,
        point_balance, membership_tier, days_since_last_visit,
        visit_count_mtd, net_amount_mtd, is_active_customer, is_churned, _run_id)
    SELECT
        @year_month, @month_end_key, c.customer_sk,
        ISNULL(last_visit.salon_sk, -1),
        -- Semi-additive: số dư CUỐI KỲ, tính bằng tổng luỹ tiến toàn bộ lịch sử
        ISNULL(pt.point_balance, 0),
        c.membership_tier,
        ISNULL(DATEDIFF(DAY, last_visit.last_date, @month_end), 9999),
        ISNULL(mtd.visit_cnt, 0),
        ISNULL(mtd.net_amount, 0),
        CASE WHEN DATEDIFF(DAY, last_visit.last_date, @month_end) <= @churn_days THEN 1 ELSE 0 END,
        CASE WHEN last_visit.last_date IS NULL
                   OR DATEDIFF(DAY, last_visit.last_date, @month_end) > @churn_days
             THEN 1 ELSE 0 END,
        @run_id
    FROM   dm.dim_customer c
    -- Cả ba phép cộng dưới đây phải gom theo customer_id, KHÔNG theo customer_sk.
    -- SCD2 sinh một sk mới mỗi lần khách đổi hạng thẻ / thành phố / nhóm tuổi, và
    -- fact nạp lúc phiên bản cũ còn hiệu lực mang sk cũ. Lọc theo c.customer_sk sẽ
    -- bỏ hết lịch sử trước lần đổi gần nhất: point_balance mất điểm đã tích,
    -- last_visit thành NULL nên days_since_last_visit = 9999 và is_churned = 1
    -- cho khách vẫn đang hoạt động.
    -- Số dư điểm: cộng dồn TOÀN BỘ lịch sử tới hết tháng, qua mọi phiên bản của khách
    OUTER APPLY (SELECT SUM(l.point_delta) AS point_balance
                 FROM dm.fact_loyalty_txn l
                 JOIN dm.dim_customer cx ON cx.customer_sk = l.customer_sk
                                        AND cx.customer_id = c.customer_id
                 JOIN dm.dim_date d ON d.date_key = l.txn_date_key
                 WHERE d.full_date <= @month_end) pt
    -- Lượt đến gần nhất tính tới hết tháng
    OUTER APPLY (SELECT TOP (1) d.full_date AS last_date, f.salon_sk
                 FROM dm.fact_sales_line f
                 JOIN dm.dim_customer cx ON cx.customer_sk = f.customer_sk
                                        AND cx.customer_id = c.customer_id
                 JOIN dm.dim_date d ON d.date_key = f.service_date_key
                 WHERE d.full_date <= @month_end
                 ORDER BY d.full_date DESC) last_visit
    -- Phát sinh trong chính tháng đó
    OUTER APPLY (SELECT COUNT(DISTINCT f.invoice_no) AS visit_cnt, SUM(f.net_amount) AS net_amount
                 FROM dm.fact_sales_line f
                 JOIN dm.dim_customer cx ON cx.customer_sk = f.customer_sk
                                        AND cx.customer_id = c.customer_id
                 JOIN dm.dim_date d ON d.date_key = f.service_date_key
                 WHERE d.year_month = @year_month) mtd
    WHERE  c.is_current = 1 AND c.customer_sk <> -1;
    COMMIT;
END
```

> `@churn_days = 90` là **giá trị khởi tạo tạm**, phải thay bằng phân vị 80–90% của khoảng cách giữa hai lượt đến và rà lại mỗi 6 tháng. Câu SQL tính ngưỡng: [07-analytics/chi-tieu-va-bao-cao.md](../07-analytics/chi-tieu-va-bao-cao.md#1-từ-điển-chỉ-tiêu).

---

## 4b. Bảng cầu nối — `dm.bridge_sales_promotion`

Bảng này phá quan hệ nhiều-nhiều giữa dòng hoá đơn và khuyến mãi. Nguồn là [`crt.invoice_line_promotion`](../03-ddl/02-crt.md).

```sql
CREATE OR ALTER PROCEDURE dm.usp_load_bridge_sales_promotion
    @business_date DATE,
    @run_id        UNIQUEIDENTIFIER
AS
BEGIN
    SET NOCOUNT, XACT_ABORT ON;
    DECLARE @date_key INT =
        YEAR(@business_date)*10000 + MONTH(@business_date)*100 + DAY(@business_date);

    BEGIN TRAN;
    DELETE FROM dm.bridge_sales_promotion WHERE service_date_key = @date_key;

    -- Hệ số phân bổ tính theo tiền giảm giá của từng khuyến mãi trên dòng đó.
    -- Nếu POS không tách được tiền theo từng khuyến mãi, chia đều: 1 / số khuyến mãi.
    -- Cả hai cách đều cho tổng allocation_factor theo invoice_line_id bằng 1,
    -- là điều DQ-ALLOC-001 mức BLOCK kiểm.
    WITH src AS (
        SELECT ilp.invoice_line_id,
               ilp.promotion_id,
               ilp.discount_amount,
               l.discount_amount AS line_discount_amount,
               COUNT(*) OVER (PARTITION BY ilp.invoice_line_id)      AS promo_cnt,
               SUM(ilp.discount_amount)
                   OVER (PARTITION BY ilp.invoice_line_id)           AS promo_amt_sum
        FROM   crt.invoice_line_promotion ilp
        JOIN   crt.invoice_line l ON l.invoice_line_id = ilp.invoice_line_id
        JOIN   crt.invoice      i ON i.invoice_id      = l.invoice_id
        WHERE  CAST(i.service_at AS DATE) = @business_date
          AND  i.invoice_status <> 'void'
          AND  i._is_deleted = 0 AND l._is_deleted = 0 AND ilp._is_deleted = 0
    )
    INSERT INTO dm.bridge_sales_promotion
        (service_date_key, invoice_line_id, promotion_sk,
         allocation_factor, allocated_discount_amount, _run_id)
    SELECT @date_key,
           s.invoice_line_id,
           ISNULL(p.promotion_sk, -1),
           CASE WHEN s.promo_amt_sum > 0
                THEN s.discount_amount / s.promo_amt_sum
                ELSE 1.0 / s.promo_cnt END,
           CASE WHEN s.promo_amt_sum > 0
                THEN s.discount_amount
                ELSE s.line_discount_amount / s.promo_cnt END,
           @run_id
    FROM   src s
    LEFT JOIN dm.dim_promotion p ON p.promotion_id = s.promotion_id;
    COMMIT;
END
```

> Sau khi nối `fact_sales_line` với bảng này, số dòng bị nhân lên theo số khuyến mãi trên mỗi dòng hoá đơn. Chỉ được dùng `allocated_discount_amount`; dùng `net_amount` sẽ cộng trùng. Ràng buộc này ghi trong [03-ddl/05-svg-bi.md mục 1](../03-ddl/05-svg-bi.md).

---

## 5. Làm mới bảng tổng hợp `svg_bi`

Toàn bộ dùng delete-insert theo khoá grain. Chạy trong `dag_refresh_svg_bi` sau khi `dm` đã nạp xong.

```sql
CREATE OR ALTER PROCEDURE svg_bi.usp_refresh_agg_revenue_daily_salon
    @business_date DATE, @run_id UNIQUEIDENTIFIER
AS
BEGIN
    SET NOCOUNT, XACT_ABORT ON;
    DECLARE @date_key INT = YEAR(@business_date)*10000 + MONTH(@business_date)*100 + DAY(@business_date);

    BEGIN TRAN;
    DELETE FROM svg_bi.agg_revenue_daily_salon WHERE date_key = @date_key;

    INSERT INTO svg_bi.agg_revenue_daily_salon (
        date_key, salon_sk, gross_amount, discount_amount, net_amount, net_excl_tax_amount,
        cash_collected_amount, cogs_amount, gross_margin_amount,
        invoice_count, line_count, treatment_count, unique_customer_count, new_customer_count,
        appointment_count, no_show_count, busy_minutes, available_minutes,
        rating_sum, response_count, _run_id, _refreshed_at)
    SELECT s.date_key, s.salon_sk,
           s.gross_amount, s.discount_amount, s.net_amount, s.net_excl_tax_amount,
           ISNULL(p.cash, 0), s.cogs_amount, s.gross_margin_amount,
           s.invoice_cnt, s.line_cnt, ISNULL(t.treatment_cnt, 0),
           s.unique_customer_cnt, s.new_customer_cnt,
           ISNULL(a.appointment_cnt, 0), ISNULL(a.no_show_cnt, 0),
           ISNULL(t.busy_minutes, 0), ISNULL(t.available_minutes, 0),
           ISNULL(f.rating_sum, 0), ISNULL(f.response_cnt, 0),
           @run_id, SYSUTCDATETIME()
    FROM (
        SELECT service_date_key AS date_key, salon_sk,
               SUM(gross_amount) AS gross_amount, SUM(discount_amount) AS discount_amount,
               SUM(net_amount) AS net_amount, SUM(net_excl_tax_amount) AS net_excl_tax_amount,
               SUM(cogs_amount) AS cogs_amount, SUM(gross_margin_amount) AS gross_margin_amount,
               COUNT(DISTINCT invoice_no) AS invoice_cnt,
               SUM(line_count) AS line_cnt,
               COUNT(DISTINCT customer_sk) AS unique_customer_cnt,
               COUNT(DISTINCT CASE WHEN j.is_first_visit = 1 THEN f.customer_sk END) AS new_customer_cnt
        FROM   dm.fact_sales_line f
        JOIN   dm.dim_booking_junk j ON j.booking_junk_sk = f.booking_junk_sk
        WHERE  f.service_date_key = @date_key
        GROUP  BY service_date_key, salon_sk
    ) s
    LEFT JOIN (SELECT payment_date_key, salon_sk, SUM(net_cash_amount) AS cash
               FROM dm.fact_payment WHERE payment_date_key = @date_key
                 AND payment_status = 'completed'
               GROUP BY payment_date_key, salon_sk) p
           ON p.payment_date_key = s.date_key AND p.salon_sk = s.salon_sk
    LEFT JOIN (SELECT treatment_date_key, salon_sk, SUM(treatment_count) AS treatment_cnt,
                      SUM(busy_minutes) AS busy_minutes, SUM(available_minutes) AS available_minutes
               FROM dm.fact_treatment WHERE treatment_date_key = @date_key
               GROUP BY treatment_date_key, salon_sk) t
           ON t.treatment_date_key = s.date_key AND t.salon_sk = s.salon_sk
    LEFT JOIN (SELECT appointment_date_key, salon_sk, SUM(appointment_count) AS appointment_cnt,
                      SUM(CAST(is_no_show AS INT)) AS no_show_cnt
               FROM dm.fact_appointment WHERE appointment_date_key = @date_key
               GROUP BY appointment_date_key, salon_sk) a
           ON a.appointment_date_key = s.date_key AND a.salon_sk = s.salon_sk
    LEFT JOIN (SELECT feedback_date_key, salon_sk, SUM(rating) AS rating_sum,
                      SUM(response_count) AS response_cnt
               FROM dm.fact_feedback WHERE feedback_date_key = @date_key
               GROUP BY feedback_date_key, salon_sk) f
           ON f.feedback_date_key = s.date_key AND f.salon_sk = s.salon_sk;
    COMMIT;
END
```

> **Đây là ví dụ chuẩn của *drilling across*.** Năm fact khác grain được **tổng hợp riêng về cùng grain `ngày × salon`** rồi mới `LEFT JOIN`. Join trực tiếp năm fact với nhau sẽ nhân dòng và làm doanh thu sai gấp nhiều lần.

Năm bảng tổng hợp còn lại theo cùng nguyên tắc:

| Bảng | Grain đích | Nguồn | Chu kỳ |
|---|---|---|---|
| `agg_funnel_daily` | ngày × salon | `fact_booking_lifecycle`, `crt.service_view` | Hằng ngày |
| `agg_therapist_utilization_daily` | ngày × KTV | `fact_treatment`, `fact_sales_line`, `fact_feedback` | Hằng ngày |
| `agg_service_perf_monthly` | tháng × salon × dịch vụ | `fact_sales_line`, `fact_treatment`, `fact_feedback` | Hằng ngày, nạp lại tháng hiện tại |
| `agg_customer_360` | 1 khách | Nhiều fact | Hằng ngày, dựng lại toàn bộ |
| `agg_cohort_retention` | cohort × tháng thứ N | `fact_sales_line` | Hằng tuần |

---

## 6. Thứ tự chạy và kiểm chứng

```
dag_build_datamart (06:00)
├── 1. usp_load_dim_*              (13 dimension, song song được)
├── 2. DQ nhóm SCD                 → BLOCK nếu FAIL
├── 3. usp_load_fact_*             (7 transaction fact, song song được)
├── 4. usp_load_fact_ad_spend      (nạp lại 7 ngày)
├── 5. usp_load_fact_booking_lifecycle
├── 6. usp_load_fact_customer_monthly_snapshot  (chỉ chạy ngày đầu tháng)
└── 7. DQ-UNIQ-*, DQ-RECON-003     → rollback phân vùng nếu FAIL

dag_refresh_svg_bi (06:40)
├── 8. usp_refresh_agg_*           (6 bảng)
├── 9. Cập nhật dim_customer.rfm_segment
└── 10. DQ-RECON-004, DQ-FRESH-003
```

Bước 2 là **cổng chặn**: dimension sai thì không được nạp fact. Bước 7 và 10 chạy **sau** khi nạp nên khi FAIL thì hành động là rollback phân vùng vừa nạp, không phải chặn trước.

### Kiểm chứng idempotent

Bắt buộc chạy trong kiểm thử nghiệm thu:

```sql
DECLARE @d DATE = '2026-08-13';
EXEC dm.usp_load_fact_sales_line @d, NEWID();
SELECT SUM(net_amount) AS run1 INTO #r1 FROM dm.fact_sales_line WHERE service_date_key = 20260813;
EXEC dm.usp_load_fact_sales_line @d, NEWID();
EXEC dm.usp_load_fact_sales_line @d, NEWID();
SELECT SUM(net_amount) AS run3 INTO #r3 FROM dm.fact_sales_line WHERE service_date_key = 20260813;
-- Phải bằng nhau. Khác nhau nghĩa là procedure không idempotent.
SELECT r1.run1, r3.run3, CASE WHEN r1.run1 = r3.run3 THEN 'PASS' ELSE 'FAIL' END AS result
FROM #r1 r1 CROSS JOIN #r3 r3;
```
