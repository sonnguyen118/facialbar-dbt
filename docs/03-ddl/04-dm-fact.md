# DDL — Schema `dm` · Fact

10 fact table đủ 3 loại: transaction, periodic snapshot, accumulating snapshot. Bảng cầu nối `dm.bridge_sales_promotion` nằm ở [05-svg-bi.md mục 1](05-svg-bi.md).

độ hạt từng bảng: [01-erd/độ hạt.md](../01-erd/độ hạt.md). Dimension nào dùng cho fact nào: [01-erd/bus-matrix.md](../01-erd/bus-matrix.md). Quy trình nạp: [04-etl/load-fact.md](../04-etl/load-fact.md).



## 1. `fact_sales_line` — Transaction fact, bảng trung tâm

```sql
CREATE TABLE dm.fact_sales_line (
    -- ===== Khoá chiều =====
    service_date_key       INT           NOT NULL,   -- ngày DỊCH VỤ được thực hiện → khoá phân vùng
    invoice_date_key       INT           NOT NULL,   -- ngày lập hoá đơn (role-playing dimension)
    service_time_key       SMALLINT      NOT NULL,
    customer_sk            BIGINT        NOT NULL,
    salon_sk               INT           NOT NULL,
    employee_sk            INT           NOT NULL,   -- KTV thực hiện; -1 với dòng bán sản phẩm
    service_sk             INT           NOT NULL,   -- -1 với dòng sản phẩm
    product_sk             INT           NOT NULL,   -- -1 với dòng dịch vụ
    promotion_sk           INT           NOT NULL,
    campaign_sk            INT           NOT NULL,
    booking_junk_sk        INT           NOT NULL,
    -- ===== Degenerate dimension (mã nghiệp vụ, không có dim riêng) =====
    invoice_line_id        BIGINT        NOT NULL,   -- ĐÂY LÀ GRAIN của bảng
    invoice_no             VARCHAR(30)   NOT NULL,
    invoice_line_no        SMALLINT      NOT NULL,
    treatment_id           BIGINT        NULL,       -- NULL nếu là dòng bán sản phẩm thuần
    booking_id             BIGINT        NULL,       -- NULL nếu khách walk-in
    -- ===== Measure — TẤT CẢ đều additive =====
    quantity               DECIMAL(9,2)  NOT NULL CONSTRAINT DF_fsl_qty     DEFAULT (0),
    unit_price_amount      DECIMAL(18,2) NOT NULL CONSTRAINT DF_fsl_unit    DEFAULT (0),
    gross_amount           DECIMAL(18,2) NOT NULL CONSTRAINT DF_fsl_gross   DEFAULT (0),
    promo_discount_amount  DECIMAL(18,2) NOT NULL CONSTRAINT DF_fsl_pdisc   DEFAULT (0),
    member_discount_amount DECIMAL(18,2) NOT NULL CONSTRAINT DF_fsl_mdisc   DEFAULT (0),
    discount_amount        DECIMAL(18,2) NOT NULL CONSTRAINT DF_fsl_disc    DEFAULT (0),
    net_amount             DECIMAL(18,2) NOT NULL CONSTRAINT DF_fsl_net     DEFAULT (0),
    tax_amount             DECIMAL(18,2) NOT NULL CONSTRAINT DF_fsl_tax     DEFAULT (0),
    net_excl_tax_amount    DECIMAL(18,2) NOT NULL CONSTRAINT DF_fsl_netex   DEFAULT (0),
    cogs_amount            DECIMAL(18,2) NOT NULL CONSTRAINT DF_fsl_cogs    DEFAULT (0),
    gross_margin_amount    DECIMAL(18,2) NOT NULL CONSTRAINT DF_fsl_margin  DEFAULT (0),
    line_count             TINYINT       NOT NULL CONSTRAINT DF_fsl_cnt     DEFAULT (1),
    -- ===== Cột kỹ thuật =====
    _run_id                UNIQUEIDENTIFIER NOT NULL,
    _loaded_at             DATETIME2(3)  NOT NULL CONSTRAINT DF_fsl_loaded DEFAULT (SYSUTCDATETIME()),

    CONSTRAINT CK_fact_sales_line_amount_nonneg
        CHECK (gross_amount >= 0 AND discount_amount >= 0 AND quantity >= 0),
    CONSTRAINT CK_fact_sales_line_net_consistent
        CHECK (ABS(net_amount - (gross_amount - discount_amount)) < 0.01),
    CONSTRAINT CK_fact_sales_line_one_of
        CHECK (service_sk <> -1 OR product_sk <> -1),   -- một dòng phải là dịch vụ HOẶC sản phẩm
    CONSTRAINT FK_fact_sales_line_dim_date
        FOREIGN KEY (service_date_key) REFERENCES dm.dim_date(date_key),
    CONSTRAINT FK_fact_sales_line_dim_date_inv
        FOREIGN KEY (invoice_date_key) REFERENCES dm.dim_date(date_key),
    CONSTRAINT FK_fact_sales_line_dim_time
        FOREIGN KEY (service_time_key) REFERENCES dm.dim_time(time_key),
    CONSTRAINT FK_fact_sales_line_dim_customer
        FOREIGN KEY (customer_sk) REFERENCES dm.dim_customer(customer_sk),
    CONSTRAINT FK_fact_sales_line_dim_salon
        FOREIGN KEY (salon_sk) REFERENCES dm.dim_salon(salon_sk),
    CONSTRAINT FK_fact_sales_line_dim_employee
        FOREIGN KEY (employee_sk) REFERENCES dm.dim_employee(employee_sk),
    CONSTRAINT FK_fact_sales_line_dim_service
        FOREIGN KEY (service_sk) REFERENCES dm.dim_service(service_sk),
    CONSTRAINT FK_fact_sales_line_dim_product
        FOREIGN KEY (product_sk) REFERENCES dm.dim_product(product_sk),
    CONSTRAINT FK_fact_sales_line_dim_promotion
        FOREIGN KEY (promotion_sk) REFERENCES dm.dim_promotion(promotion_sk),
    CONSTRAINT FK_fact_sales_line_dim_campaign
        FOREIGN KEY (campaign_sk) REFERENCES dm.dim_campaign(campaign_sk),
    CONSTRAINT FK_fact_sales_line_dim_junk
        FOREIGN KEY (booking_junk_sk) REFERENCES dm.dim_booking_junk(booking_junk_sk)
) ON ps_date_key_month (service_date_key);

-- Index chính: columnstore, aligned theo phân vùng
CREATE CLUSTERED COLUMNSTORE INDEX CCI_fact_sales_line
    ON dm.fact_sales_line
    ON ps_date_key_month (service_date_key);

-- Hàng rào chống nạp trùng — aligned (chứa cột phân vùng), xem mục 5.4
CREATE UNIQUE INDEX UX_fact_sales_line_grain
    ON dm.fact_sales_line (service_date_key, invoice_line_id);
```

**Bốn quyết định thiết kế trong bảng này, kèm lý do:**

**1. `service_date_key` làm khoá phân vùng, không phải `invoice_date_key`.** Nghiệp vụ ghi nhận doanh thu **theo ngày dịch vụ được thực hiện** (mục 4.3), nên hầu hết truy vấn lọc theo cột này. Phân vùng phải theo cột được lọc nhiều nhất, nếu không thì partition pruning vô dụng.

**2. `net_amount` được lưu vật lý, không dùng computed column.** Có thể khai báo `AS (gross_amount - discount_amount) PERSISTED`, nhưng chọn cách lưu vật lý + `CHECK` + DQ quy tắc vì: (a) khi debug thấy ngay giá trị ETL đã tính; (b) không lệ thuộc cú pháp riêng của SQL Server, nhất quán với yêu cầu giữ chi phí di trú thấp ở [mục 7.6](../08-operations/van-hanh.md#6-khả-năng-mở-rộng-và-phục-hồi).

**3. Cột `line_count` luôn bằng 1.** Đây là *counting fact*. Cần nói rõ nó **không** giải quyết vấn đề độ hạt: ở mức dòng thì `SUM(line_count)` và `COUNT(*)` cho **kết quả y hệt nhau** — cả hai đều đếm số dòng hoá đơn, không đếm số hoá đơn. Muốn đếm số hoá đơn vẫn phải `COUNT(DISTINCT invoice_no)`.

Lý do thật để có cột này chỉ có hai:
- Bảng tổng hợp ở `svg_bi` cần **cộng tiếp** số đếm từ mức ngày lên mức tháng — lúc đó `SUM` là phép duy nhất dùng được, `COUNT(*)` trên bảng đã tổng hợp sẽ ra số dòng của bảng tổng hợp, không phải số hoá đơn.
- Một số công cụ BI chỉ cho chọn `SUM` khi định nghĩa measure.

**4. `CK_fact_sales_line_one_of`** chặn dòng vừa không phải dịch vụ vừa không phải sản phẩm — một dòng hoá đơn "rỗng" thường là dấu hiệu lỗi mapping ở ETL.

## 2. `fact_payment` — tiền thực thu

```sql
CREATE TABLE dm.fact_payment (
    payment_date_key   INT           NOT NULL,
    payment_time_key   SMALLINT      NOT NULL,
    customer_sk        BIGINT        NOT NULL,
    salon_sk           INT           NOT NULL,
    payment_method_sk  INT           NOT NULL,
    -- Degenerate
    payment_id         BIGINT        NOT NULL,       -- GRAIN
    invoice_no         VARCHAR(30)   NOT NULL,
    gateway_txn_id     VARCHAR(100)  NULL,           -- để đối soát với payment gateway
    payment_status     VARCHAR(20)   NOT NULL,       -- completed / failed / refunded
    -- Measure
    payment_amount     DECIMAL(18,2) NOT NULL CONSTRAINT DF_fp_amount DEFAULT (0),
    refund_amount      DECIMAL(18,2) NOT NULL CONSTRAINT DF_fp_refund DEFAULT (0),
    fee_amount         DECIMAL(18,2) NOT NULL CONSTRAINT DF_fp_fee    DEFAULT (0),
    net_cash_amount    DECIMAL(18,2) NOT NULL CONSTRAINT DF_fp_net    DEFAULT (0),
    payment_count      TINYINT       NOT NULL CONSTRAINT DF_fp_cnt    DEFAULT (1),
    _run_id            UNIQUEIDENTIFIER NOT NULL,
    _loaded_at         DATETIME2(3)  NOT NULL CONSTRAINT DF_fp_loaded DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT CK_fact_payment_status
        CHECK (payment_status IN ('completed','failed','refunded','pending')),
    CONSTRAINT FK_fact_payment_dim_date
        FOREIGN KEY (payment_date_key) REFERENCES dm.dim_date(date_key),
    CONSTRAINT FK_fact_payment_dim_customer
        FOREIGN KEY (customer_sk) REFERENCES dm.dim_customer(customer_sk),
    CONSTRAINT FK_fact_payment_dim_salon
        FOREIGN KEY (salon_sk) REFERENCES dm.dim_salon(salon_sk),
    CONSTRAINT FK_fact_payment_dim_pm
        FOREIGN KEY (payment_method_sk) REFERENCES dm.dim_payment_method(payment_method_sk),
    CONSTRAINT FK_fact_payment_dim_time
        FOREIGN KEY (payment_time_key) REFERENCES dm.dim_time(time_key)
) ON ps_date_key_month (payment_date_key);

CREATE CLUSTERED COLUMNSTORE INDEX CCI_fact_payment
    ON dm.fact_payment ON ps_date_key_month (payment_date_key);
CREATE UNIQUE INDEX UX_fact_payment_grain
    ON dm.fact_payment (payment_date_key, payment_id);
```

> **Lưu `payment_status = 'failed'` trong fact là có chủ đích.** Nhiều thiết kế chỉ nạp giao dịch thành công, và thế là mất vĩnh viễn khả năng trả lời "tỷ lệ thanh toán thất bại theo cổng thanh toán là bao nhiêu" — một chỉ số vận hành quan trọng. Đổi lại, **mọi truy vấn doanh thu bắt buộc phải có `WHERE payment_status = 'completed'`**, nên phải tạo view chuẩn để người dùng không quên:
> ```sql
> CREATE VIEW dm.v_fact_payment_completed AS
> SELECT * FROM dm.fact_payment WHERE payment_status = 'completed';
> ```

## 3. Các transaction fact còn lại

```sql
CREATE TABLE dm.fact_booking_line (
    booked_date_key     INT      NOT NULL,          -- ngày khách BẤM đặt
    requested_date_key  INT      NOT NULL,          -- ngày khách MUỐN đến (role-playing)
    booked_time_key     SMALLINT NOT NULL,
    customer_sk         BIGINT   NOT NULL,
    salon_sk            INT      NOT NULL,
    service_sk          INT      NOT NULL,
    promotion_sk        INT      NOT NULL,
    campaign_sk         INT      NOT NULL,
    booking_junk_sk     INT      NOT NULL,
    booking_item_id     BIGINT   NOT NULL,          -- GRAIN
    booking_id          BIGINT   NOT NULL,
    booking_status      VARCHAR(20) NOT NULL,       -- created/confirmed/cancelled/completed
    cancel_reason_code  VARCHAR(30) NULL,
    quantity            DECIMAL(9,2)  NOT NULL CONSTRAINT DF_fbl_qty  DEFAULT (0),
    booked_amount       DECIMAL(18,2) NOT NULL CONSTRAINT DF_fbl_amt  DEFAULT (0),
    lead_time_hours     DECIMAL(9,2)  NOT NULL CONSTRAINT DF_fbl_lead DEFAULT (0),  -- đặt trước bao lâu
    booking_line_count  TINYINT  NOT NULL CONSTRAINT DF_fbl_cnt DEFAULT (1),
    is_cancelled        BIT      NOT NULL CONSTRAINT DF_fbl_canc DEFAULT (0),
    _run_id             UNIQUEIDENTIFIER NOT NULL,
    _loaded_at          DATETIME2(3) NOT NULL CONSTRAINT DF_fbl_loaded DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT CK_fact_booking_line_status
        CHECK (booking_status IN ('created','confirmed','cancelled','completed','rescheduled')),
    CONSTRAINT CK_fact_booking_line_lead CHECK (lead_time_hours >= 0),
    CONSTRAINT FK_fact_booking_line_dim_date     FOREIGN KEY (booked_date_key)    REFERENCES dm.dim_date(date_key),
    CONSTRAINT FK_fact_booking_line_dim_date_req FOREIGN KEY (requested_date_key) REFERENCES dm.dim_date(date_key),
    CONSTRAINT FK_fact_booking_line_dim_time     FOREIGN KEY (booked_time_key)    REFERENCES dm.dim_time(time_key),
    CONSTRAINT FK_fact_booking_line_dim_customer FOREIGN KEY (customer_sk)     REFERENCES dm.dim_customer(customer_sk),
    CONSTRAINT FK_fact_booking_line_dim_salon    FOREIGN KEY (salon_sk)        REFERENCES dm.dim_salon(salon_sk),
    CONSTRAINT FK_fact_booking_line_dim_service  FOREIGN KEY (service_sk)      REFERENCES dm.dim_service(service_sk),
    CONSTRAINT FK_fact_booking_line_dim_promo    FOREIGN KEY (promotion_sk)    REFERENCES dm.dim_promotion(promotion_sk),
    CONSTRAINT FK_fact_booking_line_dim_campaign FOREIGN KEY (campaign_sk)     REFERENCES dm.dim_campaign(campaign_sk),
    CONSTRAINT FK_fact_booking_line_dim_junk     FOREIGN KEY (booking_junk_sk) REFERENCES dm.dim_booking_junk(booking_junk_sk)
) ON ps_date_key_month (booked_date_key);
CREATE CLUSTERED COLUMNSTORE INDEX CCI_fact_booking_line
    ON dm.fact_booking_line ON ps_date_key_month (booked_date_key);
CREATE UNIQUE INDEX UX_fact_booking_line_grain
    ON dm.fact_booking_line (booked_date_key, booking_item_id);

CREATE TABLE dm.fact_appointment (
    appointment_date_key INT      NOT NULL,
    slot_time_key        SMALLINT NOT NULL,
    customer_sk          BIGINT   NOT NULL,
    salon_sk             INT      NOT NULL,
    employee_sk          INT      NOT NULL,
    room_sk              INT      NOT NULL,
    booking_junk_sk      INT      NOT NULL,
    appointment_id       BIGINT   NOT NULL,          -- GRAIN
    booking_id           BIGINT   NULL,
    appointment_status   VARCHAR(20) NOT NULL,       -- scheduled/checked_in/no_show/cancelled
    scheduled_duration_min SMALLINT NOT NULL CONSTRAINT DF_fa_sdur DEFAULT (0),
    wait_time_min        SMALLINT NOT NULL CONSTRAINT DF_fa_wait  DEFAULT (0),
    appointment_count    TINYINT  NOT NULL CONSTRAINT DF_fa_cnt   DEFAULT (1),
    is_no_show           BIT      NOT NULL CONSTRAINT DF_fa_ns    DEFAULT (0),
    is_checked_in        BIT      NOT NULL CONSTRAINT DF_fa_ci    DEFAULT (0),
    _run_id              UNIQUEIDENTIFIER NOT NULL,
    _loaded_at           DATETIME2(3) NOT NULL CONSTRAINT DF_fa_loaded DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT CK_fact_appointment_status CHECK (appointment_status IN
        ('scheduled','checked_in','no_show','cancelled','completed')),
    CONSTRAINT CK_fact_appointment_wait   CHECK (wait_time_min >= 0),
    -- no_show và checked_in loại trừ nhau
    CONSTRAINT CK_fact_appointment_excl   CHECK (is_no_show = 0 OR is_checked_in = 0),
    CONSTRAINT FK_fact_appointment_dim_date     FOREIGN KEY (appointment_date_key) REFERENCES dm.dim_date(date_key),
    CONSTRAINT FK_fact_appointment_dim_time     FOREIGN KEY (slot_time_key)     REFERENCES dm.dim_time(time_key),
    CONSTRAINT FK_fact_appointment_dim_customer FOREIGN KEY (customer_sk)       REFERENCES dm.dim_customer(customer_sk),
    CONSTRAINT FK_fact_appointment_dim_salon    FOREIGN KEY (salon_sk)          REFERENCES dm.dim_salon(salon_sk),
    CONSTRAINT FK_fact_appointment_dim_employee FOREIGN KEY (employee_sk)       REFERENCES dm.dim_employee(employee_sk),
    CONSTRAINT FK_fact_appointment_dim_room     FOREIGN KEY (room_sk)           REFERENCES dm.dim_room(room_sk),
    CONSTRAINT FK_fact_appointment_dim_junk     FOREIGN KEY (booking_junk_sk)   REFERENCES dm.dim_booking_junk(booking_junk_sk)
) ON ps_date_key_month (appointment_date_key);
CREATE CLUSTERED COLUMNSTORE INDEX CCI_fact_appointment
    ON dm.fact_appointment ON ps_date_key_month (appointment_date_key);
CREATE UNIQUE INDEX UX_fact_appointment_grain
    ON dm.fact_appointment (appointment_date_key, appointment_id);

CREATE TABLE dm.fact_treatment (
    treatment_date_key INT      NOT NULL,
    start_time_key     SMALLINT NOT NULL,
    customer_sk        BIGINT   NOT NULL,
    salon_sk           INT      NOT NULL,
    employee_sk        INT      NOT NULL,
    room_sk            INT      NOT NULL,
    service_sk         INT      NOT NULL,
    promotion_sk       INT      NOT NULL,
    booking_junk_sk    INT      NOT NULL,
    treatment_id       BIGINT   NOT NULL,            -- GRAIN
    appointment_id     BIGINT   NULL,
    -- Measure: lưu TỬ SỐ và MẪU SỐ, không lưu tỷ lệ (mục 2.6)
    busy_minutes       SMALLINT NOT NULL CONSTRAINT DF_ft_busy  DEFAULT (0),
    available_minutes  SMALLINT NOT NULL CONSTRAINT DF_ft_avail DEFAULT (0),
    standard_minutes   SMALLINT NOT NULL CONSTRAINT DF_ft_std   DEFAULT (0),
    overrun_minutes    SMALLINT NOT NULL CONSTRAINT DF_ft_over  DEFAULT (0),
    product_cogs_amount DECIMAL(18,2) NOT NULL CONSTRAINT DF_ft_cogs DEFAULT (0),
    treatment_count    TINYINT  NOT NULL CONSTRAINT DF_ft_cnt   DEFAULT (1),
    is_upsell          BIT      NOT NULL CONSTRAINT DF_ft_ups   DEFAULT (0),  -- không có trong booking gốc
    _run_id            UNIQUEIDENTIFIER NOT NULL,
    _loaded_at         DATETIME2(3) NOT NULL CONSTRAINT DF_ft_loaded DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT CK_fact_treatment_duration CHECK (busy_minutes BETWEEN 0 AND 480),
    CONSTRAINT CK_fact_treatment_avail    CHECK (available_minutes >= 0),
    CONSTRAINT FK_fact_treatment_dim_date     FOREIGN KEY (treatment_date_key) REFERENCES dm.dim_date(date_key),
    CONSTRAINT FK_fact_treatment_dim_time     FOREIGN KEY (start_time_key)  REFERENCES dm.dim_time(time_key),
    CONSTRAINT FK_fact_treatment_dim_customer FOREIGN KEY (customer_sk)     REFERENCES dm.dim_customer(customer_sk),
    CONSTRAINT FK_fact_treatment_dim_salon    FOREIGN KEY (salon_sk)        REFERENCES dm.dim_salon(salon_sk),
    CONSTRAINT FK_fact_treatment_dim_employee FOREIGN KEY (employee_sk)     REFERENCES dm.dim_employee(employee_sk),
    CONSTRAINT FK_fact_treatment_dim_room     FOREIGN KEY (room_sk)         REFERENCES dm.dim_room(room_sk),
    CONSTRAINT FK_fact_treatment_dim_service  FOREIGN KEY (service_sk)      REFERENCES dm.dim_service(service_sk),
    CONSTRAINT FK_fact_treatment_dim_promo    FOREIGN KEY (promotion_sk)    REFERENCES dm.dim_promotion(promotion_sk),
    CONSTRAINT FK_fact_treatment_dim_junk     FOREIGN KEY (booking_junk_sk) REFERENCES dm.dim_booking_junk(booking_junk_sk)
) ON ps_date_key_month (treatment_date_key);
CREATE CLUSTERED COLUMNSTORE INDEX CCI_fact_treatment
    ON dm.fact_treatment ON ps_date_key_month (treatment_date_key);
CREATE UNIQUE INDEX UX_fact_treatment_grain
    ON dm.fact_treatment (treatment_date_key, treatment_id);

CREATE TABLE dm.fact_loyalty_txn (
    txn_date_key    INT      NOT NULL,
    customer_sk     BIGINT   NOT NULL,
    salon_sk        INT      NOT NULL,
    loyalty_txn_id  BIGINT   NOT NULL,               -- GRAIN
    txn_type        VARCHAR(20) NOT NULL,            -- earn / redeem / expire / adjust
    source_payment_id BIGINT NULL,
    -- Lưu BIẾN ĐỘNG (additive), không lưu SỐ DƯ (semi-additive) — mục 2.6
    point_delta     INT      NOT NULL CONSTRAINT DF_fl_delta DEFAULT (0),
    point_value_amount DECIMAL(18,2) NOT NULL CONSTRAINT DF_fl_val DEFAULT (0),
    _run_id         UNIQUEIDENTIFIER NOT NULL,
    _loaded_at      DATETIME2(3) NOT NULL CONSTRAINT DF_fl_loaded DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT CK_fact_loyalty_type CHECK (txn_type IN ('earn','redeem','expire','adjust')),
    -- Dấu của point_delta phải khớp loại giao dịch
    CONSTRAINT CK_fact_loyalty_sign CHECK (
        (txn_type = 'earn' AND point_delta > 0)
     OR (txn_type IN ('redeem','expire') AND point_delta < 0)
     OR (txn_type = 'adjust')),
    CONSTRAINT FK_fact_loyalty_dim_date     FOREIGN KEY (txn_date_key) REFERENCES dm.dim_date(date_key),
    CONSTRAINT FK_fact_loyalty_dim_customer FOREIGN KEY (customer_sk)  REFERENCES dm.dim_customer(customer_sk),
    CONSTRAINT FK_fact_loyalty_dim_salon    FOREIGN KEY (salon_sk)     REFERENCES dm.dim_salon(salon_sk)
) ON ps_date_key_month (txn_date_key);
CREATE CLUSTERED COLUMNSTORE INDEX CCI_fact_loyalty_txn
    ON dm.fact_loyalty_txn ON ps_date_key_month (txn_date_key);
CREATE UNIQUE INDEX UX_fact_loyalty_txn_grain
    ON dm.fact_loyalty_txn (txn_date_key, loyalty_txn_id);

CREATE TABLE dm.fact_feedback (
    feedback_date_key INT      NOT NULL,
    customer_sk       BIGINT   NOT NULL,
    salon_sk          INT      NOT NULL,
    employee_sk       INT      NOT NULL,
    service_sk        INT      NOT NULL,
    feedback_id       BIGINT   NOT NULL,             -- GRAIN
    treatment_id      BIGINT   NULL,
    feedback_channel  VARCHAR(20) NOT NULL,          -- app / sms / hotline
    -- CSAT trên thang 1-5 sao. rating là NON-ADDITIVE nên lưu kèm mẫu số.
    rating            TINYINT  NOT NULL,                                      -- 1..5
    response_count    TINYINT  NOT NULL CONSTRAINT DF_ff_rcnt DEFAULT (1),    -- mẫu số của mọi tỷ lệ
    is_satisfied      BIT      NOT NULL CONSTRAINT DF_ff_sat  DEFAULT (0),    -- rating >= 4  → tử số CSAT
    is_dissatisfied   BIT      NOT NULL CONSTRAINT DF_ff_dis  DEFAULT (0),    -- rating <= 2
    -- NPS dùng thang 0-10, là CÂU HỎI KHÁC, không suy ra được từ rating 1-5
    nps_score         TINYINT  NULL,                                          -- 0..10, NULL nếu không hỏi
    is_promoter       BIT      NULL,                                          -- nps_score 9-10
    is_detractor      BIT      NULL,                                          -- nps_score 0-6
    has_comment       BIT      NOT NULL CONSTRAINT DF_ff_cmt  DEFAULT (0),
    _run_id           UNIQUEIDENTIFIER NOT NULL,
    _loaded_at        DATETIME2(3) NOT NULL CONSTRAINT DF_ff_loaded DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT CK_fact_feedback_rating CHECK (rating BETWEEN 1 AND 5),
    CONSTRAINT CK_fact_feedback_nps    CHECK (nps_score IS NULL OR nps_score BETWEEN 0 AND 10),
    -- Cờ NPS chỉ tồn tại khi có điểm NPS
    CONSTRAINT CK_fact_feedback_nps_flag CHECK (
        (nps_score IS NULL     AND is_promoter IS NULL     AND is_detractor IS NULL)
     OR (nps_score IS NOT NULL AND is_promoter IS NOT NULL AND is_detractor IS NOT NULL)),
    CONSTRAINT FK_fact_feedback_dim_date     FOREIGN KEY (feedback_date_key) REFERENCES dm.dim_date(date_key),
    CONSTRAINT FK_fact_feedback_dim_customer FOREIGN KEY (customer_sk) REFERENCES dm.dim_customer(customer_sk),
    CONSTRAINT FK_fact_feedback_dim_salon    FOREIGN KEY (salon_sk)    REFERENCES dm.dim_salon(salon_sk),
    CONSTRAINT FK_fact_feedback_dim_employee FOREIGN KEY (employee_sk) REFERENCES dm.dim_employee(employee_sk),
    CONSTRAINT FK_fact_feedback_dim_service  FOREIGN KEY (service_sk)  REFERENCES dm.dim_service(service_sk)
) ON ps_date_key_month (feedback_date_key);
CREATE CLUSTERED COLUMNSTORE INDEX CCI_fact_feedback
    ON dm.fact_feedback ON ps_date_key_month (feedback_date_key);
CREATE UNIQUE INDEX UX_fact_feedback_grain
    ON dm.fact_feedback (feedback_date_key, feedback_id);

-- Fact có grain THÔ HƠN hẳn: không có customer. Xem cảnh báo drilling across ở mục 2.7.
CREATE TABLE dm.fact_ad_spend (
    spend_date_key   INT      NOT NULL,
    campaign_sk      INT      NOT NULL,
    promotion_sk     INT      NOT NULL,
    platform         VARCHAR(30) NOT NULL,
    spend_amount     DECIMAL(18,2) NOT NULL CONSTRAINT DF_fas_spend DEFAULT (0),
    impression_count BIGINT   NOT NULL CONSTRAINT DF_fas_imp   DEFAULT (0),
    click_count      BIGINT   NOT NULL CONSTRAINT DF_fas_click DEFAULT (0),
    lead_count       INT      NOT NULL CONSTRAINT DF_fas_lead  DEFAULT (0),
    _run_id          UNIQUEIDENTIFIER NOT NULL,
    _loaded_at       DATETIME2(3) NOT NULL CONSTRAINT DF_fas_loaded DEFAULT (SYSUTCDATETIME()),
    -- Grain khai báo tường minh bằng chính PK: 1 ngày × 1 chiến dịch × 1 nền tảng
    CONSTRAINT PK_fact_ad_spend PRIMARY KEY CLUSTERED (spend_date_key, campaign_sk, platform),
    CONSTRAINT FK_fact_ad_spend_dim_date
        FOREIGN KEY (spend_date_key) REFERENCES dm.dim_date(date_key),
    CONSTRAINT FK_fact_ad_spend_dim_campaign
        FOREIGN KEY (campaign_sk) REFERENCES dm.dim_campaign(campaign_sk),
    CONSTRAINT FK_fact_ad_spend_dim_promo
        FOREIGN KEY (promotion_sk) REFERENCES dm.dim_promotion(promotion_sk)
);
```

`fact_ad_spend` **không dùng columnstore và không phân vùng** — chỉ ~21.000 dòng/năm, rowstore clustered là lựa chọn đúng. Đừng áp columnstore cho bảng nhỏ: mỗi rowgroup cần tối thiểu 102.400 dòng để nén hiệu quả, dưới ngưỡng đó columnstore **chậm hơn** rowstore.

## 4. `fact_booking_lifecycle` — Accumulating Snapshot

Bảng **duy nhất** trong `dm` được phép `UPDATE`.

```sql
CREATE TABLE dm.fact_booking_lifecycle (
    booking_id          BIGINT   NOT NULL,           -- GRAIN: 1 dòng = 1 booking, cập nhật dần
    customer_sk         BIGINT   NOT NULL,
    salon_sk            INT      NOT NULL,
    employee_sk         INT      NOT NULL,
    service_sk          INT      NOT NULL,
    promotion_sk        INT      NOT NULL,
    campaign_sk         INT      NOT NULL,
    booking_junk_sk     INT      NOT NULL,
    -- ===== Các MỐC thời gian: NULL = CHƯA xảy ra (khác hẳn -1 = không xác định) =====
    booked_date_key     INT      NOT NULL,
    confirmed_date_key  INT      NULL,
    checkin_date_key    INT      NULL,
    treatment_date_key  INT      NULL,
    paid_date_key       INT      NULL,
    feedback_date_key   INT      NULL,
    cancelled_date_key  INT      NULL,
    -- ===== Khoảng cách giữa các mốc (giờ) — đây là giá trị thật của bảng này =====
    hours_book_to_confirm  DECIMAL(9,2) NULL,
    hours_confirm_to_visit DECIMAL(9,2) NULL,
    hours_checkin_to_start DECIMAL(9,2) NULL,
    hours_treat_to_pay     DECIMAL(9,2) NULL,
    hours_book_to_pay      DECIMAL(9,2) NULL,        -- tổng thời gian toàn chu trình
    -- ===== Cờ trạng thái phễu: cho phép tính funnel bằng SUM =====
    reached_confirmed   BIT NOT NULL CONSTRAINT DF_fbc_c  DEFAULT (0),
    reached_checkin     BIT NOT NULL CONSTRAINT DF_fbc_ci DEFAULT (0),
    reached_treatment   BIT NOT NULL CONSTRAINT DF_fbc_t  DEFAULT (0),
    reached_payment     BIT NOT NULL CONSTRAINT DF_fbc_p  DEFAULT (0),
    reached_feedback    BIT NOT NULL CONSTRAINT DF_fbc_f  DEFAULT (0),
    is_cancelled        BIT NOT NULL CONSTRAINT DF_fbc_x  DEFAULT (0),
    is_no_show          BIT NOT NULL CONSTRAINT DF_fbc_ns DEFAULT (0),
    net_amount          DECIMAL(18,2) NOT NULL CONSTRAINT DF_fbc_net DEFAULT (0),
    _run_id             UNIQUEIDENTIFIER NOT NULL,
    _updated_at         DATETIME2(3) NOT NULL CONSTRAINT DF_fbc_upd DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT PK_fact_booking_lifecycle PRIMARY KEY CLUSTERED (booking_id),
    -- Cờ phễu phải nhất quán với mốc thời gian tương ứng
    -- T-SQL không có kiểu boolean nên KHÔNG so sánh được hai vị từ với nhau
    -- (`(a = 1) = (b IS NOT NULL)` là lỗi cú pháp). Phải viết thành hai nhánh OR.
    CONSTRAINT CK_fbc_flag_confirm CHECK ((reached_confirmed = 0 AND confirmed_date_key IS NULL)
                                       OR (reached_confirmed = 1 AND confirmed_date_key IS NOT NULL)),
    CONSTRAINT CK_fbc_flag_checkin CHECK ((reached_checkin   = 0 AND checkin_date_key   IS NULL)
                                       OR (reached_checkin   = 1 AND checkin_date_key   IS NOT NULL)),
    CONSTRAINT CK_fbc_flag_treat   CHECK ((reached_treatment = 0 AND treatment_date_key IS NULL)
                                       OR (reached_treatment = 1 AND treatment_date_key IS NOT NULL)),
    CONSTRAINT CK_fbc_flag_pay     CHECK ((reached_payment   = 0 AND paid_date_key      IS NULL)
                                       OR (reached_payment   = 1 AND paid_date_key      IS NOT NULL)),
    CONSTRAINT CK_fbc_flag_cancel  CHECK ((is_cancelled      = 0 AND cancelled_date_key IS NULL)
                                       OR (is_cancelled      = 1 AND cancelled_date_key IS NOT NULL)),
    -- Huỷ và no-show loại trừ nhau; huỷ thì không thể đã thanh toán
    CONSTRAINT CK_fbc_excl CHECK (is_cancelled = 0 OR (is_no_show = 0 AND reached_payment = 0)),
    CONSTRAINT FK_fbc_dim_date     FOREIGN KEY (booked_date_key)    REFERENCES dm.dim_date(date_key),
    -- 6 mốc phễu là nullable; FK vẫn hợp lệ vì NULL luôn thoả ràng buộc khoá ngoại
    CONSTRAINT FK_fbc_dim_date_cf  FOREIGN KEY (confirmed_date_key) REFERENCES dm.dim_date(date_key),
    CONSTRAINT FK_fbc_dim_date_ci  FOREIGN KEY (checkin_date_key)   REFERENCES dm.dim_date(date_key),
    CONSTRAINT FK_fbc_dim_date_tr  FOREIGN KEY (treatment_date_key) REFERENCES dm.dim_date(date_key),
    CONSTRAINT FK_fbc_dim_date_py  FOREIGN KEY (paid_date_key)      REFERENCES dm.dim_date(date_key),
    CONSTRAINT FK_fbc_dim_date_fb  FOREIGN KEY (feedback_date_key)  REFERENCES dm.dim_date(date_key),
    CONSTRAINT FK_fbc_dim_date_cx  FOREIGN KEY (cancelled_date_key) REFERENCES dm.dim_date(date_key),
    CONSTRAINT FK_fbc_dim_customer FOREIGN KEY (customer_sk)     REFERENCES dm.dim_customer(customer_sk),
    CONSTRAINT FK_fbc_dim_salon    FOREIGN KEY (salon_sk)        REFERENCES dm.dim_salon(salon_sk),
    CONSTRAINT FK_fbc_dim_employee FOREIGN KEY (employee_sk)     REFERENCES dm.dim_employee(employee_sk),
    CONSTRAINT FK_fbc_dim_service  FOREIGN KEY (service_sk)      REFERENCES dm.dim_service(service_sk),
    CONSTRAINT FK_fbc_dim_promo    FOREIGN KEY (promotion_sk)    REFERENCES dm.dim_promotion(promotion_sk),
    CONSTRAINT FK_fbc_dim_campaign FOREIGN KEY (campaign_sk)     REFERENCES dm.dim_campaign(campaign_sk),
    CONSTRAINT FK_fbc_dim_junk     FOREIGN KEY (booking_junk_sk) REFERENCES dm.dim_booking_junk(booking_junk_sk)
);

CREATE INDEX IX_fact_booking_lifecycle_booked
    ON dm.fact_booking_lifecycle (booked_date_key)
    INCLUDE (reached_confirmed, reached_checkin, reached_treatment, reached_payment, is_cancelled, is_no_show);
```

Toàn bộ phễu chuyển đổi chỉ bằng một câu, không cần join:

```sql
SELECT booked_date_key,
       COUNT(*)                    AS booking_cnt,
       SUM(CAST(reached_confirmed AS INT)) AS confirmed_cnt,
       SUM(CAST(reached_checkin   AS INT)) AS checkin_cnt,
       SUM(CAST(reached_treatment AS INT)) AS treatment_cnt,
       SUM(CAST(reached_payment   AS INT)) AS paid_cnt,
       AVG(hours_book_to_pay)      AS avg_cycle_hours
FROM   dm.fact_booking_lifecycle
GROUP  BY booked_date_key;
```

> ⚠️ **Bảng này dùng rowstore clustered, KHÔNG dùng columnstore.** Columnstore rất kém với khối lượng `UPDATE` lớn — mỗi lần sửa là đánh dấu xoá dòng cũ rồi ghi dòng mới vào delta store, làm bảng tăng dung lượng và chậm dần. Accumulating snapshot bản chất là bảng bị UPDATE liên tục → rowstore là đúng.

## 5. `fact_customer_monthly_snapshot` — Periodic Snapshot

Giải bài toán semi-additive: chốt sẵn số dư điểm và trạng thái cuối mỗi tháng.

```sql
CREATE TABLE dm.fact_customer_monthly_snapshot (
    year_month          INT      NOT NULL,           -- 202608
    month_end_date_key  INT      NOT NULL,
    customer_sk         BIGINT   NOT NULL,
    salon_sk            INT      NOT NULL,           -- salon gần nhất khách đến
    -- Semi-additive: là giá trị CUỐI KỲ, không được SUM qua các tháng
    point_balance       INT      NOT NULL CONSTRAINT DF_fcms_pb   DEFAULT (0),
    membership_tier     VARCHAR(20) NOT NULL,
    days_since_last_visit SMALLINT NOT NULL CONSTRAINT DF_fcms_dslv DEFAULT (0),
    -- Additive trong phạm vi tháng
    visit_count_mtd     SMALLINT NOT NULL CONSTRAINT DF_fcms_vc DEFAULT (0),
    net_amount_mtd      DECIMAL(18,2) NOT NULL CONSTRAINT DF_fcms_amt DEFAULT (0),
    -- Additive: đếm đầu người, cộng được theo mọi chiều TRỪ thời gian
    is_active_customer  BIT      NOT NULL CONSTRAINT DF_fcms_act DEFAULT (0),  -- có đến trong 90 ngày
    is_churned          BIT      NOT NULL CONSTRAINT DF_fcms_ch  DEFAULT (0),
    _run_id             UNIQUEIDENTIFIER NOT NULL,
    _loaded_at          DATETIME2(3) NOT NULL CONSTRAINT DF_fcms_loaded DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT PK_fact_customer_monthly_snapshot
        PRIMARY KEY CLUSTERED (year_month, customer_sk),
    CONSTRAINT CK_fcms_excl CHECK (is_active_customer = 0 OR is_churned = 0),
    CONSTRAINT FK_fcms_dim_date     FOREIGN KEY (month_end_date_key) REFERENCES dm.dim_date(date_key),
    CONSTRAINT FK_fcms_dim_customer FOREIGN KEY (customer_sk) REFERENCES dm.dim_customer(customer_sk),
    CONSTRAINT FK_fcms_dim_salon    FOREIGN KEY (salon_sk)    REFERENCES dm.dim_salon(salon_sk)
);
```

> ⚠️ **Cảnh báo dùng sai đã được đưa vào chính tên cột:** `point_balance` là semi-additive. `SUM(point_balance)` qua nhiều tháng cho ra số vô nghĩa; phải lọc `WHERE year_month = 202608`. Ghi rõ điều này vào danh mục dữ liệu cho bảng này là **bắt buộc**, không phải khuyến nghị.

---
