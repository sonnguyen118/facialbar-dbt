# SEED — Nạp dữ liệu khởi tạo

Các script trong tài liệu này chạy **một lần duy nhất** khi dựng database, **trước khi nạp bản ghi fact đầu tiên**.

Bỏ sót bất kỳ script nào ở đây đều gây lỗi nạp hoặc mất dòng fact về sau.

| Thứ tự | Việc | Bắt buộc trước khi |
|---|---|---|
| 1 | Dòng Unknown `-1` cho **mọi** dimension | Nạp fact đầu tiên |
| 2 | `dim_date` — 4.018 dòng | Nạp fact đầu tiên |
| 3 | `dim_time` — 1.440 dòng | Nạp fact đầu tiên |
| 4 | `dim_booking_junk` — 80 tổ hợp | Nạp fact đầu tiên |
| 5 | `dim_payment_method`, `dim_membership_tier` — danh mục cố định | Nạp `fact_payment` |
| 6 | `ctl.watermark`, `ctl.dq_rule`, `ctl.code_mapping` | Lần chạy pipeline đầu tiên |
| 7 | `ctl.vn_holiday` + cập nhật cờ lễ/Tết vào `dim_date` | Báo cáo so sánh cùng kỳ |

---

## 1. Dòng Unknown member cho mọi dimension

Fact khai báo FK `NOT NULL` và dùng `-1` khi thiếu khoá. Thiếu dòng `-1` thì câu `INSERT` fact sẽ vi phạm FK và cả lô bị chặn.

```sql
-- dim_salon
SET IDENTITY_INSERT dm.dim_salon ON;
INSERT INTO dm.dim_salon (salon_sk, salon_id, salon_name, salon_code, city, district, address,
    region, capacity_beds, salon_size_band, open_date, close_date, is_active,
    valid_from, valid_to, is_current, row_hash, _run_id)
VALUES (-1, -1, N'(Không xác định)', 'UNKNOWN', N'(Không xác định)', N'(Không xác định)',
    N'(Không xác định)', 'UNKNOWN', 1, 'UNKNOWN', '1900-01-01', NULL, 0,
    '1900-01-01', '9999-12-31', 1, 0x00, '00000000-0000-0000-0000-000000000000');
SET IDENTITY_INSERT dm.dim_salon OFF;

-- dim_employee
SET IDENTITY_INSERT dm.dim_employee ON;
INSERT INTO dm.dim_employee (employee_sk, employee_id, employee_name, employee_code, role_name,
    skill_level, current_salon_sk, hire_date, terminate_date, tenure_band, is_active,
    valid_from, valid_to, is_current, row_hash, _run_id)
VALUES (-1, -1, N'(Không xác định)', 'UNKNOWN', 'other', 'Junior', -1,
    '1900-01-01', NULL, 'UNKNOWN', 0,
    '1900-01-01', '9999-12-31', 1, 0x00, '00000000-0000-0000-0000-000000000000');
SET IDENTITY_INSERT dm.dim_employee OFF;

-- dim_service
SET IDENTITY_INSERT dm.dim_service ON;
INSERT INTO dm.dim_service (service_sk, service_id, service_name, service_code, category_l1,
    category_l2, standard_duration_min, list_price_amount, price_band, is_signature, is_active,
    valid_from, valid_to, is_current, row_hash, _run_id)
VALUES (-1, -1, N'(Không xác định)', 'UNKNOWN', N'(Chưa phân loại)', N'(Chưa phân loại)',
    5, 0, 'UNKNOWN', 0, 0,
    '1900-01-01', '9999-12-31', 1, 0x00, '00000000-0000-0000-0000-000000000000');
SET IDENTITY_INSERT dm.dim_service OFF;

-- Các dimension SCD1: cùng một khuôn
SET IDENTITY_INSERT dm.dim_product ON;
INSERT INTO dm.dim_product (product_sk, product_id, product_name, category, brand, unit,
    is_retail, is_consumable)
VALUES (-1, -1, N'(Không xác định)', N'(Chưa phân loại)', N'(Không xác định)', 'N/A', 0, 0);
SET IDENTITY_INSERT dm.dim_product OFF;

SET IDENTITY_INSERT dm.dim_promotion ON;
INSERT INTO dm.dim_promotion (promotion_sk, promotion_id, promotion_name, promotion_type,
    discount_value, valid_from_date, valid_to_date)
VALUES (-1, -1, N'(Không áp dụng)', 'none', 0, '1900-01-01', '9999-12-31');
SET IDENTITY_INSERT dm.dim_promotion OFF;

SET IDENTITY_INSERT dm.dim_payment_method ON;
INSERT INTO dm.dim_payment_method (payment_method_sk, payment_method_code, payment_method_name,
    method_group, is_cash)
VALUES (-1, 'UNKNOWN', N'(Không xác định)', 'other', 0);
SET IDENTITY_INSERT dm.dim_payment_method OFF;

SET IDENTITY_INSERT dm.dim_room ON;
INSERT INTO dm.dim_room (room_sk, room_id, salon_sk, room_name, room_type)
VALUES (-1, -1, -1, N'(Không xác định)', 'single');
SET IDENTITY_INSERT dm.dim_room OFF;

SET IDENTITY_INSERT dm.dim_campaign ON;
INSERT INTO dm.dim_campaign (campaign_sk, campaign_id, campaign_name, platform, objective,
    start_date, end_date)
VALUES (-1, 'UNKNOWN', N'(Không gắn chiến dịch)', 'other', 'awareness', '1900-01-01', NULL);
SET IDENTITY_INSERT dm.dim_campaign OFF;

SET IDENTITY_INSERT dm.dim_booking_junk ON;
INSERT INTO dm.dim_booking_junk (booking_junk_sk, booking_channel, is_first_visit,
    is_promotion_applied, is_member, is_rescheduled)
VALUES (-1, 'unknown', 0, 0, 0, 0);
SET IDENTITY_INSERT dm.dim_booking_junk OFF;
```

`dim_customer` xem [03-ddl/03-dm-dimension.md](../03-ddl/03-dm-dimension.md). `dim_date` và `dim_time` dùng khoá tự nhiên nên không cần dòng `-1`; thay vào đó dùng `19000101` và `0`.

### Kiểm chứng bước 1

```sql
-- Phải trả về 0 dòng. Mỗi dimension thiếu dòng -1 sẽ hiện ra ở đây.
SELECT 'dim_salon'        AS dim UNION ALL SELECT 'dim_employee' UNION ALL
SELECT 'dim_service'      UNION ALL SELECT 'dim_product'  UNION ALL
SELECT 'dim_promotion'    UNION ALL SELECT 'dim_payment_method' UNION ALL
SELECT 'dim_room'         UNION ALL SELECT 'dim_campaign' UNION ALL
SELECT 'dim_booking_junk' UNION ALL SELECT 'dim_customer'
EXCEPT
SELECT 'dim_salon'        WHERE EXISTS (SELECT 1 FROM dm.dim_salon        WHERE salon_sk        = -1)
UNION ALL SELECT 'dim_employee'  WHERE EXISTS (SELECT 1 FROM dm.dim_employee  WHERE employee_sk  = -1)
UNION ALL SELECT 'dim_service'   WHERE EXISTS (SELECT 1 FROM dm.dim_service   WHERE service_sk   = -1)
UNION ALL SELECT 'dim_product'   WHERE EXISTS (SELECT 1 FROM dm.dim_product   WHERE product_sk   = -1)
UNION ALL SELECT 'dim_promotion' WHERE EXISTS (SELECT 1 FROM dm.dim_promotion WHERE promotion_sk = -1)
UNION ALL SELECT 'dim_payment_method' WHERE EXISTS (SELECT 1 FROM dm.dim_payment_method WHERE payment_method_sk = -1)
UNION ALL SELECT 'dim_room'      WHERE EXISTS (SELECT 1 FROM dm.dim_room      WHERE room_sk      = -1)
UNION ALL SELECT 'dim_campaign'  WHERE EXISTS (SELECT 1 FROM dm.dim_campaign  WHERE campaign_sk  = -1)
UNION ALL SELECT 'dim_booking_junk' WHERE EXISTS (SELECT 1 FROM dm.dim_booking_junk WHERE booking_junk_sk = -1)
UNION ALL SELECT 'dim_customer'  WHERE EXISTS (SELECT 1 FROM dm.dim_customer  WHERE customer_sk  = -1);
```

---

## 2. `dim_date`

Script đầy đủ ở [03-ddl/03-dm-dimension.md](../03-ddl/03-dm-dimension.md). Dải nạp: `2022-01-01` đến `2032-12-31` = 4.018 dòng.

Bổ sung dòng Unknown cho ngày không xác định:

```sql
INSERT INTO dm.dim_date (date_key, full_date, day_of_month, day_of_week_iso, day_name_vi,
    week_of_year_iso, month_number, month_name_vi, quarter_number, year_number,
    year_month, year_quarter, is_weekend, is_month_end, is_vn_holiday, is_tet_season, holiday_name_vi)
VALUES (19000101, '1900-01-01', 1, 1, N'(Không xác định)', 1, 1, N'(Không xác định)', 1, 1900,
        190001, 19001, 0, 0, 0, 0, NULL);
```

---

## 3. `dim_time` — 1.440 dòng

```sql
;WITH m AS (
    SELECT 0 AS n
    UNION ALL SELECT n + 1 FROM m WHERE n < 1439
)
INSERT INTO dm.dim_time (time_key, time_value, hour_24, minute_of_hour,
                         slot_15min, time_band_vi, is_peak_hour)
SELECT
    n,
    CAST(DATEADD(MINUTE, n, CAST('00:00:00' AS TIME(0))) AS TIME(0)),
    n / 60,
    n % 60,
    -- Slot 15 phút khớp lịch đặt của salon: '14:00-14:15'
    CONCAT(FORMAT((n / 15 * 15) / 60, '00'), ':', FORMAT((n / 15 * 15) % 60, '00'), '-',
           FORMAT((n / 15 * 15 + 15) / 60 % 24, '00'), ':', FORMAT((n / 15 * 15 + 15) % 60, '00')),
    CASE WHEN n / 60 <  11 THEN N'Sáng'
         WHEN n / 60 <  14 THEN N'Trưa'
         WHEN n / 60 <  18 THEN N'Chiều'
         ELSE                   N'Tối' END,
    -- Cao điểm 17:00-20:00
    CASE WHEN n / 60 BETWEEN 17 AND 19 THEN 1 ELSE 0 END
FROM m
OPTION (MAXRECURSION 0);
```

Kiểm chứng: `SELECT COUNT(*) FROM dm.dim_time;` phải bằng **1440**.

---

## 4. `dim_booking_junk` — sinh toàn bộ tổ hợp

5 kênh × 2⁴ cờ = **80 dòng**. Sinh sẵn một lần để ETL chỉ việc tra khoá, không phải `INSERT` khi gặp tổ hợp mới.

```sql
-- Thân CTE trên SQL Server bắt buộc là SELECT; `VALUES` chỉ dùng được làm
-- derived table trong FROM. Viết `WITH x(c) AS (VALUES ...)` sẽ báo Msg 156.
;WITH ch(booking_channel) AS (
    SELECT booking_channel
    FROM   (VALUES ('app'), ('web'), ('hotline'), ('walk_in'), ('unknown')) AS t(booking_channel)
), b(v) AS (
    SELECT v FROM (VALUES (0), (1)) AS t(v)
)
INSERT INTO dm.dim_booking_junk
    (booking_channel, is_first_visit, is_promotion_applied, is_member, is_rescheduled)
SELECT ch.booking_channel, f.v, p.v, m.v, r.v
FROM   ch
CROSS JOIN b f
CROSS JOIN b p
CROSS JOIN b m
CROSS JOIN b r
WHERE NOT EXISTS (
    SELECT 1 FROM dm.dim_booking_junk j
    WHERE j.booking_channel      = ch.booking_channel
      AND j.is_first_visit       = f.v
      AND j.is_promotion_applied = p.v
      AND j.is_member            = m.v
      AND j.is_rescheduled       = r.v
);
```

Kiểm chứng: `SELECT COUNT(*) FROM dm.dim_booking_junk;` phải bằng **80**: bộ sinh tạo 79 tổ hợp, còn tổ hợp `('unknown',0,0,0,0)` đã là dòng `-1` nên bị `WHERE NOT EXISTS` bỏ qua.

Cách tra khoá trong ETL:

```sql
LEFT JOIN dm.dim_booking_junk j
       ON j.booking_channel      = ISNULL(src.booking_channel, 'unknown')
      AND j.is_first_visit       = src.is_first_visit
      AND j.is_promotion_applied = src.is_promotion_applied
      AND j.is_member            = src.is_member
      AND j.is_rescheduled       = src.is_rescheduled
```

---

## 5. Danh mục cố định

```sql
INSERT INTO dm.dim_payment_method (payment_method_code, payment_method_name, method_group, is_cash)
VALUES ('CASH',    N'Tiền mặt',        'cash',    1),
       ('CARD',    N'Thẻ ngân hàng',   'card',    0),
       ('QR',      N'QR chuyển khoản', 'digital', 0),
       ('EWALLET', N'Ví điện tử',      'digital', 0),
       ('VOUCHER', N'Voucher',         'voucher', 0),
       ('POINT',   N'Điểm thưởng',     'voucher', 0);

-- dim_membership_tier dùng khoá tự nhiên `tier_code` làm khoá chính, KHÔNG có
-- khoá đại diện, vì đây là bảng tham chiếu quy tắc cho ETL chứ không phải chiều
-- phân tích (xem 01-erd/bus-matrix.md). Dòng "không xác định" vì vậy mang mã
-- 'UNKNOWN' chứ không phải sk = -1, và DQ-SCD-003 nêu bảng này là ngoại lệ.
INSERT INTO dm.dim_membership_tier
    (tier_code, tier_name, tier_rank, min_spend_amount, discount_pct, point_multiplier)
VALUES ('UNKNOWN',  N'(Không xác định)',   0,          0, 0.0000, 1.0000);

INSERT INTO dm.dim_membership_tier
    (tier_code, tier_name, tier_rank, min_spend_amount, discount_pct, point_multiplier)
VALUES ('None',     N'Chưa là thành viên', 0,          0, 0.0000, 1.0000),
       ('Silver',   N'Bạc',                1,  5000000, 0.0500, 1.0000),
       ('Gold',     N'Vàng',               2, 20000000, 0.1000, 1.5000),
       ('Platinum', N'Bạch kim',           3, 50000000, 0.1500, 2.0000);
```

> `min_spend_amount`, `discount_pct`, `point_multiplier` là **quy tắc kinh doanh**, cần CRM xác nhận trước khi dùng. Giá trị trên là giả định để dựng hệ thống.

---

## 6. Bảng cấu hình `ctl`

`ctl.watermark` và `ctl.dq_rule`: xem [DDL ctl](../03-ddl/06-ctl-qtn.md) và [catalog DQ rule](../05-quality/dq-rules.md).

`ctl.code_mapping` — nạp từ [bảng ánh xạ trong STM](../02-mapping/source-to-target.md#ánh-xạ-danh-mục):

```sql
INSERT INTO ctl.code_mapping (mapping_group, source_value, target_value)
VALUES ('acquisition_channel', 'fb',         'fb_ads'),
       ('acquisition_channel', 'facebook',   'fb_ads'),
       ('acquisition_channel', 'fb_ads',     'fb_ads'),
       ('acquisition_channel', 'meta',       'fb_ads'),
       ('acquisition_channel', 'gg',         'google_ads'),
       ('acquisition_channel', 'google',     'google_ads'),
       ('acquisition_channel', 'google_ads', 'google_ads'),
       ('acquisition_channel', 'adwords',    'google_ads'),
       ('acquisition_channel', 'walkin',     'walk_in'),
       ('acquisition_channel', 'walk-in',    'walk_in'),
       ('acquisition_channel', 'offline',    'walk_in'),
       ('acquisition_channel', 'store',      'walk_in'),
       ('acquisition_channel', 'ref',        'referral'),
       ('acquisition_channel', 'referral',   'referral'),
       ('acquisition_channel', 'friend',     'referral'),
       ('acquisition_channel', 'zalo',       'zalo'),
       ('acquisition_channel', 'zalo_oa',    'zalo'),
       ('booking_status', 'new',         'created'),
       ('booking_status', 'created',     'created'),
       ('booking_status', 'pending',     'created'),
       ('booking_status', 'confirmed',   'confirmed'),
       ('booking_status', 'accepted',    'confirmed'),
       ('booking_status', 'cancelled',   'cancelled'),
       ('booking_status', 'canceled',    'cancelled'),
       ('booking_status', 'void',        'cancelled'),
       ('booking_status', 'done',        'completed'),
       ('booking_status', 'completed',   'completed'),
       ('booking_status', 'finished',    'completed'),
       ('booking_status', 'moved',       'rescheduled'),
       ('booking_status', 'rescheduled', 'rescheduled'),
       ('booking_status', 'changed',     'rescheduled');
```

Giá trị nguồn không khớp bảng này được ánh xạ về `UNKNOWN` và ghi cảnh báo `DQ-MAP-001` — nhờ đó phát hiện được khi nguồn thêm giá trị mới.

---

## 7. Ngày lễ và cờ Tết

Nạp `ctl.vn_holiday` mỗi năm một lần theo công bố của Chính phủ, rồi cập nhật cờ vào `dim_date`:

```sql
-- Ví dụ dữ liệu năm 2026
INSERT INTO ctl.vn_holiday (holiday_date, holiday_name, is_tet, is_observed)
VALUES ('2026-01-01', N'Tết Dương lịch',      0, 0),
       ('2026-02-17', N'Tết Nguyên đán',      1, 0),
       ('2026-02-18', N'Tết Nguyên đán',      1, 0),
       ('2026-02-19', N'Tết Nguyên đán',      1, 0),
       ('2026-04-26', N'Giỗ tổ Hùng Vương',   0, 0),
       ('2026-04-30', N'Giải phóng miền Nam', 0, 0),
       ('2026-05-01', N'Quốc tế Lao động',    0, 0),
       ('2026-09-02', N'Quốc khánh',          0, 0);

UPDATE d SET d.is_vn_holiday = 1, d.holiday_name_vi = h.holiday_name
FROM dm.dim_date d JOIN ctl.vn_holiday h ON h.holiday_date = d.full_date;

-- Cao điểm ngành spa: 21 ngày trước ngày đầu Tết
UPDATE d SET d.is_tet_season = 1
FROM dm.dim_date d
WHERE EXISTS (SELECT 1 FROM ctl.vn_holiday h
              WHERE h.is_tet = 1
                AND d.full_date BETWEEN DATEADD(DAY, -21, h.holiday_date) AND h.holiday_date);
```

> Ngày trong bảng trên là **giả định để dựng hệ thống**, cần Hành chính xác nhận theo công bố thực tế từng năm. Thiếu cờ này thì so sánh cùng kỳ tháng 1–2 sai lệch nặng và mô hình dự báo nhu cầu không giải thích được cú tăng trước Tết.

---

## Kịch bản chạy toàn bộ

```
01_create_database_and_schema.sql      -- docs/03-ddl/00-init.md
02_create_partition_scheme.sql         -- docs/03-ddl/00-init.md mục 5
03_create_crt.sql                      -- docs/03-ddl/02-crt.md, theo đúng thứ tự tạo bảng
04_create_lnd.sql                      -- sinh tự động, docs/03-ddl/01-lnd.md
05_create_ctl_qtn.sql                  -- docs/03-ddl/06-ctl-qtn.md
06_create_dm_dimension.sql             -- docs/03-ddl/03-dm-dimension.md
07_create_dm_fact.sql                  -- docs/03-ddl/04-dm-fact.md
08_create_svg_bi.sql                   -- docs/03-ddl/05-svg-bi.md
09_seed_unknown_members.sql            -- mục 1 tài liệu này
10_seed_dim_date.sql                   -- mục 2
11_seed_dim_time.sql                   -- mục 3
12_seed_dim_booking_junk.sql           -- mục 4
13_seed_reference_data.sql             -- mục 5
14_seed_ctl_config.sql                 -- mục 6
15_seed_vn_holiday.sql                 -- mục 7
16_create_procedures.sql               -- docs/04-etl/
```

Bước 9 đến 15 **không được đảo thứ tự** và phải hoàn tất trước khi chạy pipeline lần đầu.
