# DDL — Schema `crt`, tầng đối soát

Tầng đối soát. Mô hình **3NF**, giữ đúng độ hạt của nguồn, chưa áp quy tắc nghiệp vụ. Đây là tầng dùng để phân xử khi số liệu lệch: nếu `crt` khớp POS thì lỗi nằm ở `dm`; nếu `crt` lệch POS thì lỗi nằm ở thu nạp.

Nguồn dữ liệu và phép biến đổi từng cột: [Source-to-Target Mapping](../02-mapping/source-to-target.md).

---

## Quy ước chung cho toàn schema

| Hạng mục | Quy tắc |
|---|---|
| Dạng chuẩn | 3NF — mỗi dữ kiện lưu ở đúng một chỗ |
| Khoá chính | Nghiệp vụ key của hệ thống nguồn, `CLUSTERED` |
| Khoá ngoại | Enforced. Thứ tự nạp: master trước, transaction sau |
| Kiểu dữ liệu | Đã ép đúng kiểu (khác `lnd` để rộng) |
| Timestamp | `DATETIME2(3)`, **UTC** |
| Cột khoá/mã | `COLLATE Latin1_General_100_BIN2` |
| Xoá | Xoá mềm bằng `_is_deleted`, không `DELETE` thật |

Cột kỹ thuật bắt buộc trên **mọi** bảng `crt`:

```sql
    _src_system  VARCHAR(20)      NOT NULL,   -- pos / oltp / app / gw / ads / ga4 / mkt / hr
    _run_id      UNIQUEIDENTIFIER NOT NULL,
    _loaded_at   DATETIME2(3)     NOT NULL,
    _updated_at  DATETIME2(3)     NOT NULL,
    _is_deleted  BIT              NOT NULL    -- 1 = nguồn báo DELETE, giữ lại để bảo toàn lịch sử
```

---

## 1. Master

### `crt.customer`

```sql
CREATE TABLE crt.customer (
    customer_id         BIGINT        NOT NULL,
    phone               VARCHAR(20)   COLLATE Latin1_General_100_BIN2 NULL,
    email               VARCHAR(255)  COLLATE Latin1_General_100_BIN2 NULL,
    full_name           NVARCHAR(200) NOT NULL,
    date_of_birth       DATE          NULL,
    gender              VARCHAR(10)   NOT NULL,
    registration_date   DATE          NOT NULL,
    acquisition_channel VARCHAR(50)   NOT NULL,
    first_salon_id      BIGINT        NOT NULL,
    status              VARCHAR(20)   NOT NULL,
    _src_system         VARCHAR(20)   NOT NULL,
    _run_id             UNIQUEIDENTIFIER NOT NULL,
    _loaded_at          DATETIME2(3)  NOT NULL CONSTRAINT DF_crt_customer_load DEFAULT (SYSUTCDATETIME()),
    _updated_at         DATETIME2(3)  NOT NULL CONSTRAINT DF_crt_customer_upd  DEFAULT (SYSUTCDATETIME()),
    _is_deleted         BIT           NOT NULL CONSTRAINT DF_crt_customer_del  DEFAULT (0),
    CONSTRAINT PK_crt_customer PRIMARY KEY CLUSTERED (customer_id),
    CONSTRAINT CK_crt_customer_gender CHECK (gender IN ('F','M','OTHER','UNKNOWN')),
    CONSTRAINT CK_crt_customer_status CHECK (status IN ('active','inactive','blacklisted')),
    CONSTRAINT CK_crt_customer_dob    CHECK (date_of_birth IS NULL
                                        OR date_of_birth BETWEEN '1920-01-01' AND CAST(SYSUTCDATETIME() AS DATE))
);

-- Tra cứu khi gộp định danh
CREATE INDEX IX_crt_customer_phone ON crt.customer (phone) WHERE phone IS NOT NULL;
CREATE INDEX IX_crt_customer_email ON crt.customer (email) WHERE email IS NOT NULL;
CREATE INDEX IX_crt_customer_run   ON crt.customer (_run_id);
```

### `crt.customer_identity_map`

```sql
CREATE TABLE crt.customer_identity_map (
    identity_id      BIGINT IDENTITY(1,1) NOT NULL,
    source_system    VARCHAR(30)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    source_id        VARCHAR(100)  COLLATE Latin1_General_100_BIN2 NOT NULL,
    match_key        VARCHAR(255)  COLLATE Latin1_General_100_BIN2 NULL,
    customer_id      BIGINT        NOT NULL,
    match_method     VARCHAR(30)   NOT NULL,
    match_confidence DECIMAL(3,2)  NOT NULL,
    reviewed_by      VARCHAR(100)  NULL,
    matched_at       DATETIME2(3)  NOT NULL CONSTRAINT DF_cim_matched DEFAULT (SYSUTCDATETIME()),
    _run_id          UNIQUEIDENTIFIER NOT NULL,
    CONSTRAINT PK_crt_customer_identity_map PRIMARY KEY CLUSTERED (identity_id),
    -- Một danh tính nguồn chỉ được trỏ tới đúng một khách
    CONSTRAINT UQ_crt_cim_source UNIQUE (source_system, source_id),
    CONSTRAINT CK_crt_cim_method CHECK (match_method IN ('exact_phone','exact_email','fuzzy_name_dob','manual')),
    CONSTRAINT CK_crt_cim_conf   CHECK (match_confidence BETWEEN 0 AND 1),
    CONSTRAINT FK_crt_cim_customer FOREIGN KEY (customer_id) REFERENCES crt.customer(customer_id)
);

CREATE INDEX IX_crt_cim_customer  ON crt.customer_identity_map (customer_id);
CREATE INDEX IX_crt_cim_match_key ON crt.customer_identity_map (match_key) WHERE match_key IS NOT NULL;
-- Danh sách chờ người rà: độ tin cậy thấp và chưa ai xem
CREATE INDEX IX_crt_cim_review    ON crt.customer_identity_map (match_confidence)
    WHERE reviewed_by IS NULL;
```

### `crt.salon`

```sql
CREATE TABLE crt.salon (
    salon_id      BIGINT        NOT NULL,
    salon_code    VARCHAR(20)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    salon_name    NVARCHAR(100) NOT NULL,
    city          NVARCHAR(50)  NOT NULL,
    district      NVARCHAR(50)  NOT NULL,
    address       NVARCHAR(255) NOT NULL,
    region        VARCHAR(20)   NOT NULL,
    capacity_beds TINYINT       NOT NULL,
    manager_id    BIGINT        NULL,
    open_date     DATE          NOT NULL,
    close_date    DATE          NULL,
    is_active     BIT           NOT NULL,
    _src_system   VARCHAR(20)   NOT NULL,
    _run_id       UNIQUEIDENTIFIER NOT NULL,
    _loaded_at    DATETIME2(3)  NOT NULL CONSTRAINT DF_crt_salon_load DEFAULT (SYSUTCDATETIME()),
    _updated_at   DATETIME2(3)  NOT NULL CONSTRAINT DF_crt_salon_upd  DEFAULT (SYSUTCDATETIME()),
    _is_deleted   BIT           NOT NULL CONSTRAINT DF_crt_salon_del  DEFAULT (0),
    CONSTRAINT PK_crt_salon PRIMARY KEY CLUSTERED (salon_id),
    CONSTRAINT UQ_crt_salon_code UNIQUE (salon_code),
    CONSTRAINT CK_crt_salon_beds   CHECK (capacity_beds > 0),
    CONSTRAINT CK_crt_salon_region CHECK (region IN ('Bắc','Trung','Nam','UNKNOWN')),
    CONSTRAINT CK_crt_salon_dates  CHECK (close_date IS NULL OR close_date >= open_date)
);
```

### `crt.employee`

```sql
CREATE TABLE crt.employee (
    employee_id    BIGINT        NOT NULL,
    employee_code  VARCHAR(20)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    employee_name  NVARCHAR(200) NOT NULL,
    role_name      VARCHAR(30)   NOT NULL,
    skill_level    VARCHAR(20)   NOT NULL,
    salon_id       BIGINT        NOT NULL,
    hire_date      DATE          NOT NULL,
    terminate_date DATE          NULL,
    is_active      BIT           NOT NULL,
    _src_system    VARCHAR(20)   NOT NULL,
    _run_id        UNIQUEIDENTIFIER NOT NULL,
    _loaded_at     DATETIME2(3)  NOT NULL CONSTRAINT DF_crt_emp_load DEFAULT (SYSUTCDATETIME()),
    _updated_at    DATETIME2(3)  NOT NULL CONSTRAINT DF_crt_emp_upd  DEFAULT (SYSUTCDATETIME()),
    _is_deleted    BIT           NOT NULL CONSTRAINT DF_crt_emp_del  DEFAULT (0),
    CONSTRAINT PK_crt_employee PRIMARY KEY CLUSTERED (employee_id),
    CONSTRAINT UQ_crt_employee_code UNIQUE (employee_code),
    CONSTRAINT CK_crt_employee_role  CHECK (role_name IN ('therapist','receptionist','manager','other')),
    CONSTRAINT CK_crt_employee_skill CHECK (skill_level IN ('Junior','Senior','Expert')),
    CONSTRAINT CK_crt_employee_dates CHECK (terminate_date IS NULL OR terminate_date >= hire_date),
    CONSTRAINT FK_crt_employee_salon FOREIGN KEY (salon_id) REFERENCES crt.salon(salon_id)
);

CREATE INDEX IX_crt_employee_salon ON crt.employee (salon_id) INCLUDE (role_name, is_active);
```

### `crt.room`

```sql
CREATE TABLE crt.room (
    room_id     BIGINT       NOT NULL,
    salon_id    BIGINT       NOT NULL,
    room_name   NVARCHAR(50) NOT NULL,
    room_type   VARCHAR(30)  NOT NULL,
    is_active   BIT          NOT NULL,
    _src_system VARCHAR(20)  NOT NULL,
    _run_id     UNIQUEIDENTIFIER NOT NULL,
    _loaded_at  DATETIME2(3) NOT NULL CONSTRAINT DF_crt_room_load DEFAULT (SYSUTCDATETIME()),
    _updated_at DATETIME2(3) NOT NULL CONSTRAINT DF_crt_room_upd  DEFAULT (SYSUTCDATETIME()),
    _is_deleted BIT          NOT NULL CONSTRAINT DF_crt_room_del  DEFAULT (0),
    CONSTRAINT PK_crt_room PRIMARY KEY CLUSTERED (room_id),
    CONSTRAINT CK_crt_room_type CHECK (room_type IN ('single','double','vip')),
    CONSTRAINT FK_crt_room_salon FOREIGN KEY (salon_id) REFERENCES crt.salon(salon_id)
);
```

### `crt.service`

```sql
CREATE TABLE crt.service (
    service_id            BIGINT        NOT NULL,
    service_code          VARCHAR(30)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    service_name          NVARCHAR(150) NOT NULL,
    category_l1           NVARCHAR(50)  NOT NULL,
    category_l2           NVARCHAR(50)  NOT NULL,
    standard_duration_min SMALLINT      NOT NULL,
    list_price_amount     DECIMAL(18,2) NOT NULL,
    is_signature          BIT           NOT NULL,
    is_active             BIT           NOT NULL,
    _src_system           VARCHAR(20)   NOT NULL,
    _run_id               UNIQUEIDENTIFIER NOT NULL,
    _loaded_at            DATETIME2(3)  NOT NULL CONSTRAINT DF_crt_svc_load DEFAULT (SYSUTCDATETIME()),
    _updated_at           DATETIME2(3)  NOT NULL CONSTRAINT DF_crt_svc_upd  DEFAULT (SYSUTCDATETIME()),
    _is_deleted           BIT           NOT NULL CONSTRAINT DF_crt_svc_del  DEFAULT (0),
    CONSTRAINT PK_crt_service PRIMARY KEY CLUSTERED (service_id),
    CONSTRAINT UQ_crt_service_code UNIQUE (service_code),
    CONSTRAINT CK_crt_service_duration CHECK (standard_duration_min BETWEEN 5 AND 480),
    CONSTRAINT CK_crt_service_price    CHECK (list_price_amount >= 0)
);
```

### `crt.product`

```sql
CREATE TABLE crt.product (
    product_id    BIGINT        NOT NULL,
    product_code  VARCHAR(30)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    product_name  NVARCHAR(150) NOT NULL,
    category      NVARCHAR(50)  NOT NULL,
    brand         NVARCHAR(50)  NOT NULL,
    unit          VARCHAR(20)   NOT NULL,
    retail_price  DECIMAL(18,2) NOT NULL,
    cost_price    DECIMAL(18,2) NOT NULL,
    is_retail     BIT           NOT NULL,
    is_consumable BIT           NOT NULL,
    is_active     BIT           NOT NULL,
    _src_system   VARCHAR(20)   NOT NULL,
    _run_id       UNIQUEIDENTIFIER NOT NULL,
    _loaded_at    DATETIME2(3)  NOT NULL CONSTRAINT DF_crt_prd_load DEFAULT (SYSUTCDATETIME()),
    _updated_at   DATETIME2(3)  NOT NULL CONSTRAINT DF_crt_prd_upd  DEFAULT (SYSUTCDATETIME()),
    _is_deleted   BIT           NOT NULL CONSTRAINT DF_crt_prd_del  DEFAULT (0),
    CONSTRAINT PK_crt_product PRIMARY KEY CLUSTERED (product_id),
    CONSTRAINT UQ_crt_product_code UNIQUE (product_code),
    CONSTRAINT CK_crt_product_price CHECK (retail_price >= 0 AND cost_price >= 0)
);
```

### `crt.promotion` và `crt.promotion_service`

```sql
CREATE TABLE crt.promotion (
    promotion_id    BIGINT        NOT NULL,
    promotion_code  VARCHAR(30)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    promotion_name  NVARCHAR(150) NOT NULL,
    promotion_type  VARCHAR(20)   NOT NULL,
    discount_value  DECIMAL(18,2) NOT NULL,
    valid_from_date DATE          NOT NULL,
    valid_to_date   DATE          NOT NULL,
    _src_system     VARCHAR(20)   NOT NULL,
    _run_id         UNIQUEIDENTIFIER NOT NULL,
    _loaded_at      DATETIME2(3)  NOT NULL CONSTRAINT DF_crt_promo_load DEFAULT (SYSUTCDATETIME()),
    _updated_at     DATETIME2(3)  NOT NULL CONSTRAINT DF_crt_promo_upd  DEFAULT (SYSUTCDATETIME()),
    _is_deleted     BIT           NOT NULL CONSTRAINT DF_crt_promo_del  DEFAULT (0),
    CONSTRAINT PK_crt_promotion PRIMARY KEY CLUSTERED (promotion_id),
    CONSTRAINT UQ_crt_promotion_code UNIQUE (promotion_code),
    CONSTRAINT CK_crt_promotion_type  CHECK (promotion_type IN ('percent','amount','gift','bundle','none')),
    CONSTRAINT CK_crt_promotion_dates CHECK (valid_to_date >= valid_from_date)
);

-- Bảng trung gian phá quan hệ N:N giữa promotion và service
CREATE TABLE crt.promotion_service (
    promotion_id BIGINT NOT NULL,
    service_id   BIGINT NOT NULL,
    _run_id      UNIQUEIDENTIFIER NOT NULL,
    CONSTRAINT PK_crt_promotion_service PRIMARY KEY CLUSTERED (promotion_id, service_id),
    CONSTRAINT FK_crt_ps_promotion FOREIGN KEY (promotion_id) REFERENCES crt.promotion(promotion_id),
    CONSTRAINT FK_crt_ps_service   FOREIGN KEY (service_id)   REFERENCES crt.service(service_id)
);
```

### `crt.invoice_line_promotion`

Bảng trung gian phá quan hệ nhiều-nhiều **dòng hoá đơn × khuyến mãi**: một dòng hoá đơn có thể được áp nhiều khuyến mãi cùng lúc (giảm giá theo hạng thẻ cộng với khuyến mãi mùa), và một khuyến mãi áp cho nhiều dòng. Đây là nguồn duy nhất của [`dm.bridge_sales_promotion`](05-svg-bi.md).

```sql
CREATE TABLE crt.invoice_line_promotion (
    invoice_line_id  BIGINT        NOT NULL,
    promotion_id     BIGINT        NOT NULL,
    -- Số tiền giảm giá mà POS gán cho đúng cặp này. Nếu POS chỉ trả tổng giảm giá
    -- trên cả dòng, cột này để NULL và ETL phân bổ theo tỷ lệ discount_pct.
    discount_amount  DECIMAL(18,2) NULL,
    _run_id          UNIQUEIDENTIFIER NOT NULL,
    _loaded_at       DATETIME2(3)  NOT NULL CONSTRAINT DF_crt_ilp_loaded DEFAULT (SYSUTCDATETIME()),
    _is_deleted      BIT           NOT NULL CONSTRAINT DF_crt_ilp_del DEFAULT (0),
    CONSTRAINT PK_crt_invoice_line_promotion
        PRIMARY KEY CLUSTERED (invoice_line_id, promotion_id),
    CONSTRAINT FK_crt_ilp_line      FOREIGN KEY (invoice_line_id) REFERENCES crt.invoice_line(invoice_line_id),
    CONSTRAINT FK_crt_ilp_promotion FOREIGN KEY (promotion_id)    REFERENCES crt.promotion(promotion_id)
);
```

> `crt.invoice_line.promotion_id` vẫn được giữ, mang **khuyến mãi chính** để truy vấn nhanh không phải join bảng này. Khi một dòng có nhiều khuyến mãi, phân tích đầy đủ phải đi qua `crt.invoice_line_promotion`, và tổng `discount_amount` của bảng này theo `invoice_line_id` phải bằng `invoice_line.discount_amount` — `DQ-ALLOC-002` kiểm điều đó.

---

### `crt.marketing_campaign`

```sql
CREATE TABLE crt.marketing_campaign (
    campaign_id   VARCHAR(100)  COLLATE Latin1_General_100_BIN2 NOT NULL,
    campaign_name NVARCHAR(200) NOT NULL,
    platform      VARCHAR(30)   NOT NULL,
    objective     VARCHAR(30)   NOT NULL,
    start_date    DATE          NOT NULL,
    end_date      DATE          NULL,
    budget_amount DECIMAL(18,2) NULL,
    _src_system   VARCHAR(20)   NOT NULL,
    _run_id       UNIQUEIDENTIFIER NOT NULL,
    _loaded_at    DATETIME2(3)  NOT NULL CONSTRAINT DF_crt_camp_load DEFAULT (SYSUTCDATETIME()),
    _updated_at   DATETIME2(3)  NOT NULL CONSTRAINT DF_crt_camp_upd  DEFAULT (SYSUTCDATETIME()),
    _is_deleted   BIT           NOT NULL CONSTRAINT DF_crt_camp_del  DEFAULT (0),
    CONSTRAINT PK_crt_marketing_campaign PRIMARY KEY CLUSTERED (campaign_id),
    CONSTRAINT CK_crt_campaign_platform CHECK (platform IN ('facebook','google','zalo','sms','email','other'))
);
```

---

## 2. Transaction

### `crt.booking` + `crt.booking_item`

```sql
CREATE TABLE crt.booking (
    booking_id        BIGINT        NOT NULL,
    customer_id       BIGINT        NOT NULL,
    salon_id          BIGINT        NOT NULL,
    booking_channel   VARCHAR(30)   NOT NULL,
    booking_status    VARCHAR(20)   NOT NULL,
    booked_at         DATETIME2(3)  NOT NULL,
    requested_slot_at DATETIME2(3)  NOT NULL,
    cancelled_at      DATETIME2(3)  NULL,
    cancel_reason     NVARCHAR(200) NULL,
    promotion_id      BIGINT        NULL,
    campaign_id       VARCHAR(100)  COLLATE Latin1_General_100_BIN2 NULL,
    source_event_id   UNIQUEIDENTIFIER NULL,
    _src_system       VARCHAR(20)   NOT NULL,
    _lsn              BIGINT        NULL,          -- LSN của CDC, dùng để khử trùng lặp
    _run_id           UNIQUEIDENTIFIER NOT NULL,
    _loaded_at        DATETIME2(3)  NOT NULL CONSTRAINT DF_crt_bkg_load DEFAULT (SYSUTCDATETIME()),
    _updated_at       DATETIME2(3)  NOT NULL CONSTRAINT DF_crt_bkg_upd  DEFAULT (SYSUTCDATETIME()),
    _is_deleted       BIT           NOT NULL CONSTRAINT DF_crt_bkg_del  DEFAULT (0),
    CONSTRAINT PK_crt_booking PRIMARY KEY CLUSTERED (booking_id),
    CONSTRAINT CK_crt_booking_channel CHECK (booking_channel IN ('app','web','hotline','walk_in','unknown')),
    CONSTRAINT CK_crt_booking_status  CHECK (booking_status IN
        ('created','confirmed','cancelled','completed','rescheduled')),
    -- Huỷ thì phải có mốc huỷ, và ngược lại
    CONSTRAINT CK_crt_booking_cancel  CHECK (
        (booking_status = 'cancelled' AND cancelled_at IS NOT NULL)
     OR (booking_status <> 'cancelled' AND cancelled_at IS NULL)),
    CONSTRAINT FK_crt_booking_customer  FOREIGN KEY (customer_id)  REFERENCES crt.customer(customer_id),
    CONSTRAINT FK_crt_booking_salon     FOREIGN KEY (salon_id)     REFERENCES crt.salon(salon_id),
    CONSTRAINT FK_crt_booking_promotion FOREIGN KEY (promotion_id) REFERENCES crt.promotion(promotion_id)
);

CREATE INDEX IX_crt_booking_customer ON crt.booking (customer_id, booked_at);
CREATE INDEX IX_crt_booking_booked   ON crt.booking (booked_at)  INCLUDE (salon_id, booking_status);
CREATE INDEX IX_crt_booking_slot     ON crt.booking (requested_slot_at);

CREATE TABLE crt.booking_item (
    booking_item_id BIGINT        NOT NULL,
    booking_id      BIGINT        NOT NULL,
    service_id      BIGINT        NOT NULL,
    quantity        DECIMAL(9,2)  NOT NULL,
    unit_price      DECIMAL(18,2) NOT NULL,
    discount_amount DECIMAL(18,2) NOT NULL,
    line_amount     DECIMAL(18,2) NOT NULL,
    _run_id         UNIQUEIDENTIFIER NOT NULL,
    _loaded_at      DATETIME2(3)  NOT NULL CONSTRAINT DF_crt_bki_load DEFAULT (SYSUTCDATETIME()),
    _is_deleted     BIT           NOT NULL CONSTRAINT DF_crt_bki_del  DEFAULT (0),
    CONSTRAINT PK_crt_booking_item PRIMARY KEY CLUSTERED (booking_item_id),
    CONSTRAINT CK_crt_booking_item_amt CHECK (quantity >= 0 AND unit_price >= 0 AND discount_amount >= 0),
    CONSTRAINT CK_crt_booking_item_line CHECK
        (ABS(line_amount - (quantity * unit_price - discount_amount)) < 0.01),
    CONSTRAINT FK_crt_bki_booking FOREIGN KEY (booking_id) REFERENCES crt.booking(booking_id),
    CONSTRAINT FK_crt_bki_service FOREIGN KEY (service_id) REFERENCES crt.service(service_id)
);

CREATE INDEX IX_crt_booking_item_booking ON crt.booking_item (booking_id);
```

### `crt.appointment`

```sql
CREATE TABLE crt.appointment (
    appointment_id         BIGINT       NOT NULL,
    booking_id             BIGINT       NULL,      -- NULL với khách walk-in
    customer_id            BIGINT       NOT NULL,
    salon_id               BIGINT       NOT NULL,
    employee_id            BIGINT       NOT NULL,
    room_id                BIGINT       NOT NULL,
    appointment_status     VARCHAR(20)  NOT NULL,
    slot_at                DATETIME2(3) NOT NULL,
    checkin_at             DATETIME2(3) NULL,      -- NULL = chưa đến
    scheduled_duration_min SMALLINT     NOT NULL,
    wait_time_min          SMALLINT     NOT NULL,
    is_no_show             BIT          NOT NULL,
    _src_system            VARCHAR(20)  NOT NULL,
    _run_id                UNIQUEIDENTIFIER NOT NULL,
    _loaded_at             DATETIME2(3) NOT NULL CONSTRAINT DF_crt_apt_load DEFAULT (SYSUTCDATETIME()),
    _updated_at            DATETIME2(3) NOT NULL CONSTRAINT DF_crt_apt_upd  DEFAULT (SYSUTCDATETIME()),
    _is_deleted            BIT          NOT NULL CONSTRAINT DF_crt_apt_del  DEFAULT (0),
    CONSTRAINT PK_crt_appointment PRIMARY KEY CLUSTERED (appointment_id),
    CONSTRAINT CK_crt_appointment_status CHECK (appointment_status IN
        ('scheduled','checked_in','no_show','cancelled','completed')),
    -- no_show và checkin loại trừ nhau
    CONSTRAINT CK_crt_appointment_noshow CHECK (is_no_show = 0 OR checkin_at IS NULL),
    CONSTRAINT CK_crt_appointment_wait   CHECK (wait_time_min >= 0),
    CONSTRAINT FK_crt_apt_booking  FOREIGN KEY (booking_id)  REFERENCES crt.booking(booking_id),
    CONSTRAINT FK_crt_apt_customer FOREIGN KEY (customer_id) REFERENCES crt.customer(customer_id),
    CONSTRAINT FK_crt_apt_salon    FOREIGN KEY (salon_id)    REFERENCES crt.salon(salon_id),
    CONSTRAINT FK_crt_apt_employee FOREIGN KEY (employee_id) REFERENCES crt.employee(employee_id),
    CONSTRAINT FK_crt_apt_room     FOREIGN KEY (room_id)     REFERENCES crt.room(room_id)
);

CREATE INDEX IX_crt_appointment_slot     ON crt.appointment (slot_at) INCLUDE (salon_id, is_no_show);
CREATE INDEX IX_crt_appointment_employee ON crt.appointment (employee_id, slot_at);
CREATE INDEX IX_crt_appointment_booking  ON crt.appointment (booking_id) WHERE booking_id IS NOT NULL;
```

### `crt.treatment` + `crt.treatment_product_usage`

```sql
CREATE TABLE crt.treatment (
    treatment_id      BIGINT       NOT NULL,
    appointment_id    BIGINT       NULL,
    customer_id       BIGINT       NOT NULL,
    salon_id          BIGINT       NOT NULL,
    employee_id       BIGINT       NOT NULL,
    room_id           BIGINT       NOT NULL,
    service_id        BIGINT       NOT NULL,
    promotion_id      BIGINT       NULL,
    started_at        DATETIME2(3) NOT NULL,
    completed_at      DATETIME2(3) NULL,
    busy_minutes      SMALLINT     NOT NULL,
    available_minutes SMALLINT     NOT NULL,
    standard_minutes  SMALLINT     NOT NULL,
    overrun_minutes   SMALLINT     NOT NULL,
    is_upsell         BIT          NOT NULL,
    _src_system       VARCHAR(20)  NOT NULL,
    _run_id           UNIQUEIDENTIFIER NOT NULL,
    _loaded_at        DATETIME2(3) NOT NULL CONSTRAINT DF_crt_trt_load DEFAULT (SYSUTCDATETIME()),
    _updated_at       DATETIME2(3) NOT NULL CONSTRAINT DF_crt_trt_upd  DEFAULT (SYSUTCDATETIME()),
    _is_deleted       BIT          NOT NULL CONSTRAINT DF_crt_trt_del  DEFAULT (0),
    CONSTRAINT PK_crt_treatment PRIMARY KEY CLUSTERED (treatment_id),
    CONSTRAINT CK_crt_treatment_minutes CHECK (busy_minutes BETWEEN 0 AND 480),
    CONSTRAINT CK_crt_treatment_time    CHECK (completed_at IS NULL OR completed_at >= started_at),
    CONSTRAINT FK_crt_trt_appointment FOREIGN KEY (appointment_id) REFERENCES crt.appointment(appointment_id),
    CONSTRAINT FK_crt_trt_customer    FOREIGN KEY (customer_id)    REFERENCES crt.customer(customer_id),
    CONSTRAINT FK_crt_trt_salon       FOREIGN KEY (salon_id)       REFERENCES crt.salon(salon_id),
    CONSTRAINT FK_crt_trt_employee    FOREIGN KEY (employee_id)    REFERENCES crt.employee(employee_id),
    CONSTRAINT FK_crt_trt_room        FOREIGN KEY (room_id)        REFERENCES crt.room(room_id),
    CONSTRAINT FK_crt_trt_service     FOREIGN KEY (service_id)     REFERENCES crt.service(service_id)
);

CREATE INDEX IX_crt_treatment_started  ON crt.treatment (started_at) INCLUDE (salon_id, service_id);
CREATE INDEX IX_crt_treatment_employee ON crt.treatment (employee_id, started_at)
    INCLUDE (busy_minutes, available_minutes);
CREATE INDEX IX_crt_treatment_customer ON crt.treatment (customer_id, started_at);

-- Vật tư tiêu hao trong buồng: nguồn của COGS dịch vụ
CREATE TABLE crt.treatment_product_usage (
    usage_id     BIGINT        NOT NULL,
    treatment_id BIGINT        NOT NULL,
    product_id   BIGINT        NOT NULL,
    quantity     DECIMAL(9,3)  NOT NULL,
    unit_cost    DECIMAL(18,2) NOT NULL,
    cost_amount  DECIMAL(18,2) NOT NULL,
    _run_id      UNIQUEIDENTIFIER NOT NULL,
    _loaded_at   DATETIME2(3)  NOT NULL CONSTRAINT DF_crt_tpu_load DEFAULT (SYSUTCDATETIME()),
    _is_deleted  BIT           NOT NULL CONSTRAINT DF_crt_tpu_del  DEFAULT (0),
    CONSTRAINT PK_crt_treatment_product_usage PRIMARY KEY CLUSTERED (usage_id),
    CONSTRAINT CK_crt_tpu_amt  CHECK (quantity >= 0 AND unit_cost >= 0),
    CONSTRAINT CK_crt_tpu_cost CHECK (ABS(cost_amount - quantity * unit_cost) < 0.01),
    CONSTRAINT FK_crt_tpu_treatment FOREIGN KEY (treatment_id) REFERENCES crt.treatment(treatment_id),
    CONSTRAINT FK_crt_tpu_product   FOREIGN KEY (product_id)   REFERENCES crt.product(product_id)
);

CREATE INDEX IX_crt_tpu_treatment ON crt.treatment_product_usage (treatment_id);
```

### `crt.invoice` + `crt.invoice_line`

Luồng doanh thu — bảng được đối soát với POS hằng ngày.

```sql
CREATE TABLE crt.invoice (
    invoice_id     BIGINT       NOT NULL,
    invoice_no     VARCHAR(30)  COLLATE Latin1_General_100_BIN2 NOT NULL,
    customer_id    BIGINT       NOT NULL,
    salon_id       BIGINT       NOT NULL,
    invoice_status VARCHAR(20)  NOT NULL,
    service_at     DATETIME2(3) NOT NULL,   -- CỘT CHỐT KỲ DOANH THU
    invoiced_at    DATETIME2(3) NOT NULL,
    campaign_id    VARCHAR(100) COLLATE Latin1_General_100_BIN2 NULL,
    _src_system    VARCHAR(20)  NOT NULL,
    _run_id        UNIQUEIDENTIFIER NOT NULL,
    _loaded_at     DATETIME2(3) NOT NULL CONSTRAINT DF_crt_inv_load DEFAULT (SYSUTCDATETIME()),
    _updated_at    DATETIME2(3) NOT NULL CONSTRAINT DF_crt_inv_upd  DEFAULT (SYSUTCDATETIME()),
    _is_deleted    BIT          NOT NULL CONSTRAINT DF_crt_inv_del  DEFAULT (0),
    CONSTRAINT PK_crt_invoice PRIMARY KEY CLUSTERED (invoice_id),
    CONSTRAINT UQ_crt_invoice_no UNIQUE (invoice_no),
    CONSTRAINT CK_crt_invoice_status CHECK (invoice_status IN ('paid','unpaid','void','partial')),
    CONSTRAINT FK_crt_invoice_customer FOREIGN KEY (customer_id) REFERENCES crt.customer(customer_id),
    CONSTRAINT FK_crt_invoice_salon    FOREIGN KEY (salon_id)    REFERENCES crt.salon(salon_id)
);

-- Phục vụ đối soát doanh thu theo ngày dịch vụ × salon
CREATE INDEX IX_crt_invoice_service_at ON crt.invoice (service_at, salon_id)
    INCLUDE (invoice_status, customer_id);
CREATE INDEX IX_crt_invoice_customer   ON crt.invoice (customer_id, service_at);

CREATE TABLE crt.invoice_line (
    invoice_line_id        BIGINT        NOT NULL,   -- GRAIN
    invoice_id             BIGINT        NOT NULL,
    invoice_line_no        SMALLINT      NOT NULL,
    line_type              VARCHAR(10)   NOT NULL,   -- service / product
    service_id             BIGINT        NULL,
    product_id             BIGINT        NULL,
    treatment_id           BIGINT        NULL,
    employee_id            BIGINT        NOT NULL,
    promotion_id           BIGINT        NULL,
    quantity               DECIMAL(9,2)  NOT NULL,
    unit_price_amount      DECIMAL(18,2) NOT NULL,
    gross_amount           DECIMAL(18,2) NOT NULL,
    promo_discount_amount  DECIMAL(18,2) NOT NULL,
    member_discount_amount DECIMAL(18,2) NOT NULL,
    discount_amount        DECIMAL(18,2) NOT NULL,
    net_amount             DECIMAL(18,2) NOT NULL,
    tax_amount             DECIMAL(18,2) NOT NULL,
    net_excl_tax_amount    DECIMAL(18,2) NOT NULL,
    cogs_amount            DECIMAL(18,2) NOT NULL,
    gross_margin_amount    DECIMAL(18,2) NOT NULL,
    _run_id                UNIQUEIDENTIFIER NOT NULL,
    _loaded_at             DATETIME2(3)  NOT NULL CONSTRAINT DF_crt_invl_load DEFAULT (SYSUTCDATETIME()),
    _is_deleted            BIT           NOT NULL CONSTRAINT DF_crt_invl_del  DEFAULT (0),
    CONSTRAINT PK_crt_invoice_line PRIMARY KEY CLUSTERED (invoice_line_id),
    CONSTRAINT UQ_crt_invoice_line_seq UNIQUE (invoice_id, invoice_line_no),
    CONSTRAINT CK_crt_invl_type CHECK (line_type IN ('service','product')),
    -- Dòng dịch vụ phải có service_id; dòng sản phẩm phải có product_id
    CONSTRAINT CK_crt_invl_item CHECK (
        (line_type = 'service' AND service_id IS NOT NULL AND product_id IS NULL)
     OR (line_type = 'product' AND product_id IS NOT NULL AND service_id IS NULL)),
    CONSTRAINT CK_crt_invl_nonneg CHECK (quantity >= 0 AND unit_price_amount >= 0
                                     AND promo_discount_amount >= 0 AND member_discount_amount >= 0),
    CONSTRAINT CK_crt_invl_gross  CHECK (ABS(gross_amount - quantity * unit_price_amount) < 0.01),
    CONSTRAINT CK_crt_invl_disc   CHECK (ABS(discount_amount
                                    - (promo_discount_amount + member_discount_amount)) < 0.01),
    CONSTRAINT CK_crt_invl_net    CHECK (ABS(net_amount - (gross_amount - discount_amount)) < 0.01),
    CONSTRAINT CK_crt_invl_margin CHECK (ABS(gross_margin_amount - (net_amount - cogs_amount)) < 0.01),
    CONSTRAINT FK_crt_invl_invoice   FOREIGN KEY (invoice_id)   REFERENCES crt.invoice(invoice_id),
    CONSTRAINT FK_crt_invl_service   FOREIGN KEY (service_id)   REFERENCES crt.service(service_id),
    CONSTRAINT FK_crt_invl_product   FOREIGN KEY (product_id)   REFERENCES crt.product(product_id),
    CONSTRAINT FK_crt_invl_treatment FOREIGN KEY (treatment_id) REFERENCES crt.treatment(treatment_id),
    CONSTRAINT FK_crt_invl_employee  FOREIGN KEY (employee_id)  REFERENCES crt.employee(employee_id)
);

CREATE INDEX IX_crt_invoice_line_invoice_id ON crt.invoice_line (invoice_id);
CREATE INDEX IX_crt_invoice_line_treatment  ON crt.invoice_line (treatment_id)
    WHERE treatment_id IS NOT NULL;
```

> Bốn `CHECK` về số tiền (`gross`, `disc`, `net`, `margin`) là hàng rào chặn lỗi tính toán ngay tại tầng `crt`, trước khi số liệu đi vào kho phân tích. Dòng vi phạm bị đẩy sang [`qtn.reject_row`](06-ctl-qtn.md).

### `crt.payment` + `crt.payment_allocation`

```sql
CREATE TABLE crt.payment (
    payment_id          BIGINT        NOT NULL,
    invoice_id          BIGINT        NOT NULL,
    customer_id         BIGINT        NOT NULL,
    salon_id            BIGINT        NOT NULL,
    payment_method_code VARCHAR(30)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    payment_status      VARCHAR(20)   NOT NULL,
    payment_amount      DECIMAL(18,2) NOT NULL,
    refund_amount       DECIMAL(18,2) NOT NULL,
    fee_amount          DECIMAL(18,2) NOT NULL,
    net_cash_amount     DECIMAL(18,2) NOT NULL,
    gateway_txn_id      VARCHAR(100)  COLLATE Latin1_General_100_BIN2 NULL,
    error_code          VARCHAR(50)   NULL,
    paid_at             DATETIME2(3)  NOT NULL,
    _src_system         VARCHAR(20)   NOT NULL,
    _run_id             UNIQUEIDENTIFIER NOT NULL,
    _loaded_at          DATETIME2(3)  NOT NULL CONSTRAINT DF_crt_pay_load DEFAULT (SYSUTCDATETIME()),
    _updated_at         DATETIME2(3)  NOT NULL CONSTRAINT DF_crt_pay_upd  DEFAULT (SYSUTCDATETIME()),
    _is_deleted         BIT           NOT NULL CONSTRAINT DF_crt_pay_del  DEFAULT (0),
    CONSTRAINT PK_crt_payment PRIMARY KEY CLUSTERED (payment_id),
    CONSTRAINT CK_crt_payment_status CHECK (payment_status IN ('completed','failed','refunded','pending')),
    CONSTRAINT CK_crt_payment_amt    CHECK (payment_amount >= 0 AND refund_amount >= 0 AND fee_amount >= 0),
    CONSTRAINT FK_crt_payment_invoice  FOREIGN KEY (invoice_id)  REFERENCES crt.invoice(invoice_id),
    CONSTRAINT FK_crt_payment_customer FOREIGN KEY (customer_id) REFERENCES crt.customer(customer_id),
    CONSTRAINT FK_crt_payment_salon    FOREIGN KEY (salon_id)    REFERENCES crt.salon(salon_id)
);

CREATE INDEX IX_crt_payment_paid_at ON crt.payment (paid_at, salon_id) INCLUDE (payment_status);
CREATE INDEX IX_crt_payment_invoice ON crt.payment (invoice_id);
-- Phục vụ đối soát với cổng thanh toán
CREATE INDEX IX_crt_payment_gateway ON crt.payment (gateway_txn_id) WHERE gateway_txn_id IS NOT NULL;

-- Giải quan hệ N:N khi một lần trả phân bổ cho nhiều hoá đơn, hoặc một hoá đơn trả nhiều lần
CREATE TABLE crt.payment_allocation (
    alloc_id          BIGINT        NOT NULL,
    payment_id        BIGINT        NOT NULL,
    invoice_id        BIGINT        NOT NULL,
    allocated_amount  DECIMAL(18,2) NOT NULL,
    _run_id           UNIQUEIDENTIFIER NOT NULL,
    CONSTRAINT PK_crt_payment_allocation PRIMARY KEY CLUSTERED (alloc_id),
    CONSTRAINT UQ_crt_pa_pair UNIQUE (payment_id, invoice_id),
    CONSTRAINT CK_crt_pa_amt  CHECK (allocated_amount > 0),
    CONSTRAINT FK_crt_pa_payment FOREIGN KEY (payment_id) REFERENCES crt.payment(payment_id),
    CONSTRAINT FK_crt_pa_invoice FOREIGN KEY (invoice_id) REFERENCES crt.invoice(invoice_id)
);
```

### `crt.loyalty_transaction`

```sql
CREATE TABLE crt.loyalty_transaction (
    loyalty_txn_id     BIGINT        NOT NULL,
    customer_id        BIGINT        NOT NULL,
    salon_id           BIGINT        NOT NULL,
    txn_type           VARCHAR(20)   NOT NULL,
    point_delta        INT           NOT NULL,   -- âm với redeem/expire
    point_value_amount DECIMAL(18,2) NOT NULL,
    source_payment_id  BIGINT        NULL,
    txn_at             DATETIME2(3)  NOT NULL,
    _src_system        VARCHAR(20)   NOT NULL,
    _run_id            UNIQUEIDENTIFIER NOT NULL,
    _loaded_at         DATETIME2(3)  NOT NULL CONSTRAINT DF_crt_loy_load DEFAULT (SYSUTCDATETIME()),
    _is_deleted        BIT           NOT NULL CONSTRAINT DF_crt_loy_del  DEFAULT (0),
    CONSTRAINT PK_crt_loyalty_transaction PRIMARY KEY CLUSTERED (loyalty_txn_id),
    CONSTRAINT CK_crt_loy_type CHECK (txn_type IN ('earn','redeem','expire','adjust')),
    -- Dấu của point_delta phải khớp loại giao dịch
    CONSTRAINT CK_crt_loy_sign CHECK (
        (txn_type = 'earn'   AND point_delta > 0)
     OR (txn_type IN ('redeem','expire') AND point_delta < 0)
     OR (txn_type = 'adjust')),
    CONSTRAINT FK_crt_loy_customer FOREIGN KEY (customer_id)       REFERENCES crt.customer(customer_id),
    CONSTRAINT FK_crt_loy_salon    FOREIGN KEY (salon_id)          REFERENCES crt.salon(salon_id),
    CONSTRAINT FK_crt_loy_payment  FOREIGN KEY (source_payment_id) REFERENCES crt.payment(payment_id)
);

CREATE INDEX IX_crt_loyalty_customer ON crt.loyalty_transaction (customer_id, txn_at)
    INCLUDE (point_delta);
```

### `crt.membership_subscription`

```sql
CREATE TABLE crt.membership_subscription (
    subscription_id BIGINT        NOT NULL,
    customer_id     BIGINT        NOT NULL,
    tier_code       VARCHAR(20)   NOT NULL,
    valid_from      DATE          NOT NULL,
    valid_to        DATE          NOT NULL,
    purchase_amount DECIMAL(18,2) NOT NULL,
    purchased_at    DATETIME2(3)  NOT NULL,
    _src_system     VARCHAR(20)   NOT NULL,
    _run_id         UNIQUEIDENTIFIER NOT NULL,
    _loaded_at      DATETIME2(3)  NOT NULL CONSTRAINT DF_crt_mem_load DEFAULT (SYSUTCDATETIME()),
    _is_deleted     BIT           NOT NULL CONSTRAINT DF_crt_mem_del  DEFAULT (0),
    CONSTRAINT PK_crt_membership_subscription PRIMARY KEY CLUSTERED (subscription_id),
    CONSTRAINT CK_crt_mem_dates CHECK (valid_to >= valid_from),
    CONSTRAINT FK_crt_mem_customer FOREIGN KEY (customer_id) REFERENCES crt.customer(customer_id)
);

CREATE INDEX IX_crt_membership_customer ON crt.membership_subscription (customer_id, valid_from, valid_to);
```

> Kỳ thành viên của cùng một khách **không được chồng nhau**. Ràng buộc này không biểu diễn được bằng `CHECK`; kiểm tra bằng [DQ-MEM-001](../05-quality/dq-rules.md).

### `crt.feedback`

```sql
CREATE TABLE crt.feedback (
    feedback_id      BIGINT         NOT NULL,
    customer_id      BIGINT         NOT NULL,
    treatment_id     BIGINT         NULL,
    salon_id         BIGINT         NOT NULL,
    employee_id      BIGINT         NOT NULL,
    service_id       BIGINT         NOT NULL,
    feedback_channel VARCHAR(20)    NOT NULL,
    rating           TINYINT        NOT NULL,   -- 1..5, dùng cho CSAT
    is_satisfied     BIT            NOT NULL,
    is_dissatisfied  BIT            NOT NULL,
    nps_score        TINYINT        NULL,       -- 0..10, câu hỏi riêng
    is_promoter      BIT            NULL,
    is_detractor     BIT            NULL,
    comment_text     NVARCHAR(2000) NULL,       -- KHÔNG đưa lên dm
    has_comment      BIT            NOT NULL,
    feedback_at      DATETIME2(3)   NOT NULL,
    _src_system      VARCHAR(20)    NOT NULL,
    _run_id          UNIQUEIDENTIFIER NOT NULL,
    _loaded_at       DATETIME2(3)   NOT NULL CONSTRAINT DF_crt_fb_load DEFAULT (SYSUTCDATETIME()),
    _is_deleted      BIT            NOT NULL CONSTRAINT DF_crt_fb_del  DEFAULT (0),
    CONSTRAINT PK_crt_feedback PRIMARY KEY CLUSTERED (feedback_id),
    CONSTRAINT CK_crt_fb_channel CHECK (feedback_channel IN ('app','sms','hotline','web')),
    CONSTRAINT CK_crt_fb_rating  CHECK (rating BETWEEN 1 AND 5),
    CONSTRAINT CK_crt_fb_nps     CHECK (nps_score IS NULL OR nps_score BETWEEN 0 AND 10),
    -- Cờ NPS chỉ tồn tại khi có điểm NPS
    CONSTRAINT CK_crt_fb_nps_flag CHECK (
        (nps_score IS NULL     AND is_promoter IS NULL     AND is_detractor IS NULL)
     OR (nps_score IS NOT NULL AND is_promoter IS NOT NULL AND is_detractor IS NOT NULL)),
    CONSTRAINT FK_crt_fb_customer  FOREIGN KEY (customer_id)  REFERENCES crt.customer(customer_id),
    CONSTRAINT FK_crt_fb_treatment FOREIGN KEY (treatment_id) REFERENCES crt.treatment(treatment_id),
    CONSTRAINT FK_crt_fb_salon     FOREIGN KEY (salon_id)     REFERENCES crt.salon(salon_id),
    CONSTRAINT FK_crt_fb_employee  FOREIGN KEY (employee_id)  REFERENCES crt.employee(employee_id),
    CONSTRAINT FK_crt_fb_service   FOREIGN KEY (service_id)   REFERENCES crt.service(service_id)
);

-- Khử trùng lặp: một lượt điều trị chỉ nhận một phiếu
CREATE UNIQUE INDEX UX_crt_feedback_treatment
    ON crt.feedback (treatment_id) WHERE treatment_id IS NOT NULL AND _is_deleted = 0;
CREATE INDEX IX_crt_feedback_at ON crt.feedback (feedback_at, salon_id) INCLUDE (rating);
```

### `crt.ad_spend` và `crt.campaign_send`

```sql
CREATE TABLE crt.ad_spend (
    spend_date       DATE          NOT NULL,
    campaign_id      VARCHAR(100)  COLLATE Latin1_General_100_BIN2 NOT NULL,
    platform         VARCHAR(30)   NOT NULL,
    spend_amount     DECIMAL(18,2) NOT NULL,
    impression_count BIGINT        NOT NULL,
    click_count      BIGINT        NOT NULL,
    lead_count       INT           NOT NULL,
    _src_system      VARCHAR(20)   NOT NULL,
    _run_id          UNIQUEIDENTIFIER NOT NULL,
    _loaded_at       DATETIME2(3)  NOT NULL CONSTRAINT DF_crt_ads_load DEFAULT (SYSUTCDATETIME()),
    -- Grain khai báo bằng chính PK
    CONSTRAINT PK_crt_ad_spend PRIMARY KEY CLUSTERED (spend_date, campaign_id, platform),
    CONSTRAINT CK_crt_ads_nonneg CHECK (spend_amount >= 0 AND impression_count >= 0
                                    AND click_count >= 0 AND lead_count >= 0),
    -- Số click không thể vượt số lần hiển thị
    CONSTRAINT CK_crt_ads_ctr CHECK (click_count <= impression_count),
    CONSTRAINT FK_crt_ads_campaign FOREIGN KEY (campaign_id)
        REFERENCES crt.marketing_campaign(campaign_id)
);

CREATE TABLE crt.campaign_send (
    send_id     BIGINT       NOT NULL,
    campaign_id VARCHAR(100) COLLATE Latin1_General_100_BIN2 NOT NULL,
    customer_id BIGINT       NOT NULL,
    channel     VARCHAR(20)  NOT NULL,
    sent_at     DATETIME2(3) NOT NULL,
    opened_at   DATETIME2(3) NULL,
    clicked_at  DATETIME2(3) NULL,
    _src_system VARCHAR(20)  NOT NULL,
    _run_id     UNIQUEIDENTIFIER NOT NULL,
    _loaded_at  DATETIME2(3) NOT NULL CONSTRAINT DF_crt_cs_load DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT PK_crt_campaign_send PRIMARY KEY CLUSTERED (send_id),
    CONSTRAINT CK_crt_cs_channel CHECK (channel IN ('email','sms','zalo','push')),
    -- Thứ tự mốc: gửi → mở → bấm
    CONSTRAINT CK_crt_cs_order CHECK (
        (opened_at  IS NULL OR opened_at  >= sent_at)
    AND (clicked_at IS NULL OR (opened_at IS NOT NULL AND clicked_at >= opened_at))),
    CONSTRAINT FK_crt_cs_campaign FOREIGN KEY (campaign_id)
        REFERENCES crt.marketing_campaign(campaign_id),
    CONSTRAINT FK_crt_cs_customer FOREIGN KEY (customer_id) REFERENCES crt.customer(customer_id)
);

CREATE INDEX IX_crt_cs_campaign ON crt.campaign_send (campaign_id, sent_at);
CREATE INDEX IX_crt_cs_customer ON crt.campaign_send (customer_id, sent_at);
```

### `crt.service_view`

```sql
CREATE TABLE crt.service_view (
    view_id      BIGINT       NOT NULL,
    session_id   VARCHAR(100) COLLATE Latin1_General_100_BIN2 NOT NULL,
    customer_id  BIGINT       NOT NULL,   -- -1 khi khách chưa đăng nhập
    service_id   BIGINT       NOT NULL,
    duration_sec INT          NOT NULL,
    viewed_at    DATETIME2(3) NOT NULL,
    _src_system  VARCHAR(20)  NOT NULL,
    _run_id      UNIQUEIDENTIFIER NOT NULL,
    _loaded_at   DATETIME2(3) NOT NULL CONSTRAINT DF_crt_sv_load DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT PK_crt_service_view PRIMARY KEY CLUSTERED (view_id),
    CONSTRAINT CK_crt_sv_duration CHECK (duration_sec >= 0)
);

CREATE INDEX IX_crt_sv_viewed  ON crt.service_view (viewed_at) INCLUDE (service_id);
CREATE INDEX IX_crt_sv_session ON crt.service_view (session_id);
```

> Không đặt FK tới `crt.customer` vì `customer_id = -1` với khách chưa đăng nhập. Toàn vẹn kiểm bằng DQ quy tắc.

---

## 3. View phục vụ nạp `dm`

### `crt.v_customer_for_dim`

View này được [`dm.usp_load_dim_customer`](../04-etl/load-dimension.md) gọi tới.

```sql
CREATE OR ALTER VIEW crt.v_customer_for_dim AS
SELECT
    c.customer_id,
    c.full_name,
    -- Che số điện thoại: chỉ dm mới dùng bản che, bản đầy đủ giữ ở crt
    -- Che 4 số giữa: 0901234567 -> 090****567 (khớp ví dụ ở 03-dm-dimension.md)
    CASE WHEN c.phone IS NULL THEN 'N/A'
         ELSE LEFT(c.phone, 3) + '****' + RIGHT(c.phone, 3) END      AS phone_masked,
    c.gender,
    -- DATEDIFF(YEAR,...) đếm mốc năm, không phải tuổi thật: sinh 31/12/2001 thì
    -- ngày 01/01/2026 ra 25 thay vì 24, và cả tập khách nhảy nhóm tuổi vào 01/01.
    CASE WHEN c.date_of_birth IS NULL THEN 'UNKNOWN'
         WHEN DATEDIFF(DAY, c.date_of_birth, SYSUTCDATETIME()) / 365.25 < 25 THEN '<25'
         WHEN DATEDIFF(DAY, c.date_of_birth, SYSUTCDATETIME()) / 365.25 < 35 THEN '25-34'
         WHEN DATEDIFF(DAY, c.date_of_birth, SYSUTCDATETIME()) / 365.25 < 45 THEN '35-44'
         ELSE '45+' END                                              AS age_group,
    ISNULL(s.city, N'(Không xác định)')                              AS city,
    -- Hạng thẻ tại thời điểm chạy, suy từ kỳ thành viên đang hiệu lực
    ISNULL(m.tier_code, 'None')                                      AS membership_tier,
    c.acquisition_channel,
    -- View này CHỈ đọc schema `crt`. Trước đây nó join `dm.dim_salon` và
    -- `svg_bi.agg_customer_360`; vì `CREATE VIEW` không có deferred name resolution,
    -- tạo view ở bước 03 khi hai schema kia chưa tồn tại sẽ lỗi Msg 208. Đồng thời
    -- crt phụ thuộc svg_bi tạo vòng: dim_customer <- view <- agg_customer_360 <- fact <- dim_customer.
    -- Khoá đại diện `first_salon_sk` và `rfm_segment` được giải ở thủ tục nạp dim.
    c.first_salon_id,
    c._is_deleted
FROM       crt.customer c
LEFT JOIN  crt.salon    s  ON s.salon_id = c.first_salon_id
OUTER APPLY (
    SELECT TOP (1) ms.tier_code
    FROM   crt.membership_subscription ms
    WHERE  ms.customer_id = c.customer_id
      AND  CAST(SYSUTCDATETIME() AS DATE) BETWEEN ms.valid_from AND ms.valid_to
      AND  ms._is_deleted = 0
    ORDER BY ms.valid_from DESC
) m;
```

---

## Thứ tự tạo bảng

FK enforced nên phải tạo theo đúng thứ tự sau:

```
1. salon
2. customer  →  customer_identity_map
3. employee, room                      (phụ thuộc salon)
4. service, product, promotion, marketing_campaign
5. promotion_service                   (phụ thuộc promotion, service)
6. booking  →  booking_item
7. appointment                         (phụ thuộc booking, employee, room)
8. treatment  →  treatment_product_usage
9. invoice  →  invoice_line
10. payment  →  payment_allocation
11. loyalty_transaction                (phụ thuộc payment)
12. membership_subscription, feedback, ad_spend, campaign_send, service_view
14. v_customer_for_dim                 (view, tạo sau cùng)
```

Thứ tự **xoá** khi dựng lại: đảo ngược danh sách trên.

## Danh mục bảng

| # | Bảng | Loại | độ hạt | Nguồn chính |
|---|---|---|---|---|
| 1 | `crt.customer` | Master | 1 khách đã gộp định danh | OLTP + POS |
| 2 | `crt.customer_identity_map` | Master | 1 danh tính ở 1 hệ thống | Nhiều nguồn |
| 3 | `crt.salon` | Master | 1 chi nhánh | POS |
| 4 | `crt.employee` | Master | 1 nhân viên | HR |
| 5 | `crt.room` | Master | 1 buồng | POS |
| 6 | `crt.service` | Master | 1 dịch vụ | POS |
| 7 | `crt.product` | Master | 1 sản phẩm | POS |
| 8 | `crt.promotion` | Master | 1 khuyến mãi | POS |
| 9 | `crt.promotion_service` | Bridge | 1 cặp khuyến mãi × dịch vụ | POS |
| 10 | `crt.marketing_campaign` | Master | 1 chiến dịch | ADS + MKT |
| 11 | `crt.booking` | Transaction | 1 lần đặt lịch | OLTP (CDC) |
| 12 | `crt.booking_item` | Transaction | 1 dịch vụ trong 1 booking | OLTP (CDC) |
| 13 | `crt.appointment` | Transaction | 1 lịch hẹn | POS |
| 14 | `crt.treatment` | Transaction | 1 dịch vụ đã thực hiện | POS |
| 15 | `crt.treatment_product_usage` | Transaction | 1 vật tư trong 1 lượt điều trị | POS |
| 16 | `crt.invoice` | Transaction | 1 hoá đơn | POS |
| 17 | `crt.invoice_line` | Transaction | 1 dòng hoá đơn | POS |
| 18 | `crt.payment` | Transaction | 1 lần chuyển tiền | POS + GW |
| 19 | `crt.payment_allocation` | Bridge | 1 cặp thanh toán × hoá đơn | POS |
| 20 | `crt.loyalty_transaction` | Transaction | 1 lần điểm biến động | OLTP (CDC) |
| 21 | `crt.membership_subscription` | Transaction | 1 kỳ thành viên | OLTP |
| 22 | `crt.feedback` | Transaction | 1 phiếu đánh giá | APP + MKT |
| 23 | `crt.ad_spend` | Transaction | 1 ngày × chiến dịch × nền tảng | ADS |
| 24 | `crt.campaign_send` | Transaction | 1 lần gửi tới 1 khách | MKT |
| 25 | `crt.service_view` | Transaction | 1 lần xem trang dịch vụ | APP + GA4 |
| 26 | `crt.invoice_line_promotion` | Trung gian | 1 cặp dòng hoá đơn × khuyến mãi | POS |
| 27 | `crt.v_customer_for_dim` | View | 1 khách | — |

**Tổng: 26 bảng + 1 view.**
