# CATALOG QUY TẮC KIỂM SOÁT CHẤT LƯỢNG DỮ LIỆU

Danh mục đầy đủ các quy tắc chạy tại cổng kiểm tra chất lượng. Nội dung bảng này được nạp vào [`ctl.dq_rule`](../03-ddl/06-ctl-qtn.md#ctldq_rule) để sửa ngưỡng không cần đổi code.

## Quy ước

**Mã quy tắc:** `DQ-<NHÓM>-<số>`

| Nhóm | Phạm vi |
|---|---|
`CUS` khách hàng · `BKG` đặt lịch · `APT` lịch hẹn · `TRT` điều trị · `INV` hoá đơn · `PAY` thanh toán · `LOY` điểm thưởng · `MEM` thành viên · `FBK` đánh giá · `ADS` quảng cáo · `UNIQ` duy nhất · `SCD` chiều lịch sử · `ALLOC` phân bổ · `RECON` đối soát · `FRESH` kịp thời · `MAP` ánh xạ danh mục

**Mức xử lý:**

| Mức | Hành vi | Thông báo |
|---|---|---|
| `BLOCK` | Dừng nhánh, dòng lỗi sang `qtn.reject_row`, không nạp vào `dm` | P1 — gọi điện + Slack |
| `WARN` | Vẫn nạp, gắn dấu hiệu trên báo cáo | P2 — Slack cho người phụ trách |
| `INFO` | Chỉ ghi `ctl.dq_result` | P3 — email tổng hợp hằng ngày |

**Nguyên tắc cổng:** chỉ **dừng nhánh** bị lỗi, không dừng toàn hệ thống. `fact_payment` bị chặn thì `fact_feedback` vẫn nạp bình thường.

---

## 1. Completeness — Đầy đủ

| Mã | Đối tượng | Kiểm tra | Ngưỡng | Mức | Chủ |
|---|---|---|---|---|---|
| `DQ-COM-001` | `crt.invoice` | Mọi salon đang hoạt động phải có ≥ 1 hoá đơn mỗi ngày | 0 salon thiếu | `BLOCK` | Vận hành |
| `DQ-COM-002` | `crt.invoice_line` | Mọi hoá đơn phải có ≥ 1 dòng | 0 hoá đơn rỗng | `BLOCK` | Vận hành |
| `DQ-COM-003` | `crt.payment` | Hoá đơn `status='paid'` phải có ≥ 1 thanh toán | 0 vi phạm | `BLOCK` | Tài chính |
| `DQ-COM-004` | `crt.treatment` | Lịch hẹn đã check-in phải có ≥ 1 điều trị | ≤ 2% | `WARN` | Vận hành |
| `DQ-COM-005` | `crt.customer` | `customer_id` không NULL | 0 vi phạm | `BLOCK` | CRM |
| `DQ-COM-006` | `crt.treatment` | `available_minutes > 0` (cần lịch làm việc KTV) | ≤ 5% | `WARN` | Vận hành |

```sql
-- DQ-COM-001: salon đang hoạt động mà không có hoá đơn trong ngày
SELECT s.salon_id, s.salon_name
FROM   crt.salon s
WHERE  s.is_active = 1
  AND  s.open_date <= @business_date
  AND  NOT EXISTS (SELECT 1 FROM crt.invoice i
                   WHERE i.salon_id = s.salon_id
                     AND CAST(i.service_at AS DATE) = @business_date
                     AND i._is_deleted = 0);

-- DQ-COM-002: hoá đơn không có dòng nào
SELECT i.invoice_id, i.invoice_no
FROM   crt.invoice i
WHERE  CAST(i.service_at AS DATE) = @business_date
  AND  i.invoice_status <> 'void'
  AND  NOT EXISTS (SELECT 1 FROM crt.invoice_line l
                   WHERE l.invoice_id = i.invoice_id AND l._is_deleted = 0);
```

---

## 2. Accuracy — Chính xác

| Mã | Đối tượng | Kiểm tra | Ngưỡng | Mức | Chủ |
|---|---|---|---|---|---|
| `DQ-INV-005` | `crt.invoice_line` | `gross_amount = quantity × unit_price_amount` | 0 vi phạm | `BLOCK` | Tài chính |
| `DQ-INV-006` | `crt.invoice_line` | `discount_amount = promo + member` | 0 vi phạm | `BLOCK` | Tài chính |
| `DQ-INV-007` | `crt.invoice_line` | `net_amount = gross − discount` | 0 vi phạm | `BLOCK` | Tài chính |
| `DQ-INV-008` | `crt.invoice_line` | `cogs_amount ≤ net_amount` (biên lãi không âm bất thường) | ≤ 1% | `WARN` | Kế toán |
| `DQ-INV-009` | `crt.invoice_line` | `discount_amount ≤ gross_amount` | 0 vi phạm | `BLOCK` | Tài chính |
| `DQ-TRT-001` | `crt.treatment` | `busy_minutes` trong khoảng 5–480 | 0 vi phạm | `BLOCK` | Vận hành |
| `DQ-TRT-002` | `crt.treatment` | `completed_at ≥ started_at` | 0 vi phạm | `BLOCK` | Vận hành |
| `DQ-FBK-001` | `crt.feedback` | `rating` trong 1–5 | 0 vi phạm | `BLOCK` | CX |
| `DQ-FBK-002` | `crt.feedback` | `nps_score` NULL hoặc trong 0–10 | 0 vi phạm | `BLOCK` | CX |
| `DQ-PAY-002` | `crt.payment` | `payment_amount ≥ 0` | 0 vi phạm | `BLOCK` | Tài chính |
| `DQ-BKG-004` | `crt.booking` | `booked_at ≤ thời điểm hiện tại` | 0 vi phạm | `BLOCK` | Vận hành |
| `DQ-BKG-006` | `crt.booking` | `requested_slot_at ≥ booked_at` | ≤ 0,5% | `WARN` | Vận hành |
| `DQ-ADS-001` | `crt.ad_spend` | `click_count ≤ impression_count` | 0 vi phạm | `BLOCK` | Marketing |

```sql
-- DQ-INV-007: net_amount không khớp công thức
SELECT l.invoice_line_id, l.gross_amount, l.discount_amount, l.net_amount
FROM   crt.invoice_line l
JOIN   crt.invoice i ON i.invoice_id = l.invoice_id
WHERE  CAST(i.service_at AS DATE) = @business_date
  AND  ABS(l.net_amount - (l.gross_amount - l.discount_amount)) >= 0.01;
```

---

## 3. Consistency — Nhất quán

Nhóm quan trọng nhất: đây là các quy tắc bảo vệ tiêu chí nghiệm thu *doanh thu khớp POS ±0,1%*.

| Mã | Đối tượng | Kiểm tra | Ngưỡng | Mức | Chủ |
|---|---|---|---|---|---|
| `DQ-RECON-001` | `crt.invoice_line` ↔ POS | Doanh thu thuần theo ngày × salon | lệch ≤ 0,1% | `BLOCK` | Tài chính |
| `DQ-RECON-002` | `crt.payment` ↔ cổng thanh toán | Tiền thực thu theo ngày | lệch ≤ 0,1% | `BLOCK` | Tài chính |
| `DQ-RECON-003` | `dm.fact_sales_line` ↔ `crt.invoice_line` | Doanh thu theo ngày × salon | lệch = 0 | `BLOCK` | Dữ liệu |
| `DQ-RECON-004` | `svg_bi.agg_revenue_daily_salon` ↔ `dm.fact_sales_line` | Doanh thu theo ngày × salon | lệch = 0 | `BLOCK` | Dữ liệu |
| `DQ-PAY-004` | `crt.payment_allocation` | Tổng phân bổ = `payment_amount` | lệch ≤ 1 đồng | `BLOCK` | Tài chính |
| `DQ-PAY-005` | `crt.payment` ↔ `crt.invoice` | Tổng thanh toán ≤ tổng hoá đơn | ≤ 0,5% | `WARN` | Tài chính |

```sql
-- DQ-RECON-001: đối soát crt với POS. FULL OUTER JOIN là cố ý —
-- bắt được cả trường hợp DWH có mà POS không có (nạp trùng) và ngược lại (mất dữ liệu).
WITH dwh AS (
    SELECT i.salon_id, CAST(i.service_at AS DATE) AS d, SUM(l.net_amount) AS amt
    FROM   crt.invoice_line l JOIN crt.invoice i ON i.invoice_id = l.invoice_id
    WHERE  i._is_deleted = 0 AND l._is_deleted = 0 AND i.invoice_status <> 'void'
    GROUP  BY i.salon_id, CAST(i.service_at AS DATE)
),
pos AS (
    SELECT store_id AS salon_id, service_date AS d, SUM(net_amount) AS amt
    FROM   lnd.pos_revenue_control          -- bảng số liệu đối chiếu do POS cung cấp
    GROUP  BY store_id, service_date
)
SELECT COALESCE(dwh.salon_id, pos.salon_id) AS salon_id,
       COALESCE(dwh.d, pos.d)               AS business_date,
       ISNULL(dwh.amt, 0)                   AS dwh_amount,
       ISNULL(pos.amt, 0)                   AS pos_amount,
       ISNULL(dwh.amt,0) - ISNULL(pos.amt,0) AS diff,
       ABS(ISNULL(dwh.amt,0) - ISNULL(pos.amt,0)) / NULLIF(pos.amt, 0) AS diff_pct
FROM       dwh
FULL OUTER JOIN pos ON pos.salon_id = dwh.salon_id AND pos.d = dwh.d
WHERE  ABS(ISNULL(dwh.amt,0) - ISNULL(pos.amt,0)) > 1000        -- ngưỡng làm tròn
   OR  ABS(ISNULL(dwh.amt,0) - ISNULL(pos.amt,0)) / NULLIF(pos.amt,0) > 0.001;

-- DQ-PAY-004: tổng phân bổ phải bằng số tiền đã trả
SELECT p.payment_id, p.payment_amount, SUM(a.allocated_amount) AS allocated
FROM   crt.payment p
JOIN   crt.payment_allocation a ON a.payment_id = p.payment_id
WHERE  CAST(p.paid_at AS DATE) = @business_date
GROUP  BY p.payment_id, p.payment_amount
HAVING ABS(p.payment_amount - SUM(a.allocated_amount)) > 1;
```

> `DQ-RECON-001` cần POS cung cấp **bảng số liệu đối chiếu** (`lnd.pos_revenue_control`) — tổng doanh thu theo ngày × salon do chính POS tính. Không có bảng này thì việc đối soát chỉ là so `crt` với `lnd`, tức là so dữ liệu với chính nó, không có giá trị kiểm soát. **Cần đưa vào yêu cầu với nhà cung cấp POS.**

---

## 4. Uniqueness — Duy nhất

| Mã | Đối tượng | Kiểm tra | Ngưỡng | Mức | Chủ |
|---|---|---|---|---|---|
| `DQ-UNIQ-001` | `dm.fact_sales_line` | `invoice_line_id` duy nhất toàn cục | 0 trùng | `BLOCK` | Dữ liệu |
| `DQ-UNIQ-002` | `dm.fact_payment` | `payment_id` duy nhất toàn cục | 0 trùng | `BLOCK` | Dữ liệu |
| `DQ-UNIQ-003` | `dm.fact_treatment` | `treatment_id` duy nhất toàn cục | 0 trùng | `BLOCK` | Dữ liệu |
| `DQ-UNIQ-004` | `dm.fact_booking_line` | `booking_item_id` duy nhất toàn cục | 0 trùng | `BLOCK` | Dữ liệu |
| `DQ-UNIQ-005` | `dm.fact_appointment` | `appointment_id` duy nhất toàn cục | 0 trùng | `BLOCK` | Dữ liệu |
| `DQ-APT-001` | `crt.appointment` | Một KTV không có 2 lịch hẹn chồng giờ | 0 vi phạm | `BLOCK` | Vận hành |
| `DQ-APT-002` | `crt.appointment` | Một buồng không có 2 lịch hẹn chồng giờ | 0 vi phạm | `BLOCK` | Vận hành |
| `DQ-MEM-001` | `crt.membership_subscription` | Kỳ thành viên của cùng khách không chồng nhau | 0 vi phạm | `BLOCK` | CRM |
| `DQ-CUS-006` | `crt.customer_identity_map` | Một `source_id` chỉ trỏ tới một `customer_id` | 0 vi phạm | `BLOCK` | CRM |

```sql
-- DQ-UNIQ-001: duy nhất toàn cục. Cần vì UNIQUE index trên bảng phân vùng
-- chỉ đảm bảo duy nhất trong cùng service_date_key (xem docs/03-ddl/00-init.md mục 4).
SELECT invoice_line_id, COUNT(*) AS dup_cnt
FROM   dm.fact_sales_line
GROUP  BY invoice_line_id
HAVING COUNT(*) > 1;

-- DQ-APT-001: KTV bị xếp 2 lịch chồng giờ
SELECT a1.appointment_id, a2.appointment_id AS conflict_with, a1.employee_id
FROM   crt.appointment a1
JOIN   crt.appointment a2
       ON  a2.employee_id = a1.employee_id
       AND a2.appointment_id > a1.appointment_id
       AND a2.slot_at < DATEADD(MINUTE, a1.scheduled_duration_min, a1.slot_at)
       AND a1.slot_at < DATEADD(MINUTE, a2.scheduled_duration_min, a2.slot_at)
WHERE  a1.appointment_status NOT IN ('cancelled','no_show')
  AND  a2.appointment_status NOT IN ('cancelled','no_show')
  AND  CAST(a1.slot_at AS DATE) = @business_date;

-- DQ-MEM-001: kỳ thành viên chồng nhau
SELECT m1.customer_id, m1.subscription_id, m2.subscription_id AS overlap_with
FROM   crt.membership_subscription m1
JOIN   crt.membership_subscription m2
       ON  m2.customer_id = m1.customer_id
       AND m2.subscription_id > m1.subscription_id
       AND m1.valid_from <= m2.valid_to
       AND m2.valid_from <= m1.valid_to
WHERE  m1._is_deleted = 0 AND m2._is_deleted = 0;
```

---

## 5. Validity — Hợp lệ

| Mã | Đối tượng | Kiểm tra | Ngưỡng | Mức | Chủ |
|---|---|---|---|---|---|
| `DQ-CUS-002` | `crt.customer` | `phone` khớp định dạng E.164 `+84…` | ≤ 2% | `WARN` | CRM |
| `DQ-CUS-003` | `crt.customer` | `email` có `@` và tên miền hợp lệ | ≤ 5% | `WARN` | CRM |
| `DQ-CUS-004` | `crt.customer` | `date_of_birth` trong 1920 → hôm nay | 0 vi phạm | `BLOCK` | CRM |
| `DQ-CUS-007` | `crt.customer_identity_map` | `match_confidence < 0,80` phải có `reviewed_by` | 0 vi phạm | `BLOCK` | CRM |
| `DQ-BKG-003` | `crt.booking` | `booking_status` thuộc danh mục | 0 vi phạm | `BLOCK` | Vận hành |
| `DQ-INV-004` | `crt.invoice_line` | `line_type` là `service` hoặc `product`, và có đúng một trong hai ID | 0 vi phạm | `BLOCK` | Vận hành |
| `DQ-MAP-001` | `ctl.code_mapping` | Không có giá trị nguồn nào bị ánh xạ về `UNKNOWN` | ≤ 1% | `WARN` | Dữ liệu |
| `DQ-FK-001` | `dm.fact_*` | Tỷ lệ dòng fact trỏ `sk = -1` | ≤ 1% | `WARN` | Dữ liệu |
| `DQ-FK-002` | `dm.fact_sales_line` | Tỷ lệ dòng trỏ `customer_sk = -1` | ≤ 5% | `WARN` | CRM |

```sql
-- DQ-CUS-007: gộp định danh độ tin cậy thấp mà chưa ai rà
SELECT identity_id, source_system, source_id, customer_id, match_confidence
FROM   crt.customer_identity_map
WHERE  match_confidence < 0.80 AND reviewed_by IS NULL;

-- DQ-FK-001: đo tỷ lệ khoá thiếu trên toàn bộ fact chính
SELECT 'fact_sales_line' AS entity,
       CAST(SUM(CASE WHEN customer_sk = -1 THEN 1 ELSE 0 END) AS DECIMAL(18,4))
         / NULLIF(COUNT(*), 0) AS unknown_pct
FROM   dm.fact_sales_line
WHERE  service_date_key = @date_key;

-- DQ-MAP-001: nguồn xuất hiện giá trị mới chưa có trong bảng ánh xạ
SELECT DISTINCT c.acquisition_channel
FROM   crt.customer c
WHERE  c.acquisition_channel = 'UNKNOWN'
  AND  CAST(c.registration_date AS DATE) = @business_date;
```

> `DQ-MAP-001` là quy tắc phát hiện nguồn **thay đổi danh mục** mà không thông báo. Đây là lớp phòng cho rủi ro R1 (nhà cung cấp POS đổi cấu trúc dữ liệu).

---

## 6. Freshness — Kịp thời

| Mã | Đối tượng | Kiểm tra | Ngưỡng | Mức | Chủ |
|---|---|---|---|---|---|
| `DQ-FRESH-001` | `crt.invoice` | Dữ liệu ngày N có trước 06:00 ngày N+1 | 0 trễ | `BLOCK` | Dữ liệu |
| `DQ-FRESH-002` | `crt.payment` | Như trên | 0 trễ | `BLOCK` | Dữ liệu |
| `DQ-FRESH-003` | `svg_bi.agg_revenue_daily_salon` | Làm mới xong trước 08:00 | 0 trễ | `BLOCK` | Dữ liệu |
| `DQ-FRESH-004` | `crt.ad_spend` | Dữ liệu 7 ngày gần nhất được nạp lại mỗi lần chạy | 0 thiếu | `WARN` | Marketing |
| `DQ-FRESH-005` | Kafka | Consumer lag của `cg-s3-sink` | ≤ 100.000 message | `WARN` | Dữ liệu |
| `DQ-FRESH-006` | `crt.feedback` | Độ trễ từ `occurred_at` tới `_loaded_at` | ≤ 24 giờ | `INFO` | CX |

```sql
-- DQ-FRESH-001: kiểm tra watermark đã tiến tới ngày nghiệp vụ chưa
SELECT source_name, entity_name, last_value, updated_at
FROM   ctl.watermark
WHERE  source_name = 'pos'
  AND  entity_name IN ('invoice','invoice_line','payment')
  AND  TRY_CAST(last_value AS DATE) < @business_date;
```

---

## 7. Quy tắc cho mô hình chiều

| Mã | Đối tượng | Kiểm tra | Ngưỡng | Mức | Chủ |
|---|---|---|---|---|---|
| `DQ-SCD-001` | Mọi `dm.dim_*` SCD2 | Lịch sử liền mạch — không hở, không chồng khoảng | 0 vi phạm | `BLOCK` | Dữ liệu |
| `DQ-SCD-002` | Mọi `dm.dim_*` SCD2 | Mỗi nghiệp vụ key có đúng 1 phiên bản `is_current = 1` | 0 vi phạm | `BLOCK` | Dữ liệu |
| `DQ-SCD-003` | 10 dim có khoá đại diện | Tồn tại dòng Unknown `sk = -1` | 0 thiếu | `BLOCK` | Dữ liệu |
| `DQ-SCD-004` | 3 dim dùng khoá tự nhiên | `dim_date` có `19000101`, `dim_time` có `0`, `dim_membership_tier` có `tier_code = 'UNKNOWN'` | 0 thiếu | `BLOCK` | Dữ liệu |
| `DQ-ALLOC-001` | `dm.bridge_sales_promotion` | Tổng `allocation_factor` theo dòng hoá đơn = 1 | 0 vi phạm | `BLOCK` | Dữ liệu |
| `DQ-ALLOC-002` | `crt.invoice_line_promotion` | Tổng `discount_amount` theo dòng hoá đơn khớp `invoice_line.discount_amount` | sai lệch ≤ 1 đồng | `BLOCK` | Dữ liệu |
| `DQ-DIM-001` | `dm.dim_booking_junk` | Đủ 80 dòng (79 tổ hợp sinh tự động + dòng Unknown `('unknown',0,0,0,0)`) | 0 thiếu | `BLOCK` | Dữ liệu |
| `DQ-DIM-002` | `dm.dim_date` | Có đủ ngày cho `business_date` đang nạp | 0 thiếu | `BLOCK` | Dữ liệu |
| `DQ-DIM-003` | `dm.dim_time` | Đủ 1.440 dòng | 0 thiếu | `BLOCK` | Dữ liệu |

```sql
-- DQ-SCD-001: khoảng hở hoặc chồng trong lịch sử SCD2
WITH v AS (
    SELECT customer_id, valid_from, valid_to,
           LEAD(valid_from) OVER (PARTITION BY customer_id ORDER BY valid_from) AS next_from
    FROM   dm.dim_customer WHERE customer_sk <> -1
)
SELECT * FROM v WHERE next_from IS NOT NULL AND next_from <> valid_to;

-- DQ-SCD-002: nhiều hơn một phiên bản hiện hành
SELECT customer_id, COUNT(*) AS current_cnt
FROM   dm.dim_customer WHERE is_current = 1
GROUP  BY customer_id HAVING COUNT(*) <> 1;

-- DQ-ALLOC-001: hệ số phân bổ không cộng về 1
SELECT invoice_line_id, SUM(allocation_factor) AS total_factor
FROM   dm.bridge_sales_promotion
GROUP  BY invoice_line_id
HAVING ABS(SUM(allocation_factor) - 1.0) > 0.000001;
```

> `DQ-SCD-001` và `DQ-SCD-002` không phải quy tắc phụ. Lỗi SCD2 làm **nhân đôi dòng fact** khi tra khoá theo thời điểm, biểu hiện ra ngoài là doanh thu của vài khách tự tăng gấp đôi — rất khó truy nếu không có hai quy tắc này.

---

## Tổng hợp

| Chiều | Số quy tắc | `BLOCK` | `WARN` | `INFO` |
|---|---|---|---|---|
| Completeness | 6 | 4 | 2 | 0 |
| Accuracy | 13 | 11 | 2 | 0 |
| Consistency | 6 | 5 | 1 | 0 |
| Uniqueness | 9 | 9 | 0 | 0 |
| Validity | 9 | 4 | 5 | 0 |
| Freshness | 6 | 3 | 2 | 1 |
| Mô hình chiều | 9 | 9 | 0 | 0 |
| **Tổng** | **58** | **45** | **12** | **1** |

## Phụ thuộc chưa có

| Cần | Chặn quy tắc | Bên cung cấp |
|---|---|---|
| Bảng số liệu đối chiếu doanh thu từ POS | `DQ-RECON-001` | Nhà cung cấp POS |
| Dữ liệu đối chiếu từ cổng thanh toán | `DQ-RECON-002` | Cổng thanh toán |
| Lịch làm việc / phân ca KTV | `DQ-COM-006` | Vận hành |
| Quy tắc tính COGS dịch vụ | `DQ-INV-008` | Kế toán |

## Kịch bản chạy

Cổng kiểm tra chạy trong `dag_dq_gate` (05:40), sau `dag_load_dwh` và trước `dag_build_datamart`.

```
1. Chạy toàn bộ quy tắc trên crt        → ghi ctl.dq_result
2. Đẩy dòng vi phạm sang qtn.reject_row → kèm rule_id và payload JSON
3. Nếu có BLOCK nào FAIL:
     - Dừng nhánh entity tương ứng, không nạp vào dm
     - Ghi ctl.pipeline_run.status = 'BLOCKED'
     - Cảnh báo P1
4. Sau dag_build_datamart: chạy nhóm quy tắc mô hình chiều và DQ-RECON-003/004
5. Sau dag_refresh_svg_bi: chạy DQ-FRESH-003
```

Quy tắc kiểm tra `dm` và `svg_bi` chạy **sau** khi nạp — không chặn được trước, nên khi FAIL thì hành động là **rollback phân vùng vừa nạp** thay vì chặn.
