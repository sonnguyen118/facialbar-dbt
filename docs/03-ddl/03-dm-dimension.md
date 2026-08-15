# DDL — Schema `dm` · Dimension

13 dimension của star schema. Nguồn từng cột: [Source-to-Target Mapping](../02-mapping/source-to-target.md#ánh-xạ-crt-sang-dm).
Script nạp dữ liệu khởi tạo: [04-etl/seed.md](../04-etl/seed.md). Quy trình nạp: [04-etl/load-dimension.md](../04-etl/load-dimension.md).



## 1. `dim_date` — dimension nền tảng, phải làm đầu tiên

```sql
CREATE TABLE dm.dim_date (
    date_key          INT           NOT NULL,   -- 20260814
    full_date         DATE          NOT NULL,
    day_of_month      TINYINT       NOT NULL,
    day_of_week_iso   TINYINT       NOT NULL,   -- 1 = Thứ Hai ... 7 = Chủ Nhật
    day_name_vi       NVARCHAR(12)  NOT NULL,
    week_of_year_iso  TINYINT       NOT NULL,
    month_number      TINYINT       NOT NULL,
    month_name_vi     NVARCHAR(12)  NOT NULL,
    quarter_number    TINYINT       NOT NULL,
    year_number       SMALLINT      NOT NULL,
    year_month        INT           NOT NULL,   -- 202608 — khoá tổng hợp theo tháng
    year_quarter      INT           NOT NULL,   -- 20263
    -- Cờ nghiệp vụ: nhóm BIT liền nhau để SQL Server đóng gói vào cùng byte
    is_weekend        BIT           NOT NULL,
    is_month_end      BIT           NOT NULL,
    is_vn_holiday     BIT           NOT NULL,
    is_tet_season     BIT           NOT NULL,   -- 21 ngày trước Tết: cao điểm của ngành spa
    holiday_name_vi   NVARCHAR(50)  NULL,
    CONSTRAINT PK_dim_date PRIMARY KEY CLUSTERED (date_key),
    CONSTRAINT UQ_dim_date_full_date UNIQUE (full_date),
    CONSTRAINT CK_dim_date_dow CHECK (day_of_week_iso BETWEEN 1 AND 7)
);

CREATE INDEX IX_dim_date_year_month ON dm.dim_date (year_month) INCLUDE (full_date);
```

Nạp dữ liệu (10 năm = 4.018 dòng — nạp một lần, không bao giờ nạp lại):

```sql
WITH d AS (
    SELECT CAST('2022-01-01' AS DATE) AS dt
    UNION ALL
    SELECT DATEADD(DAY, 1, dt) FROM d WHERE dt < '2032-12-31'
)
INSERT INTO dm.dim_date
    (date_key, full_date, day_of_month, day_of_week_iso, day_name_vi, week_of_year_iso,
     month_number, month_name_vi, quarter_number, year_number, year_month, year_quarter,
     is_weekend, is_month_end, is_vn_holiday, is_tet_season, holiday_name_vi)
SELECT
    YEAR(dt)*10000 + MONTH(dt)*100 + DAY(dt),
    dt,
    DAY(dt),
    dow.iso,
    CHOOSE(dow.iso, N'Thứ Hai', N'Thứ Ba', N'Thứ Tư', N'Thứ Năm', N'Thứ Sáu', N'Thứ Bảy', N'Chủ Nhật'),
    DATEPART(ISO_WEEK, dt),
    MONTH(dt),
    CONCAT(N'Tháng ', MONTH(dt)),
    DATEPART(QUARTER, dt),
    YEAR(dt),
    YEAR(dt)*100 + MONTH(dt),
    YEAR(dt)*10  + DATEPART(QUARTER, dt),
    CASE WHEN dow.iso >= 6 THEN 1 ELSE 0 END,
    CASE WHEN dt = EOMONTH(dt) THEN 1 ELSE 0 END,
    0, 0, NULL                              -- ngày lễ được UPDATE riêng, xem ghi chú
FROM d
CROSS APPLY (SELECT ((DATEPART(WEEKDAY, dt) + @@DATEFIRST - 2) % 7) + 1 AS iso) AS dow
OPTION (MAXRECURSION 0);
```

> **Hai chi tiết không hiển nhiên trong đoạn trên:**
>
> **1. Công thức `((DATEPART(WEEKDAY, dt) + @@DATEFIRST - 2) % 7) + 1`.** `DATEPART(WEEKDAY)` phụ thuộc thiết lập `@@DATEFIRST` của session — cùng một câu SQL chạy ở hai server có thể ra thứ khác nhau. Công thức này triệt tiêu ảnh hưởng đó, luôn cho Thứ Hai = 1.
>
> **2. `is_vn_holiday` và `is_tet_season` để 0 rồi UPDATE sau, không tính bằng công thức.** Tết theo âm lịch nên **không có công thức**; ngày nghỉ bù cũng do Chính phủ công bố từng năm. Vì vậy cần bảng `ctl.vn_holiday` nạp thủ công mỗi năm một lần:
> ```sql
> CREATE TABLE ctl.vn_holiday (
>     holiday_date  DATE          NOT NULL CONSTRAINT PK_vn_holiday PRIMARY KEY,
>     holiday_name  NVARCHAR(50)  NOT NULL,
>     is_tet        BIT           NOT NULL
> );
>
> UPDATE d
>    SET d.is_vn_holiday   = 1,
>        d.holiday_name_vi = h.holiday_name
> FROM dm.dim_date d JOIN ctl.vn_holiday h ON h.holiday_date = d.full_date;
>
> -- Cao điểm spa: 21 ngày trước ngày đầu Tết
> UPDATE d SET d.is_tet_season = 1
> FROM dm.dim_date d
> WHERE EXISTS (SELECT 1 FROM ctl.vn_holiday h
>               WHERE h.is_tet = 1 AND d.full_date BETWEEN DATEADD(DAY,-21,h.holiday_date) AND h.holiday_date);
> ```
> Thiếu cờ này thì mọi phân tích so sánh cùng kỳ sẽ sai lệch nặng vào tháng 1–2, và model dự báo nhu cầu sẽ không giải thích được cú tăng vọt trước Tết.

## 2. `dim_time` — tách riêng khỏi `dim_date`

```sql
CREATE TABLE dm.dim_time (
    time_key        SMALLINT     NOT NULL,   -- 0..1439 = số phút kể từ 00:00
    time_value      TIME(0)      NOT NULL,
    hour_24         TINYINT      NOT NULL,
    minute_of_hour  TINYINT      NOT NULL,
    slot_15min      VARCHAR(11)  NOT NULL,   -- '14:00-14:15' — khớp slot đặt lịch của salon
    time_band_vi    NVARCHAR(10) NOT NULL,   -- Sáng / Trưa / Chiều / Tối
    is_peak_hour    BIT          NOT NULL,   -- 17:00–20:00
    CONSTRAINT PK_dim_time PRIMARY KEY CLUSTERED (time_key)
);
```

> **Căn cứ — tách `dim_date` và `dim_time` thành hai bảng thay vì một `dim_datetime`:**
> Một dimension datetime ở mức phút cho 10 năm sẽ có `4.018 × 1.440 = 5,79 triệu` dòng — mất hết ưu điểm "dimension nhỏ, join nhanh". Tách ra thì tổng chỉ còn `4.018 + 1.440 = 5.458` dòng. Đây là mẫu thiết kế chuẩn, không phải tối ưu hoá non.

## 3. `dim_customer` — SCD Type 2 kết hợp Type 1

```sql
CREATE TABLE dm.dim_customer (
    customer_sk         BIGINT        IDENTITY(1,1) NOT NULL,
    customer_id         BIGINT        NOT NULL,          -- business key, LẶP qua các phiên bản
    -- Thuộc tính Type 1 (ghi đè, không giữ lịch sử)
    full_name           NVARCHAR(200) NOT NULL,
    phone_masked        VARCHAR(20)   NOT NULL,          -- '090****567' — bản đầy đủ chỉ ở crt
    gender              VARCHAR(10)   NOT NULL,
    -- Thuộc tính Type 2 (thay đổi thì tạo phiên bản mới)
    age_group           VARCHAR(20)   NOT NULL,          -- <25 / 25-34 / 35-44 / 45+
    city                NVARCHAR(50)  NOT NULL,
    membership_tier     VARCHAR(20)   NOT NULL,          -- None/Silver/Gold/Platinum
    acquisition_channel VARCHAR(50)   NOT NULL,
    rfm_segment         VARCHAR(30)   NOT NULL,          -- Champion / Loyal / At-Risk / Lost
    first_salon_sk      INT           NOT NULL,
    -- Bộ điều khiển SCD2
    valid_from          DATETIME2(3)  NOT NULL,
    valid_to            DATETIME2(3)  NOT NULL,          -- '9999-12-31' nếu đang hiệu lực
    is_current          BIT           NOT NULL,
    row_hash            VARBINARY(32) NOT NULL,
    _run_id             UNIQUEIDENTIFIER NOT NULL,
    _updated_at         DATETIME2(3)  NOT NULL CONSTRAINT DF_dim_customer_upd DEFAULT (SYSUTCDATETIME()),

    CONSTRAINT PK_dim_customer PRIMARY KEY CLUSTERED (customer_sk),
    CONSTRAINT UQ_dim_customer_bk_validfrom UNIQUE (customer_id, valid_from),
    CONSTRAINT CK_dim_customer_validity CHECK (valid_to > valid_from),
    CONSTRAINT CK_dim_customer_gender CHECK (gender IN ('F','M','OTHER','UNKNOWN'))
);

-- Chỉ MỘT phiên bản được là hiện hành cho mỗi khách. Filtered index vừa ràng buộc vừa tăng tốc.
CREATE UNIQUE INDEX UX_dim_customer_current
    ON dm.dim_customer (customer_id)
    WHERE is_current = 1;

-- Phục vụ temporal join khi nạp fact lịch sử
CREATE INDEX IX_dim_customer_bk_range
    ON dm.dim_customer (customer_id, valid_from, valid_to)
    INCLUDE (customer_sk, membership_tier);
```

**Dòng Unknown member** — phải seed ngay sau khi tạo bảng, trước khi nạp fact đầu tiên:

```sql
SET IDENTITY_INSERT dm.dim_customer ON;
INSERT INTO dm.dim_customer
    (customer_sk, customer_id, full_name, phone_masked, gender, age_group, city,
     membership_tier, acquisition_channel, rfm_segment, first_salon_sk,
     valid_from, valid_to, is_current, row_hash, _run_id)
VALUES
    (-1, -1, N'(Không xác định)', 'N/A', 'UNKNOWN', 'UNKNOWN', N'(Không xác định)',
     'None', 'UNKNOWN', 'UNKNOWN', -1,
     '1900-01-01', '9999-12-31', 1, 0x00, '00000000-0000-0000-0000-000000000000');
SET IDENTITY_INSERT dm.dim_customer OFF;
```

Cần `SET IDENTITY_INSERT` vì `IDENTITY(1,1)` không tự sinh được giá trị âm. **Mọi dimension đều phải có dòng `-1` này** — thiếu nó thì fact có khoá lỗi sẽ bị `INNER JOIN` xoá mất, và doanh thu bị hụt không có dấu vết.

## 4. Các dimension SCD2 còn lại

```sql
CREATE TABLE dm.dim_salon (
    salon_sk        INT           IDENTITY(1,1) NOT NULL,
    salon_id        BIGINT        NOT NULL,
    salon_name      NVARCHAR(100) NOT NULL,
    salon_code      VARCHAR(20)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    city            NVARCHAR(50)  NOT NULL,
    district        NVARCHAR(50)  NOT NULL,
    address         NVARCHAR(255) NOT NULL,
    region          VARCHAR(20)   NOT NULL,          -- Bắc / Trung / Nam
    capacity_beds   TINYINT       NOT NULL,
    salon_size_band VARCHAR(20)   NOT NULL,          -- Small / Medium / Large
    open_date       DATE          NOT NULL,
    close_date      DATE          NULL,
    is_active       BIT           NOT NULL,
    valid_from      DATETIME2(3)  NOT NULL,
    valid_to        DATETIME2(3)  NOT NULL,
    is_current      BIT           NOT NULL,
    row_hash        VARBINARY(32) NOT NULL,
    _run_id         UNIQUEIDENTIFIER NOT NULL,
    _updated_at     DATETIME2(3)  NOT NULL CONSTRAINT DF_dim_salon_upd DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT PK_dim_salon PRIMARY KEY CLUSTERED (salon_sk),
    CONSTRAINT UQ_dim_salon_bk_validfrom UNIQUE (salon_id, valid_from),
    CONSTRAINT CK_dim_salon_beds CHECK (capacity_beds > 0)
);
CREATE UNIQUE INDEX UX_dim_salon_current ON dm.dim_salon (salon_id) WHERE is_current = 1;

CREATE TABLE dm.dim_employee (
    employee_sk     INT           IDENTITY(1,1) NOT NULL,
    employee_id     BIGINT        NOT NULL,
    employee_name   NVARCHAR(200) NOT NULL,
    employee_code   VARCHAR(20)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    role_name       VARCHAR(30)   NOT NULL,          -- therapist / receptionist / manager
    skill_level     VARCHAR(20)   NOT NULL,          -- Junior / Senior / Expert
    current_salon_sk INT          NOT NULL,
    hire_date       DATE          NOT NULL,
    terminate_date  DATE          NULL,
    tenure_band     VARCHAR(20)   NOT NULL,          -- <6m / 6-12m / 1-3y / 3y+
    is_active       BIT           NOT NULL,
    valid_from      DATETIME2(3)  NOT NULL,
    valid_to        DATETIME2(3)  NOT NULL,
    is_current      BIT           NOT NULL,
    row_hash        VARBINARY(32) NOT NULL,
    _run_id         UNIQUEIDENTIFIER NOT NULL,
    _updated_at     DATETIME2(3)  NOT NULL CONSTRAINT DF_dim_employee_upd DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT PK_dim_employee PRIMARY KEY CLUSTERED (employee_sk),
    CONSTRAINT UQ_dim_employee_bk_validfrom UNIQUE (employee_id, valid_from),
    CONSTRAINT CK_dim_employee_role CHECK (role_name IN ('therapist','receptionist','manager','other'))
);
CREATE UNIQUE INDEX UX_dim_employee_current ON dm.dim_employee (employee_id) WHERE is_current = 1;

CREATE TABLE dm.dim_service (
    service_sk       INT           IDENTITY(1,1) NOT NULL,
    service_id       BIGINT        NOT NULL,
    service_name     NVARCHAR(150) NOT NULL,
    service_code     VARCHAR(30)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    category_l1      NVARCHAR(50)  NOT NULL,        -- Facial / Body / Nail / Combo
    category_l2      NVARCHAR(50)  NOT NULL,        -- Hydrafacial / Peeling / ...
    standard_duration_min SMALLINT NOT NULL,
    list_price_amount DECIMAL(18,2) NOT NULL,
    price_band       VARCHAR(20)   NOT NULL,        -- Economy / Standard / Premium
    is_signature     BIT           NOT NULL,
    is_active        BIT           NOT NULL,
    valid_from       DATETIME2(3)  NOT NULL,
    valid_to         DATETIME2(3)  NOT NULL,
    is_current       BIT           NOT NULL,
    row_hash         VARBINARY(32) NOT NULL,
    _run_id          UNIQUEIDENTIFIER NOT NULL,
    _updated_at      DATETIME2(3)  NOT NULL CONSTRAINT DF_dim_service_upd DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT PK_dim_service PRIMARY KEY CLUSTERED (service_sk),
    CONSTRAINT UQ_dim_service_bk_validfrom UNIQUE (service_id, valid_from),
    CONSTRAINT CK_dim_service_duration CHECK (standard_duration_min BETWEEN 5 AND 480),
    CONSTRAINT CK_dim_service_price CHECK (list_price_amount >= 0)
);
CREATE UNIQUE INDEX UX_dim_service_current ON dm.dim_service (service_id) WHERE is_current = 1;
```

> **Căn cứ — `dim_service` cần SCD2 dù giá đã có trong fact:** công ty tái cấu trúc danh mục, chuyển *Hydrafacial* từ nhóm `Standard` sang `Premium`. Nếu ghi đè (Type 1), toàn bộ doanh thu quá khứ của dịch vụ đó bị dồn sang `Premium` → báo cáo "cơ cấu doanh thu theo phân khúc giá" của các năm trước **tự thay đổi**, và không ai hiểu vì sao con số tháng trước khác con số đã in ra.

## 5. Các dimension SCD Type 1 (nhỏ, không cần lịch sử)

Tất cả cùng một khuôn: SK + nghiệp vụ key + thuộc tính + `_updated_at`, không có bộ SCD2.

```sql
CREATE TABLE dm.dim_product (
    product_sk     INT IDENTITY(1,1) NOT NULL CONSTRAINT PK_dim_product PRIMARY KEY CLUSTERED,
    product_id     BIGINT       NOT NULL CONSTRAINT UQ_dim_product_bk UNIQUE,
    product_name   NVARCHAR(150) NOT NULL,
    category       NVARCHAR(50)  NOT NULL,
    brand          NVARCHAR(50)  NOT NULL,
    unit           VARCHAR(20)   NOT NULL,
    is_retail      BIT           NOT NULL,     -- bán lẻ cho khách
    is_consumable  BIT           NOT NULL,     -- vật tư tiêu hao trong buồng
    _updated_at    DATETIME2(3)  NOT NULL CONSTRAINT DF_dim_product_upd DEFAULT (SYSUTCDATETIME())
);

CREATE TABLE dm.dim_promotion (
    promotion_sk   INT IDENTITY(1,1) NOT NULL CONSTRAINT PK_dim_promotion PRIMARY KEY CLUSTERED,
    promotion_id   BIGINT       NOT NULL CONSTRAINT UQ_dim_promotion_bk UNIQUE,
    promotion_name NVARCHAR(150) NOT NULL,
    promotion_type VARCHAR(20)   NOT NULL,     -- percent / amount / gift / bundle
    discount_value DECIMAL(18,2) NOT NULL,
    valid_from_date DATE         NOT NULL,
    valid_to_date   DATE         NOT NULL,
    _updated_at    DATETIME2(3)  NOT NULL CONSTRAINT DF_dim_promotion_upd DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT CK_dim_promotion_type CHECK (promotion_type IN ('percent','amount','gift','bundle','none'))
);

CREATE TABLE dm.dim_payment_method (
    payment_method_sk INT IDENTITY(1,1) NOT NULL CONSTRAINT PK_dim_payment_method PRIMARY KEY CLUSTERED,
    payment_method_code VARCHAR(30) COLLATE Latin1_General_100_BIN2 NOT NULL
        CONSTRAINT UQ_dim_payment_method_bk UNIQUE,
    payment_method_name NVARCHAR(50) NOT NULL,   -- Tiền mặt / Thẻ / QR / Ví điện tử / Voucher
    method_group      VARCHAR(20)  NOT NULL,     -- cash / card / digital / voucher
    is_cash           BIT          NOT NULL,     -- phân biệt tiền mặt để đối soát quỹ
    _updated_at       DATETIME2(3) NOT NULL CONSTRAINT DF_dim_pm_upd DEFAULT (SYSUTCDATETIME())
);

CREATE TABLE dm.dim_room (
    room_sk     INT IDENTITY(1,1) NOT NULL CONSTRAINT PK_dim_room PRIMARY KEY CLUSTERED,
    room_id     BIGINT       NOT NULL CONSTRAINT UQ_dim_room_bk UNIQUE,
    salon_sk    INT          NOT NULL,
    room_name   NVARCHAR(50) NOT NULL,
    room_type   VARCHAR(30)  NOT NULL,           -- single / double / vip
    _updated_at DATETIME2(3) NOT NULL CONSTRAINT DF_dim_room_upd DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT FK_dim_room_dim_salon FOREIGN KEY (salon_sk) REFERENCES dm.dim_salon(salon_sk)
);

CREATE TABLE dm.dim_campaign (
    campaign_sk    INT IDENTITY(1,1) NOT NULL CONSTRAINT PK_dim_campaign PRIMARY KEY CLUSTERED,
    campaign_id    VARCHAR(100) COLLATE Latin1_General_100_BIN2 NOT NULL
        CONSTRAINT UQ_dim_campaign_bk UNIQUE,
    campaign_name  NVARCHAR(200) NOT NULL,
    platform       VARCHAR(30)   NOT NULL,       -- facebook / google / zalo / sms / email
    objective      VARCHAR(30)   NOT NULL,       -- awareness / conversion / retention
    start_date     DATE          NOT NULL,
    end_date       DATE          NULL,
    _updated_at    DATETIME2(3)  NOT NULL CONSTRAINT DF_dim_campaign_upd DEFAULT (SYSUTCDATETIME())
);

-- Bảng THAM CHIẾU quy tắc hạng thành viên (ETL dùng), KHÔNG phải chiều phân tích.
-- Hạng thẻ để phân tích nằm trong dim_customer và được SCD2 theo dõi — xem ghi chú ở mục 2.7.
CREATE TABLE dm.dim_membership_tier (
    tier_code        VARCHAR(20) NOT NULL CONSTRAINT PK_dim_membership_tier PRIMARY KEY CLUSTERED,
    tier_name        NVARCHAR(50) NOT NULL,
    tier_rank        TINYINT      NOT NULL,      -- 0=None, 1=Silver, 2=Gold, 3=Platinum
    min_spend_amount DECIMAL(18,2) NOT NULL,
    discount_pct     DECIMAL(9,4) NOT NULL,
    point_multiplier DECIMAL(9,4) NOT NULL,
    _updated_at      DATETIME2(3) NOT NULL CONSTRAINT DF_dim_tier_upd DEFAULT (SYSUTCDATETIME())
);
```

Mỗi bảng trên cũng cần seed dòng `-1` / `'UNKNOWN'` tương ứng.

## 6. `dim_booking_junk` — Junk dimension

```sql
CREATE TABLE dm.dim_booking_junk (
    booking_junk_sk      INT IDENTITY(1,1) NOT NULL
        CONSTRAINT PK_dim_booking_junk PRIMARY KEY CLUSTERED,
    booking_channel      VARCHAR(20) NOT NULL,   -- app / web / hotline / walk_in
    is_first_visit       BIT NOT NULL,
    is_promotion_applied BIT NOT NULL,
    is_member            BIT NOT NULL,
    is_rescheduled       BIT NOT NULL,
    CONSTRAINT UQ_dim_booking_junk_combo
        UNIQUE (booking_channel, is_first_visit, is_promotion_applied, is_member, is_rescheduled),
    CONSTRAINT CK_dim_booking_junk_channel
        CHECK (booking_channel IN ('app','web','hotline','walk_in','unknown'))
);
```

5 cột cờ nhỏ nếu để trực tiếp trong fact sẽ bị **nhân bản trên từng dòng fact**. Gom lại thành một `INT` duy nhất:

| Cách | Cột trong fact | Byte/dòng | Với 42 triệu dòng |
|---|---|---|---|
| Để cờ trong fact | `VARCHAR(20)` + 4 × `BIT` | ~13 byte | ~546 MB |
| **Junk dimension** | 1 × `INT` | 4 byte | **~168 MB** |

Bảng junk chỉ có tối đa `5 × 2 × 2 × 2 × 2 = 80` dòng — sinh sẵn toàn bộ tổ hợp một lần bằng `CROSS JOIN`, ETL chỉ việc tra khoá.

---
