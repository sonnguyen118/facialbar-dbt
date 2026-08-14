# ETL — Nạp Dimension

13 dimension. Thứ tự bắt buộc: **dimension trước, fact sau** — fact cần surrogate key do dimension sinh ra.

| Nhóm | Bảng | Cơ chế | Procedure |
|---|---|---|---|
| Nền tảng | `dim_date`, `dim_time` | Seed một lần | [seed.md](seed.md) |
| SCD Type 2 | `dim_customer`, `dim_salon`, `dim_employee`, `dim_service` | 4 bước đóng/mở phiên bản | `usp_load_dim_<tên>` |
| SCD Type 1 | `dim_product`, `dim_promotion`, `dim_payment_method`, `dim_room`, `dim_campaign` | `MERGE` | `usp_load_dim_product` là bản mẫu; 4 dim còn lại dùng cùng khuôn `MERGE`, mỗi dim một thủ tục `usp_load_dim_<tên>` |
| Tham chiếu | `dim_membership_tier` | Seed một lần | [seed.md](seed.md) |
| Junk | `dim_booking_junk` | Sinh sẵn toàn bộ tổ hợp | [seed.md](seed.md) |

---

## 1. Mẫu SCD Type 2


### Nạp dimension SCD Type 2 (kết hợp Type 1)

```sql
CREATE OR ALTER PROCEDURE dm.usp_load_dim_customer
    @run_id UNIQUEIDENTIFIER
AS
BEGIN
    SET NOCOUNT, XACT_ABORT ON;

    -- Bước 0: dựng bản nguồn, băm CHỈ những cột cần theo dõi lịch sử.
    -- `rfm_segment` KHÔNG nằm trong hash: nó là thuộc tính Type 1, bị ghi đè ở
    -- dag_refresh_svg_bi (mục 4). Nếu đưa vào hash thì mỗi lần phân khúc đổi sẽ
    -- sinh một phiên bản SCD2 mới — đúng điều mục 4 cam kết không làm.
    SELECT c.customer_id, c.full_name, c.phone_masked, c.gender,
           c.age_group, c.city, c.membership_tier, c.acquisition_channel,
           ISNULL(ds.salon_sk, -1) AS first_salon_sk,
           HASHBYTES('SHA2_256',
               CONCAT_WS('|', c.age_group, c.city, c.membership_tier,
                              c.acquisition_channel)
           ) AS row_hash
    INTO   #src
    FROM       crt.v_customer_for_dim c   -- view chỉ đọc `crt`, không tham chiếu dm/svg_bi
    LEFT JOIN  dm.dim_salon ds ON ds.salon_id = c.first_salon_id AND ds.is_current = 1
    WHERE  c._is_deleted = 0;

    CREATE UNIQUE CLUSTERED INDEX IX_src ON #src (customer_id);

    -- Giữ luôn rfm_segment của phiên bản vừa đóng để chuyển tiếp sang phiên bản mới;
    -- nếu không, mỗi lần đổi thuộc tính Type-2 sẽ xoá mất phân khúc RFM đang có.
    DECLARE @changed TABLE (customer_id BIGINT PRIMARY KEY, rfm_segment VARCHAR(20) NOT NULL);
    DECLARE @now DATETIME2(3) = SYSUTCDATETIME();

    BEGIN TRAN;

    -- Bước 1: ĐÓNG phiên bản hiện hành của những khách có thay đổi ở cột Type-2
    UPDATE d
       SET d.valid_to    = @now,
           d.is_current  = 0,
           d._updated_at = @now
    OUTPUT deleted.customer_id, deleted.rfm_segment INTO @changed
    FROM   dm.dim_customer d
    JOIN   #src s ON s.customer_id = d.customer_id
    WHERE  d.is_current = 1
      AND  d.row_hash  <> s.row_hash;

    -- Bước 2: MỞ phiên bản mới cho đúng những khách vừa bị đóng
    INSERT INTO dm.dim_customer
        (customer_id, full_name, phone_masked, gender, age_group, city,
         membership_tier, acquisition_channel, rfm_segment, first_salon_sk,
         valid_from, valid_to, is_current, row_hash, _run_id)
    SELECT s.customer_id, s.full_name, s.phone_masked, s.gender, s.age_group, s.city,
           s.membership_tier, s.acquisition_channel, ch.rfm_segment, s.first_salon_sk,
           @now, '9999-12-31', 1, s.row_hash, @run_id
    FROM   #src s
    JOIN   @changed ch ON ch.customer_id = s.customer_id;

    -- Bước 3: THÊM khách hoàn toàn mới (chưa từng có dòng nào trong dim)
    INSERT INTO dm.dim_customer
        (customer_id, full_name, phone_masked, gender, age_group, city,
         membership_tier, acquisition_channel, rfm_segment, first_salon_sk,
         valid_from, valid_to, is_current, row_hash, _run_id)
    SELECT s.customer_id, s.full_name, s.phone_masked, s.gender, s.age_group, s.city,
           s.membership_tier, s.acquisition_channel, 'UNKNOWN', s.first_salon_sk,
           -- valid_from của phiên bản ĐẦU TIÊN phải là mốc mở, KHÔNG phải giờ nạp.
           -- Job dim chạy 06:00 ngày N+1 còn hoá đơn phát sinh 09:00 ngày N; để @now
           -- thì temporal join không khớp phiên bản nào và fact rơi hết về sk = -1 —
           -- xảy ra với mọi khách/chi nhánh/KTV/dịch vụ mới, và 100% dữ liệu backfill.
           '1900-01-01', '9999-12-31', 1, s.row_hash, @run_id
    FROM   #src s
    WHERE  NOT EXISTS (SELECT 1 FROM dm.dim_customer d WHERE d.customer_id = s.customer_id);

    -- Bước 4: GHI ĐÈ thuộc tính Type-1 trên phiên bản hiện hành (sửa chính tả, đổi số ĐT)
    UPDATE d
       SET d.full_name    = s.full_name,
           d.phone_masked = s.phone_masked,
           d.gender       = s.gender,
           d._updated_at  = @now
    FROM   dm.dim_customer d
    JOIN   #src s ON s.customer_id = d.customer_id
    WHERE  d.is_current = 1
      AND (d.full_name <> s.full_name OR d.phone_masked <> s.phone_masked OR d.gender <> s.gender);

    COMMIT;
END
```

**Bốn bước, đúng thứ tự này, không đổi được:**

| Bước | Việc | Nếu làm sai thứ tự |
|---|---|---|
| 1 | Đóng phiên bản cũ, **ghi lại danh sách bị đóng** qua `OUTPUT` | Không có `OUTPUT` thì bước 2 phải suy đoán "khách nào vừa đổi" — cách phổ biến là so `valid_to` với thời gian, rất dễ sai khi job chạy lại |
| 2 | Mở phiên bản mới cho **đúng** danh sách đó | Nếu mở cho mọi khách → sinh phiên bản trùng, làm tăng kích thước dim |
| 3 | Thêm khách mới | `NOT EXISTS` quét toàn bảng nên không phụ thuộc thứ tự với bước 2; đặt sau bước 2 để mọi dòng vừa chèn đều được bước 4 cập nhật trong cùng giao dịch |
| 4 | Ghi đè Type-1 | Phải chạy **sau cùng**, để phiên bản mới ở bước 2 cũng được cập nhật |

Thủ tục này **idempotent**: chạy lại với cùng dữ liệu nguồn thì bước 1 không tìm thấy chênh lệch hash → không làm gì cả.

### Kiểm tra tính đúng đắn của SCD2 — DQ rule bắt buộc

```sql
-- DQ-SCD-001: không được có KHOẢNG HỞ hoặc KHOẢNG CHỒNG trong lịch sử một khách
WITH v AS (
    SELECT customer_id, valid_from, valid_to,
           LEAD(valid_from) OVER (PARTITION BY customer_id ORDER BY valid_from) AS next_from
    FROM   dm.dim_customer
    WHERE  customer_sk <> -1
)
SELECT * FROM v
WHERE  next_from IS NOT NULL AND next_from <> valid_to;   -- phải liền mạch tuyệt đối

-- DQ-SCD-002: mỗi khách có đúng MỘT phiên bản hiện hành
SELECT customer_id, COUNT(*) AS current_cnt
FROM   dm.dim_customer
WHERE  is_current = 1
GROUP  BY customer_id
HAVING COUNT(*) <> 1;
```

Không có hai rule này thì lỗi SCD2 sẽ âm thầm làm **nhân đôi dòng fact** khi temporal join, và biểu hiện ra ngoài là "doanh thu tự nhiên tăng gấp đôi ở vài khách hàng".

### Nạp fact

Mẫu delete-insert theo phân vùng, kèm temporal join — xem [mục 4.2](load-fact.md#2-transaction-fact--fact_sales_line). Bổ sung một chi tiết vật lý ở đây:

```sql
-- CHỈ dùng khi nạp lại TRỌN một tháng: phân vùng theo tháng, nên truncate một phân
-- vùng để nạp lại một ngày sẽ xoá cả tháng đó. Nạp lại một ngày thì dùng DELETE
-- theo date_key như thủ tục nạp fact đang làm.
-- WITH (PARTITIONS(...)) chỉ nhận hằng số, không nhận biến, nên phải qua dynamic SQL.
DECLARE @sql NVARCHAR(400) =
    N'TRUNCATE TABLE dm.fact_sales_line WITH (PARTITIONS ('
    + CAST(@partition_number AS NVARCHAR(10)) + N'));';
EXEC sys.sp_executesql @sql;
```

---
---

## 2. Ba dimension SCD2 còn lại

`dim_salon`, `dim_employee`, `dim_service` dùng **đúng 4 bước như trên**, chỉ khác nguồn và tập cột băm. Bảng dưới đây là tham số hoá của cùng một khuôn:

| Dimension | Nguồn | Cột theo dõi lịch sử (đưa vào `row_hash`) | Cột Type 1 (ghi đè) |
|---|---|---|---|
| `dim_customer` | `crt.v_customer_for_dim` | `age_group`, `city`, `membership_tier`, `acquisition_channel`, `rfm_segment` | `full_name`, `phone_masked`, `gender` |
| `dim_salon` | `crt.salon` | `city`, `district`, `region`, `salon_size_band`, `is_active` | `salon_name`, `address` |
| `dim_employee` | `crt.employee` | `role_name`, `skill_level`, `current_salon_sk`, `tenure_band`, `is_active` | `employee_name` |
| `dim_service` | `crt.service` | `category_l1`, `category_l2`, `price_band`, `is_signature`, `is_active` | `service_name`, `standard_duration_min`, `list_price_amount` |

```sql
CREATE OR ALTER PROCEDURE dm.usp_load_dim_salon
    @run_id UNIQUEIDENTIFIER
AS
BEGIN
    SET NOCOUNT, XACT_ABORT ON;

    SELECT s.salon_id, s.salon_name, s.salon_code, s.city, s.district, s.address,
           s.region, s.capacity_beds,
           CASE WHEN s.capacity_beds <  5 THEN 'Small'
                WHEN s.capacity_beds <= 10 THEN 'Medium'
                ELSE 'Large' END AS salon_size_band,
           s.open_date, s.close_date, s.is_active,
           HASHBYTES('SHA2_256', CONCAT_WS('|', s.city, s.district, s.region,
               CASE WHEN s.capacity_beds < 5 THEN 'Small'
                    WHEN s.capacity_beds <= 10 THEN 'Medium' ELSE 'Large' END,
               CAST(s.is_active AS CHAR(1)))) AS row_hash
    INTO   #src
    FROM   crt.salon s
    WHERE  s._is_deleted = 0;

    CREATE UNIQUE CLUSTERED INDEX IX_src ON #src (salon_id);

    DECLARE @changed TABLE (salon_id BIGINT PRIMARY KEY);
    DECLARE @now DATETIME2(3) = SYSUTCDATETIME();

    BEGIN TRAN;

    -- Bước 1: đóng phiên bản hiện hành của những salon có thay đổi
    UPDATE d SET d.valid_to = @now, d.is_current = 0, d._updated_at = @now
    OUTPUT deleted.salon_id INTO @changed
    FROM   dm.dim_salon d JOIN #src s ON s.salon_id = d.salon_id
    WHERE  d.is_current = 1 AND d.row_hash <> s.row_hash;

    -- Bước 2: mở phiên bản mới cho đúng danh sách vừa bị đóng
    INSERT INTO dm.dim_salon (salon_id, salon_name, salon_code, city, district, address,
        region, capacity_beds, salon_size_band, open_date, close_date, is_active,
        valid_from, valid_to, is_current, row_hash, _run_id)
    SELECT s.salon_id, s.salon_name, s.salon_code, s.city, s.district, s.address,
           s.region, s.capacity_beds, s.salon_size_band, s.open_date, s.close_date, s.is_active,
           @now, '9999-12-31', 1, s.row_hash, @run_id
    FROM   #src s JOIN @changed ch ON ch.salon_id = s.salon_id;

    -- Bước 3: thêm salon hoàn toàn mới
    INSERT INTO dm.dim_salon (salon_id, salon_name, salon_code, city, district, address,
        region, capacity_beds, salon_size_band, open_date, close_date, is_active,
        valid_from, valid_to, is_current, row_hash, _run_id)
    SELECT s.salon_id, s.salon_name, s.salon_code, s.city, s.district, s.address,
           s.region, s.capacity_beds, s.salon_size_band, s.open_date, s.close_date, s.is_active,
           @now, '9999-12-31', 1, s.row_hash, @run_id
    FROM   #src s
    WHERE  NOT EXISTS (SELECT 1 FROM dm.dim_salon d WHERE d.salon_id = s.salon_id);

    -- Bước 4: ghi đè thuộc tính Type 1 trên phiên bản hiện hành
    UPDATE d SET d.salon_name = s.salon_name, d.address = s.address, d._updated_at = @now
    FROM   dm.dim_salon d JOIN #src s ON s.salon_id = d.salon_id
    WHERE  d.is_current = 1
      AND (d.salon_name <> s.salon_name OR d.address <> s.address);

    COMMIT;
END
```

`usp_load_dim_employee` và `usp_load_dim_service` viết theo đúng khuôn này, thay nguồn và tập cột theo bảng tham số ở trên.

> **Ba lỗi thường gặp khi nhân bản khuôn này:**
> 1. Quên `OUTPUT ... INTO @changed` ở bước 1 → bước 2 phải suy đoán ai vừa đổi, thường suy theo thời gian và sai khi job chạy lại.
> 2. Đảo bước 2 và bước 3 → khách vừa đổi bị coi là mới và thêm lần hai.
> 3. Đưa cột Type 1 vào `row_hash` → sửa lỗi chính tả tên khách cũng sinh một phiên bản lịch sử mới, làm phình dim vô ích.

---

## 3. Mẫu SCD Type 1 — dùng chung cho 5 dimension

SCD1 chỉ ghi đè, không giữ lịch sử, nên một procedure tham số hoá là đủ.

```sql
CREATE OR ALTER PROCEDURE dm.usp_load_dim_product
    @run_id UNIQUEIDENTIFIER
AS
BEGIN
    SET NOCOUNT, XACT_ABORT ON;

    MERGE dm.dim_product AS tgt
    USING (SELECT product_id, product_name, category, brand, unit, is_retail, is_consumable
           FROM   crt.product WHERE _is_deleted = 0) AS src
       ON tgt.product_id = src.product_id
    WHEN MATCHED AND (tgt.product_name <> src.product_name
                   OR tgt.category     <> src.category
                   OR tgt.brand        <> src.brand
                   OR tgt.unit         <> src.unit
                   OR tgt.is_retail    <> src.is_retail
                   OR tgt.is_consumable<> src.is_consumable)
        THEN UPDATE SET tgt.product_name  = src.product_name,
                        tgt.category      = src.category,
                        tgt.brand         = src.brand,
                        tgt.unit          = src.unit,
                        tgt.is_retail     = src.is_retail,
                        tgt.is_consumable = src.is_consumable,
                        tgt._updated_at   = SYSUTCDATETIME()
    WHEN NOT MATCHED BY TARGET
        THEN INSERT (product_id, product_name, category, brand, unit, is_retail, is_consumable)
             VALUES (src.product_id, src.product_name, src.category, src.brand,
                     src.unit, src.is_retail, src.is_consumable);
    -- KHÔNG dùng WHEN NOT MATCHED BY SOURCE THEN DELETE:
    -- sản phẩm ngừng bán vẫn phải giữ để fact lịch sử tra được khoá.
END
```

`usp_load_dim_promotion`, `usp_load_dim_payment_method`, `usp_load_dim_room`, `usp_load_dim_campaign` theo đúng khuôn này.

> **Không bao giờ `DELETE` trong dimension.** Sản phẩm ngừng bán, KTV nghỉ việc, salon đóng cửa — tất cả vẫn phải tồn tại trong dim, đánh dấu `is_active = 0`. Xoá đi thì fact lịch sử mất khoá và doanh thu quá khứ bị hụt.

---

## 4. Cập nhật thuộc tính phái sinh

Ba cột trong `dim_customer` phụ thuộc dữ liệu tổng hợp nên phải cập nhật **sau** khi `agg_customer_360` được làm mới, tạo thành vòng phụ thuộc hai chiều.

| Cột | Phụ thuộc | Thứ tự trong DAG |
|---|---|---|
| `rfm_segment` | `svg_bi.agg_customer_360` | Sau `dag_refresh_svg_bi` |
| `age_group` | Chỉ `date_of_birth` | Cùng lúc nạp dim |
| `tenure_band` (`dim_employee`) | Chỉ `hire_date` | Cùng lúc nạp dim |

Cách phá vòng phụ thuộc: `rfm_segment` được cập nhật **ở lần chạy hôm sau**, dùng phân khúc của ngày trước. Đây là đánh đổi có ý thức — phân khúc RFM trễ một ngày không ảnh hưởng quyết định nghiệp vụ, còn để vòng phụ thuộc thì DAG không chạy được.

```sql
-- Chạy trong dag_refresh_svg_bi, task cuối cùng
UPDATE d
   SET d.rfm_segment = a.rfm_segment, d._updated_at = SYSUTCDATETIME()
FROM   dm.dim_customer d
JOIN   svg_bi.agg_customer_360 a ON a.customer_sk = d.customer_sk
WHERE  d.is_current = 1 AND d.rfm_segment <> a.rfm_segment;
```

> Cập nhật này là **Type 1** (ghi đè), cố ý không sinh phiên bản SCD2 mới. Phân khúc RFM biến động hằng ngày; theo dõi lịch sử nó sẽ làm `dim_customer` phình lên gấp nhiều lần mà không mang thêm giá trị phân tích.

---

## 5. Kiểm chứng sau khi nạp

Chạy ngay sau `dag_build_datamart`, trước khi nạp fact:

```sql
EXEC ctl.usp_run_dq_group @group = 'SCD', @run_id = @run_id, @business_date = @business_date;
```

Bốn quy tắc, đều mức `BLOCK`: lịch sử liền mạch (`DQ-SCD-001`), đúng một phiên bản hiện hành (`DQ-SCD-002`), tồn tại dòng Unknown `-1` ở 10 dim có khoá đại diện (`DQ-SCD-003`), và giá trị "không xác định" ở 3 dim dùng khoá tự nhiên (`DQ-SCD-004`). Thủ tục cổng: [ctl.usp_run_dq_group](../03-ddl/06-ctl-qtn.md#thủ-tục-chạy-cổng-chất-lượng--ctlusp_run_dq_group). Chi tiết quy tắc: [dq-rules.md mục 7](../05-quality/dq-rules.md#7-quy-tắc-cho-mô-hình-chiều).

Lỗi SCD2 làm **nhân đôi dòng fact** khi tra khoá theo thời điểm — biểu hiện ra ngoài là doanh thu của vài khách tự tăng gấp đôi, rất khó truy nếu không chạy ba quy tắc này trước khi nạp fact.
