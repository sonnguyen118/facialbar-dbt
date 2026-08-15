# Thiết kế cơ sở dữ liệu ứng dụng — Facial Bar

Cơ sở dữ liệu vận hành của web-app: nơi khách đặt lịch, lễ tân xếp lịch, kỹ thuật viên ghi điều trị, thu ngân thu tiền. Đây là **nơi dữ liệu được sinh ra**.

PostgreSQL 14 hoặc mới hơn. Schema `app`. 40 bảng.

| | Cơ sở dữ liệu ứng dụng (tài liệu này) | Kho phân tích ([README](README.md)) |
|---|---|---|
| Việc phải làm nhanh | Ghi một lịch hẹn, thu một khoản tiền | Cộng doanh thu 5 năm theo chi nhánh |
| Đọc hay ghi nhiều hơn | Ghi nhiều, đọc từng dòng | Chỉ đọc, quét hàng triệu dòng |
| Dạng chuẩn | 3NF, không lặp dữ liệu | Phi chuẩn hoá có kiểm soát |
| Lịch sử | Chỉ giữ trạng thái hiện tại | Giữ toàn bộ lịch sử bằng SCD Type 2 |
| Một câu truy vấn | Dưới 10 ms | Dưới 5 giây |
| Ràng buộc | Chặt nhất có thể, chặn tại tầng dữ liệu | Chặn bằng 58 quy tắc chất lượng khi nạp |

Dữ liệu chảy từ đây sang kho qua Debezium đọc WAL: `app` → Kafka → hồ dữ liệu → `lnd` → `crt`. Xem [Flow.md](Flow.md).

---

## 1. Bốn nguyên tắc

| # | Nguyên tắc | Vì sao ở cơ sở dữ liệu ứng dụng nó quan trọng hơn |
|---|---|---|
| 1 | **Ràng buộc đặt ở tầng dữ liệu, không ở tầng ứng dụng** | Web, ứng dụng di động, máy POS và công cụ quản trị cùng ghi vào một cơ sở dữ liệu. Kiểm tra trong mã nguồn của web không ngăn được máy POS ghi sai |
| 2 | **Không xoá cứng dữ liệu giao dịch** | Hoá đơn và lịch hẹn đã phát sinh phải giữ lại để đối soát. Dùng `status` hoặc `voided_at`, không `DELETE` |
| 3 | **Mọi mốc thời gian là `TIMESTAMPTZ` lưu theo UTC** | Chuỗi có chi nhánh nhiều tỉnh; giờ địa phương đổi theo nơi hiển thị, không đổi theo nơi lưu |
| 4 | **Mỗi thay đổi nghiệp vụ ghi một dòng vào `outbox_event` trong cùng giao dịch** | Bảo đảm sự kiện gửi ra Kafka không mất khi ứng dụng chết giữa lúc ghi. Xem [mục 5.4](#54-gửi-sự-kiện-ra-kafka-không-mất-và-không-trùng) |

## 2. Chuẩn kỹ thuật

| Loại | Kiểu dùng | Không dùng |
|---|---|---|
| Khoá chính | `BIGINT GENERATED ALWAYS AS IDENTITY` | `SERIAL` — không chặn được `INSERT` chỉ định tay vào cột khoá |
| Khoá nghiệp vụ đối ngoại | `TEXT` kèm `UNIQUE`, ví dụ mã hoá đơn | Số tăng dần lộ ra ngoài — đoán được số lượng đơn |
| Tiền | `NUMERIC(18,2)` | `FLOAT`, `REAL` — cộng tiền bị sai số |
| Thời điểm | `TIMESTAMPTZ` | `TIMESTAMP` không múi giờ |
| Ngày nghiệp vụ | `DATE` | — |
| Trạng thái | `TEXT` kèm `CHECK (... IN (...))` | `ENUM` của PostgreSQL — thêm giá trị mới phải `ALTER TYPE`, khoá bảng |
| Email | `TEXT` kèm index `UNIQUE (lower(email))` | `UNIQUE (email)` — `A@x.com` và `a@x.com` thành hai tài khoản |
| Số điện thoại | `TEXT` chuẩn E.164, `UNIQUE` | Lưu nguyên dạng người dùng nhập |
| Mật khẩu | `TEXT` chứa chuỗi băm Argon2id | Bất cứ gì khác |

```sql
CREATE SCHEMA app;
CREATE EXTENSION IF NOT EXISTS btree_gist;   -- cần cho ràng buộc chống trùng lịch, mục 5.1
CREATE EXTENSION IF NOT EXISTS pgcrypto;     -- sinh mã đối ngoại
CREATE EXTENSION IF NOT EXISTS pg_trgm;      -- tìm khách theo tên gần đúng

SET TIME ZONE 'UTC';
```

---

## 3. DDL

### 3.1. Tài khoản và phân quyền

```sql
CREATE TABLE app.app_user (
    user_id        BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email          TEXT,
    phone          TEXT,                       -- E.164, ví dụ +84901234567
    password_hash  TEXT,                       -- NULL nếu chỉ đăng nhập bằng OTP
    full_name      TEXT        NOT NULL,
    user_type      TEXT        NOT NULL,
    status         TEXT        NOT NULL DEFAULT 'active',
    failed_login_count SMALLINT NOT NULL DEFAULT 0,
    locked_until   TIMESTAMPTZ,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT app_user_type_chk   CHECK (user_type IN ('customer','staff')),
    CONSTRAINT app_user_status_chk CHECK (status IN ('active','suspended','deleted')),
    -- Phải có ít nhất một cách liên lạc để đăng nhập và để gộp định danh ở kho
    CONSTRAINT app_user_contact_chk CHECK (email IS NOT NULL OR phone IS NOT NULL),
    CONSTRAINT app_user_phone_e164_chk CHECK (phone IS NULL OR phone ~ '^\+[1-9][0-9]{7,14}$')
);
CREATE UNIQUE INDEX app_user_email_uq ON app.app_user (lower(email)) WHERE email IS NOT NULL;
CREATE UNIQUE INDEX app_user_phone_uq ON app.app_user (phone)        WHERE phone IS NOT NULL;

CREATE TABLE app.role (
    role_id   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    role_code TEXT NOT NULL UNIQUE,            -- customer, receptionist, therapist, manager, admin
    role_name TEXT NOT NULL
);

CREATE TABLE app.permission (
    permission_id   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    permission_code TEXT NOT NULL UNIQUE,      -- booking.create, invoice.void, report.read
    description     TEXT NOT NULL
);

CREATE TABLE app.role_permission (
    role_id       BIGINT NOT NULL REFERENCES app.role(role_id)             ON DELETE CASCADE,
    permission_id BIGINT NOT NULL REFERENCES app.permission(permission_id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE app.user_role (
    user_id    BIGINT NOT NULL REFERENCES app.app_user(user_id) ON DELETE CASCADE,
    role_id    BIGINT NOT NULL REFERENCES app.role(role_id),
    -- Quyền có thể giới hạn theo chi nhánh: quản lý chi nhánh A không xem được số của B
    salon_id   BIGINT,          -- khoá ngoại thêm ở cuối script, xem mục 3.12
    granted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by BIGINT REFERENCES app.app_user(user_id),
    user_role_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
);
-- PostgreSQL không nhận biểu thức trong PRIMARY KEY, nên khoá duy nhất thật của
-- bảng này khai bằng hai index có điều kiện: một cho quyền gắn với chi nhánh,
-- một cho quyền toàn hệ thống. Gộp làm một index thường sẽ không chặn được việc
-- cấp trùng quyền toàn hệ thống, vì NULL không bằng NULL.
CREATE UNIQUE INDEX user_role_scoped_uq ON app.user_role (user_id, role_id, salon_id)
    WHERE salon_id IS NOT NULL;
CREATE UNIQUE INDEX user_role_global_uq ON app.user_role (user_id, role_id)
    WHERE salon_id IS NULL;

CREATE TABLE app.user_session (
    session_id     UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id        BIGINT      NOT NULL REFERENCES app.app_user(user_id) ON DELETE CASCADE,
    refresh_token_hash TEXT    NOT NULL,       -- chỉ lưu chuỗi băm, không lưu token
    device_label   TEXT,
    ip_address     INET,
    issued_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at     TIMESTAMPTZ NOT NULL,
    revoked_at     TIMESTAMPTZ,
    CONSTRAINT user_session_expiry_chk CHECK (expires_at > issued_at)
);
CREATE INDEX user_session_user_active_idx ON app.user_session (user_id)
    WHERE revoked_at IS NULL;

CREATE TABLE app.password_reset (
    reset_id    BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id     BIGINT      NOT NULL REFERENCES app.app_user(user_id) ON DELETE CASCADE,
    token_hash  TEXT        NOT NULL UNIQUE,
    expires_at  TIMESTAMPTZ NOT NULL,
    used_at     TIMESTAMPTZ
);
```

> **Tách `app_user` khỏi `customer` là có chủ đích.** Khách walk-in do lễ tân tạo hộ không có tài khoản đăng nhập; một khách có thể vừa là khách vừa là nhân viên. Gộp hai bảng buộc phải để `password_hash` và `email` cho phép NULL trên phần lớn số dòng, và không mô tả được khách không có tài khoản.

### 3.2. Chi nhánh, buồng, dịch vụ, sản phẩm

```sql
CREATE TABLE app.salon (
    salon_id      BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    salon_code    TEXT NOT NULL UNIQUE,
    salon_name    TEXT NOT NULL,
    address_line  TEXT NOT NULL,
    ward          TEXT,
    district      TEXT,
    city          TEXT NOT NULL,
    phone         TEXT,
    opened_on     DATE NOT NULL,
    closed_on     DATE,
    status        TEXT NOT NULL DEFAULT 'active',
    CONSTRAINT salon_status_chk CHECK (status IN ('active','temporarily_closed','closed')),
    CONSTRAINT salon_close_after_open_chk CHECK (closed_on IS NULL OR closed_on >= opened_on)
);

-- Giờ mở cửa theo thứ trong tuần. Đây chính là dữ liệu mà kho phân tích đang
-- thiếu để tính Tỷ lệ lấp buồng (README mục 9, "Năm dữ liệu chưa có nguồn").
CREATE TABLE app.salon_opening_hour (
    salon_id    BIGINT   NOT NULL REFERENCES app.salon(salon_id) ON DELETE CASCADE,
    day_of_week SMALLINT NOT NULL,             -- 1 = Thứ Hai ... 7 = Chủ Nhật
    open_time   TIME     NOT NULL,
    close_time  TIME     NOT NULL,
    PRIMARY KEY (salon_id, day_of_week),
    CONSTRAINT opening_dow_chk   CHECK (day_of_week BETWEEN 1 AND 7),
    CONSTRAINT opening_order_chk CHECK (close_time > open_time)
);

CREATE TABLE app.room (
    room_id   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    salon_id  BIGINT NOT NULL REFERENCES app.salon(salon_id),
    room_code TEXT   NOT NULL,
    room_type TEXT   NOT NULL DEFAULT 'facial',
    status    TEXT   NOT NULL DEFAULT 'active',
    CONSTRAINT room_status_chk CHECK (status IN ('active','maintenance','retired')),
    CONSTRAINT room_code_uq UNIQUE (salon_id, room_code)
);

CREATE TABLE app.service_category (
    category_id   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    category_code TEXT NOT NULL UNIQUE,
    category_name TEXT NOT NULL
);

CREATE TABLE app.service (
    service_id       BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    service_code     TEXT   NOT NULL UNIQUE,
    service_name     TEXT   NOT NULL,
    category_id      BIGINT NOT NULL REFERENCES app.service_category(category_id),
    duration_minutes SMALLINT NOT NULL,
    requires_room    BOOLEAN  NOT NULL DEFAULT true,
    status           TEXT     NOT NULL DEFAULT 'active',
    CONSTRAINT service_status_chk   CHECK (status IN ('active','paused','retired')),
    CONSTRAINT service_duration_chk CHECK (duration_minutes BETWEEN 5 AND 480)
);

-- Giá theo khoảng thời gian: đổi giá KHÔNG ghi đè dòng cũ, vì hoá đơn quá khứ
-- phải tra lại được giá tại thời điểm bán.
CREATE TABLE app.service_price (
    service_price_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    service_id       BIGINT        NOT NULL REFERENCES app.service(service_id),
    list_price        NUMERIC(18,2) NOT NULL,
    valid_from       DATE          NOT NULL,
    valid_to         DATE          NOT NULL DEFAULT DATE '9999-12-31',
    CONSTRAINT service_price_amount_chk CHECK (list_price >= 0),
    CONSTRAINT service_price_range_chk  CHECK (valid_to > valid_from),
    -- Một dịch vụ không được có hai giá cùng hiệu lực tại một ngày
    CONSTRAINT service_price_no_overlap EXCLUDE USING gist (
        service_id WITH =,
        daterange(valid_from, valid_to, '[)') WITH &&
    )
);

CREATE TABLE app.product (
    product_id    BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    product_code  TEXT   NOT NULL UNIQUE,
    product_name  TEXT   NOT NULL,
    unit          TEXT   NOT NULL,
    is_retail     BOOLEAN NOT NULL,             -- true: bán lẻ, false: chỉ dùng trong buồng
    list_price    NUMERIC(18,2),
    unit_cost     NUMERIC(18,2),
    status        TEXT   NOT NULL DEFAULT 'active',
    CONSTRAINT product_retail_price_chk
        CHECK (is_retail = false OR list_price IS NOT NULL),
    CONSTRAINT product_status_chk CHECK (status IN ('active','discontinued'))
);
```

### 3.3. Nhân viên, tay nghề, lịch làm việc

```sql
CREATE TABLE app.employee (
    employee_id   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id       BIGINT UNIQUE REFERENCES app.app_user(user_id),
    employee_code TEXT   NOT NULL UNIQUE,
    full_name     TEXT   NOT NULL,
    position      TEXT   NOT NULL,             -- therapist, receptionist, manager
    grade         TEXT,                        -- bậc tay nghề
    salon_id      BIGINT NOT NULL REFERENCES app.salon(salon_id),
    hired_on      DATE   NOT NULL,
    terminated_on DATE,
    CONSTRAINT employee_term_chk CHECK (terminated_on IS NULL OR terminated_on >= hired_on)
);

-- Kỹ thuật viên nào làm được dịch vụ nào. Chặn xếp lịch sai tay nghề.
CREATE TABLE app.employee_service (
    employee_id BIGINT NOT NULL REFERENCES app.employee(employee_id) ON DELETE CASCADE,
    service_id  BIGINT NOT NULL REFERENCES app.service(service_id),
    certified_on DATE  NOT NULL DEFAULT CURRENT_DATE,
    PRIMARY KEY (employee_id, service_id)
);

-- Phân ca. Đây là dữ liệu thứ hai mà kho phân tích đang thiếu: mẫu số của
-- Năng suất kỹ thuật viên và Tỷ lệ lấp buồng.
CREATE TABLE app.shift (
    shift_id    BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    employee_id BIGINT      NOT NULL REFERENCES app.employee(employee_id),
    salon_id    BIGINT      NOT NULL REFERENCES app.salon(salon_id),
    start_at    TIMESTAMPTZ NOT NULL,
    end_at      TIMESTAMPTZ NOT NULL,
    shift_type  TEXT        NOT NULL DEFAULT 'regular',
    CONSTRAINT shift_order_chk CHECK (end_at > start_at),
    CONSTRAINT shift_type_chk  CHECK (shift_type IN ('regular','overtime','leave')),
    -- Một người không thể có hai ca chồng nhau
    CONSTRAINT shift_no_overlap EXCLUDE USING gist (
        employee_id WITH =,
        tstzrange(start_at, end_at, '[)') WITH &&
    )
);
```

### 3.4. Khách hàng

```sql
CREATE TABLE app.customer (
    customer_id      BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id          BIGINT UNIQUE REFERENCES app.app_user(user_id),  -- NULL nếu khách walk-in
    customer_code    TEXT        NOT NULL UNIQUE DEFAULT encode(gen_random_bytes(8), 'hex'),
    full_name        TEXT        NOT NULL,
    phone            TEXT,
    email            TEXT,
    date_of_birth    DATE,
    gender           TEXT,
    first_salon_id   BIGINT      REFERENCES app.salon(salon_id),
    acquisition_channel TEXT,               -- fb_ads, google_ads, walk_in, referral
    registered_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    status           TEXT        NOT NULL DEFAULT 'active',
    CONSTRAINT customer_gender_chk CHECK (gender IS NULL OR gender IN ('F','M','OTHER')),
    CONSTRAINT customer_status_chk CHECK (status IN ('active','inactive','blacklisted')),
    CONSTRAINT customer_dob_chk    CHECK (date_of_birth IS NULL
                                          OR date_of_birth BETWEEN DATE '1900-01-01' AND CURRENT_DATE)
);
CREATE UNIQUE INDEX customer_phone_uq ON app.customer (phone) WHERE phone IS NOT NULL;
CREATE INDEX customer_name_trgm_idx ON app.customer USING gin (full_name gin_trgm_ops);

-- Tình trạng da và ảnh trước/sau KHÔNG nằm trong bảng này và không đẩy sang kho
-- phân tích (quyết định QĐ-06 ở bản trình phê duyệt). Chúng ở vùng lưu riêng,
-- phân quyền riêng, chỉ kỹ thuật viên phụ trách xem được.
CREATE TABLE app.customer_skin_profile (
    customer_id   BIGINT      PRIMARY KEY REFERENCES app.customer(customer_id) ON DELETE CASCADE,
    skin_type_code TEXT       NOT NULL,        -- 5 nhóm đã mã hoá, không mô tả tự do
    allergy_note  TEXT,
    updated_by    BIGINT      REFERENCES app.employee(employee_id),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 3.5. Đặt lịch và lịch hẹn

`booking` là **yêu cầu** của khách. `appointment` là **chỗ đã xếp** trong lịch: buồng nào, kỹ thuật viên nào, từ mấy giờ đến mấy giờ. Một `booking` nhiều dịch vụ sinh nhiều `appointment`. Tách hai bảng vì khách đổi lịch thì `appointment` đổi mà `booking` vẫn là một.

```sql
CREATE TABLE app.booking (
    booking_id     BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    booking_code   TEXT        NOT NULL UNIQUE DEFAULT encode(gen_random_bytes(6), 'hex'),
    customer_id    BIGINT      NOT NULL REFERENCES app.customer(customer_id),
    salon_id       BIGINT      NOT NULL REFERENCES app.salon(salon_id),
    channel        TEXT        NOT NULL,       -- app, web, hotline, walk_in
    status         TEXT        NOT NULL DEFAULT 'created',
    requested_at   TIMESTAMPTZ NOT NULL,       -- giờ khách muốn đến
    booked_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    confirmed_at   TIMESTAMPTZ,
    cancelled_at   TIMESTAMPTZ,
    cancel_reason  TEXT,
    promotion_id   BIGINT,      -- khoá ngoại thêm ở cuối script, xem mục 3.12
    campaign_code  TEXT,
    created_by     BIGINT      REFERENCES app.app_user(user_id),
    CONSTRAINT booking_channel_chk CHECK (channel IN ('app','web','hotline','walk_in')),
    CONSTRAINT booking_status_chk  CHECK (status IN
        ('created','confirmed','rescheduled','completed','cancelled','no_show')),
    -- Mốc thời gian phải nhất quán với trạng thái
    CONSTRAINT booking_confirm_chk CHECK ((status = 'created') = (confirmed_at IS NULL)
                                          OR status IN ('cancelled','no_show')),
    CONSTRAINT booking_cancel_chk  CHECK ((status = 'cancelled') = (cancelled_at IS NOT NULL))
);
CREATE INDEX booking_customer_idx ON app.booking (customer_id, booked_at DESC);
CREATE INDEX booking_salon_day_idx ON app.booking (salon_id, requested_at);

CREATE TABLE app.booking_item (
    booking_item_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    booking_id      BIGINT   NOT NULL REFERENCES app.booking(booking_id) ON DELETE CASCADE,
    service_id      BIGINT   NOT NULL REFERENCES app.service(service_id),
    quantity        SMALLINT NOT NULL DEFAULT 1,
    -- Giá chốt tại lúc đặt, không đọc lại service_price khi hiển thị lịch sử
    quoted_price    NUMERIC(18,2) NOT NULL,
    CONSTRAINT booking_item_qty_chk   CHECK (quantity > 0),
    CONSTRAINT booking_item_price_chk CHECK (quoted_price >= 0)
);
CREATE INDEX booking_item_booking_idx ON app.booking_item (booking_id);

CREATE TABLE app.appointment (
    appointment_id  BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    booking_id      BIGINT      NOT NULL REFERENCES app.booking(booking_id),
    booking_item_id BIGINT      NOT NULL REFERENCES app.booking_item(booking_item_id),
    salon_id        BIGINT      NOT NULL REFERENCES app.salon(salon_id),
    room_id         BIGINT      REFERENCES app.room(room_id),
    employee_id     BIGINT      REFERENCES app.employee(employee_id),
    start_at        TIMESTAMPTZ NOT NULL,
    end_at          TIMESTAMPTZ NOT NULL,
    status          TEXT        NOT NULL DEFAULT 'scheduled',
    checked_in_at   TIMESTAMPTZ,
    CONSTRAINT appointment_order_chk  CHECK (end_at > start_at),
    CONSTRAINT appointment_status_chk CHECK (status IN
        ('scheduled','checked_in','in_progress','done','cancelled','no_show'))
);
CREATE INDEX appointment_day_idx ON app.appointment (salon_id, start_at);

-- Vết chuyển trạng thái. Không có bảng này thì không tính được thời gian mỗi
-- chặng phễu, và không biết ai đã đổi lịch của khách.
CREATE TABLE app.appointment_event (
    appointment_event_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    appointment_id BIGINT      NOT NULL REFERENCES app.appointment(appointment_id) ON DELETE CASCADE,
    from_status    TEXT,
    to_status      TEXT        NOT NULL,
    occurred_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    actor_user_id  BIGINT      REFERENCES app.app_user(user_id),
    note           TEXT
);
CREATE INDEX appointment_event_appt_idx ON app.appointment_event (appointment_id, occurred_at);
```

### 3.6. Điều trị

```sql
CREATE TABLE app.treatment (
    treatment_id   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    appointment_id BIGINT      NOT NULL UNIQUE REFERENCES app.appointment(appointment_id),
    employee_id    BIGINT      NOT NULL REFERENCES app.employee(employee_id),
    service_id     BIGINT      NOT NULL REFERENCES app.service(service_id),
    room_id        BIGINT      REFERENCES app.room(room_id),
    started_at     TIMESTAMPTZ NOT NULL,
    finished_at    TIMESTAMPTZ,
    therapist_note TEXT,
    CONSTRAINT treatment_order_chk CHECK (finished_at IS NULL OR finished_at > started_at)
);

CREATE TABLE app.treatment_product_usage (
    treatment_id BIGINT        NOT NULL REFERENCES app.treatment(treatment_id) ON DELETE CASCADE,
    product_id   BIGINT        NOT NULL REFERENCES app.product(product_id),
    quantity     NUMERIC(12,3) NOT NULL,
    PRIMARY KEY (treatment_id, product_id),
    CONSTRAINT usage_qty_chk CHECK (quantity > 0)
);
```

### 3.7. Khuyến mãi

```sql
CREATE TABLE app.promotion (
    promotion_id   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    promotion_code TEXT NOT NULL UNIQUE,
    promotion_name TEXT NOT NULL,
    discount_type  TEXT NOT NULL,              -- percent, amount
    discount_value NUMERIC(18,2) NOT NULL,
    max_discount_amount NUMERIC(18,2),
    valid_from     DATE NOT NULL,
    valid_to       DATE NOT NULL,
    usage_limit    INTEGER,                    -- NULL: không giới hạn
    used_count     INTEGER NOT NULL DEFAULT 0,
    CONSTRAINT promotion_type_chk  CHECK (discount_type IN ('percent','amount')),
    CONSTRAINT promotion_value_chk CHECK (discount_value > 0
        AND (discount_type <> 'percent' OR discount_value <= 100)),
    CONSTRAINT promotion_range_chk CHECK (valid_to >= valid_from),
    CONSTRAINT promotion_usage_chk CHECK (usage_limit IS NULL OR used_count <= usage_limit)
);

CREATE TABLE app.promotion_service (
    promotion_id BIGINT NOT NULL REFERENCES app.promotion(promotion_id) ON DELETE CASCADE,
    service_id   BIGINT NOT NULL REFERENCES app.service(service_id),
    PRIMARY KEY (promotion_id, service_id)
);
```

### 3.8. Hoá đơn và thanh toán

```sql
CREATE TABLE app.invoice (
    invoice_id     BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    invoice_no     TEXT        NOT NULL UNIQUE,
    customer_id    BIGINT      NOT NULL REFERENCES app.customer(customer_id),
    salon_id       BIGINT      NOT NULL REFERENCES app.salon(salon_id),
    booking_id     BIGINT      REFERENCES app.booking(booking_id),
    service_at     TIMESTAMPTZ NOT NULL,       -- ngày thực hiện dịch vụ
    issued_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    status         TEXT        NOT NULL DEFAULT 'open',
    voided_at      TIMESTAMPTZ,
    void_reason    TEXT,
    cashier_id     BIGINT      REFERENCES app.employee(employee_id),
    CONSTRAINT invoice_status_chk CHECK (status IN ('open','paid','partially_paid','void')),
    CONSTRAINT invoice_void_chk   CHECK ((status = 'void') = (voided_at IS NOT NULL))
);
CREATE INDEX invoice_service_day_idx ON app.invoice (salon_id, service_at);
CREATE INDEX invoice_customer_idx    ON app.invoice (customer_id, service_at DESC);

CREATE TABLE app.invoice_line (
    invoice_line_id  BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    invoice_id       BIGINT   NOT NULL REFERENCES app.invoice(invoice_id) ON DELETE CASCADE,
    line_no          SMALLINT NOT NULL,
    item_type        TEXT     NOT NULL,        -- service, product
    service_id       BIGINT   REFERENCES app.service(service_id),
    product_id       BIGINT   REFERENCES app.product(product_id),
    treatment_id     BIGINT   REFERENCES app.treatment(treatment_id),
    employee_id      BIGINT   REFERENCES app.employee(employee_id),
    quantity         NUMERIC(12,3) NOT NULL DEFAULT 1,
    unit_price       NUMERIC(18,2) NOT NULL,
    -- Lưu cả ba loại giảm giá riêng, không gộp thành một cột: kho phân tích cần
    -- tách được giảm giá do khuyến mãi và giảm giá do hạng thẻ.
    promo_discount   NUMERIC(18,2) NOT NULL DEFAULT 0,
    member_discount  NUMERIC(18,2) NOT NULL DEFAULT 0,
    tax_amount       NUMERIC(18,2) NOT NULL DEFAULT 0,
    unit_cost        NUMERIC(18,2),
    CONSTRAINT invoice_line_no_uq UNIQUE (invoice_id, line_no),
    CONSTRAINT invoice_line_item_chk CHECK (
        (item_type = 'service' AND service_id IS NOT NULL AND product_id IS NULL) OR
        (item_type = 'product' AND product_id IS NOT NULL AND service_id IS NULL)),
    CONSTRAINT invoice_line_qty_chk      CHECK (quantity > 0),
    CONSTRAINT invoice_line_price_chk    CHECK (unit_price >= 0),
    CONSTRAINT invoice_line_discount_chk CHECK (promo_discount >= 0 AND member_discount >= 0
        AND promo_discount + member_discount <= quantity * unit_price)
);
CREATE INDEX invoice_line_invoice_idx ON app.invoice_line (invoice_id);

CREATE TABLE app.invoice_line_promotion (
    invoice_line_id BIGINT        NOT NULL REFERENCES app.invoice_line(invoice_line_id) ON DELETE CASCADE,
    promotion_id    BIGINT        NOT NULL REFERENCES app.promotion(promotion_id),
    discount_amount NUMERIC(18,2) NOT NULL,
    PRIMARY KEY (invoice_line_id, promotion_id),
    CONSTRAINT ilp_amount_chk CHECK (discount_amount >= 0)
);

CREATE TABLE app.payment (
    payment_id      BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    payment_no      TEXT        NOT NULL UNIQUE,
    customer_id     BIGINT      NOT NULL REFERENCES app.customer(customer_id),
    salon_id        BIGINT      NOT NULL REFERENCES app.salon(salon_id),
    method_code     TEXT        NOT NULL,      -- cash, card, qr, ewallet, voucher, point
    amount          NUMERIC(18,2) NOT NULL,
    status          TEXT        NOT NULL DEFAULT 'pending',
    gateway_code    TEXT,                      -- vnpay, momo, ... NULL nếu tiền mặt
    gateway_txn_id  TEXT,
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT payment_amount_chk CHECK (amount > 0),
    CONSTRAINT payment_status_chk CHECK (status IN ('pending','completed','failed','refunded')),
    CONSTRAINT payment_paid_chk   CHECK ((status = 'completed') = (paid_at IS NOT NULL)),
    -- Lũy đẳng cổng thanh toán: xem mục 5.2
    CONSTRAINT payment_gateway_txn_uq UNIQUE (gateway_code, gateway_txn_id)
);
CREATE INDEX payment_day_idx ON app.payment (salon_id, paid_at);

-- Một lần trả có thể chia cho nhiều hoá đơn, và một hoá đơn có thể trả nhiều lần.
CREATE TABLE app.payment_allocation (
    payment_id       BIGINT        NOT NULL REFERENCES app.payment(payment_id) ON DELETE CASCADE,
    invoice_id       BIGINT        NOT NULL REFERENCES app.invoice(invoice_id),
    allocated_amount NUMERIC(18,2) NOT NULL,
    PRIMARY KEY (payment_id, invoice_id),
    CONSTRAINT allocation_amount_chk CHECK (allocated_amount > 0)
);

CREATE TABLE app.refund (
    refund_id   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    payment_id  BIGINT        NOT NULL REFERENCES app.payment(payment_id),
    amount      NUMERIC(18,2) NOT NULL,
    reason      TEXT          NOT NULL,
    refunded_at TIMESTAMPTZ   NOT NULL DEFAULT now(),
    approved_by BIGINT        REFERENCES app.app_user(user_id),
    CONSTRAINT refund_amount_chk CHECK (amount > 0)
);
```

### 3.9. Thành viên và điểm thưởng

```sql
CREATE TABLE app.membership_plan (
    plan_code        TEXT PRIMARY KEY,          -- None, Silver, Gold, Platinum
    plan_name        TEXT NOT NULL,
    plan_rank        SMALLINT NOT NULL UNIQUE,
    min_spend_amount NUMERIC(18,2) NOT NULL,
    discount_pct     NUMERIC(9,4)  NOT NULL,
    point_multiplier NUMERIC(9,4)  NOT NULL DEFAULT 1,
    CONSTRAINT plan_discount_chk CHECK (discount_pct BETWEEN 0 AND 1)
);

CREATE TABLE app.membership_subscription (
    subscription_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id     BIGINT NOT NULL REFERENCES app.customer(customer_id),
    plan_code       TEXT   NOT NULL REFERENCES app.membership_plan(plan_code),
    valid_from      DATE   NOT NULL,
    valid_to        DATE   NOT NULL,
    CONSTRAINT subscription_range_chk CHECK (valid_to > valid_from),
    -- Một khách không thể ở hai hạng cùng lúc
    CONSTRAINT subscription_no_overlap EXCLUDE USING gist (
        customer_id WITH =,
        daterange(valid_from, valid_to, '[)') WITH &&
    )
);

CREATE TABLE app.loyalty_account (
    customer_id    BIGINT      PRIMARY KEY REFERENCES app.customer(customer_id),
    balance_points INTEGER     NOT NULL DEFAULT 0,
    updated_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT loyalty_balance_chk CHECK (balance_points >= 0)
);

CREATE TABLE app.loyalty_transaction (
    loyalty_txn_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id    BIGINT      NOT NULL REFERENCES app.loyalty_account(customer_id),
    txn_type       TEXT        NOT NULL,       -- earn, redeem, expire, adjust
    point_delta    INTEGER     NOT NULL,       -- dương khi tích, âm khi tiêu
    invoice_id     BIGINT      REFERENCES app.invoice(invoice_id),
    occurred_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    note           TEXT,
    CONSTRAINT loyalty_type_chk  CHECK (txn_type IN ('earn','redeem','expire','adjust')),
    CONSTRAINT loyalty_delta_chk CHECK (point_delta <> 0
        AND (txn_type <> 'earn'   OR point_delta > 0)
        AND (txn_type <> 'redeem' OR point_delta < 0))
);
CREATE INDEX loyalty_txn_customer_idx ON app.loyalty_transaction (customer_id, occurred_at);
```

> **Lưu cả số dư và biến động là có chủ đích, và không trái nguyên tắc của kho phân tích.** Ứng dụng cần đọc số dư trong 1 ms để hiển thị lúc thanh toán, không thể cộng lại toàn bộ lịch sử mỗi lần. Kho phân tích thì chỉ nạp `loyalty_transaction`; số dư ở kho được chốt lại ở `fact_customer_monthly_snapshot`. Ràng buộc giữ hai bên khớp nhau nằm ở [mục 5.3](#53-điểm-thưởng-không-âm-khi-hai-giao-dịch-chạy-cùng-lúc).

### 3.10. Đánh giá

```sql
CREATE TABLE app.feedback (
    feedback_id  BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    treatment_id BIGINT      NOT NULL UNIQUE REFERENCES app.treatment(treatment_id),
    customer_id  BIGINT      NOT NULL REFERENCES app.customer(customer_id),
    rating       SMALLINT    NOT NULL,         -- CSAT, thang 1..5
    nps_score    SMALLINT,                     -- NPS, thang 0..10, câu hỏi RIÊNG
    comment      TEXT,
    submitted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT feedback_rating_chk CHECK (rating BETWEEN 1 AND 5),
    CONSTRAINT feedback_nps_chk    CHECK (nps_score IS NULL OR nps_score BETWEEN 0 AND 10)
);
```

> `rating` thang 1–5 dùng tính CSAT. `nps_score` thang 0–10 là **câu hỏi khác**, không quy đổi được từ `rating`. Hai cột riêng để không ai tính NPS từ số sao.

### 3.11. Outbox và vết kiểm toán

```sql
-- Debezium đọc bảng này để phát sự kiện ra Kafka. Xem mục 5.4.
CREATE TABLE app.outbox_event (
    event_id       BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    aggregate_type TEXT        NOT NULL,       -- booking, invoice, payment, customer
    aggregate_id   TEXT        NOT NULL,
    event_type     TEXT        NOT NULL,       -- booking_created, payment_completed, ...
    payload        JSONB       NOT NULL,
    occurred_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    published_at   TIMESTAMPTZ
);
CREATE INDEX outbox_unpublished_idx ON app.outbox_event (event_id)
    WHERE published_at IS NULL;

CREATE TABLE app.audit_log (
    audit_id    BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    table_name  TEXT        NOT NULL,
    record_pk   TEXT        NOT NULL,
    action      TEXT        NOT NULL,          -- insert, update, delete
    changed_by  BIGINT      REFERENCES app.app_user(user_id),
    changed_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    old_value   JSONB,
    new_value   JSONB,
    CONSTRAINT audit_action_chk CHECK (action IN ('insert','update','delete'))
);
CREATE INDEX audit_log_record_idx ON app.audit_log (table_name, record_pk, changed_at DESC);
```

### 3.12. Khoá ngoại trỏ ngược và thứ tự chạy

Hai bảng tham chiếu tới bảng được tạo sau chúng, nên khoá ngoại tách ra chạy cuối cùng. Không tách thì script dừng ở `relation "app.salon" does not exist`.

```sql
ALTER TABLE app.user_role
    ADD CONSTRAINT user_role_salon_fk FOREIGN KEY (salon_id) REFERENCES app.salon(salon_id);
ALTER TABLE app.booking
    ADD CONSTRAINT booking_promotion_fk FOREIGN KEY (promotion_id) REFERENCES app.promotion(promotion_id);
```

Thứ tự chạy đầy đủ: extension → `salon` và danh mục (3.2) → tài khoản (3.1) → nhân sự (3.3) → khách hàng (3.4) → khuyến mãi (3.7) → đặt lịch (3.5) → điều trị (3.6) → hoá đơn (3.8) → thành viên (3.9) → đánh giá (3.10) → hệ thống (3.11) → khoá ngoại trỏ ngược (3.12) → hai ràng buộc chống trùng lịch ([mục 5.1](#51-hai-khách-đặt-cùng-một-buồng-cùng-một-giờ)).

---

## 4. Danh mục 40 bảng

| Nhóm | Bảng |
|---|---|
| Tài khoản và phân quyền (7) | `app_user`, `role`, `permission`, `role_permission`, `user_role`, `user_session`, `password_reset` |
| Chi nhánh và danh mục (7) | `salon`, `salon_opening_hour`, `room`, `service_category`, `service`, `service_price`, `product` |
| Nhân sự (3) | `employee`, `employee_service`, `shift` |
| Khách hàng (2) | `customer`, `customer_skin_profile` |
| Đặt lịch (4) | `booking`, `booking_item`, `appointment`, `appointment_event` |
| Điều trị (2) | `treatment`, `treatment_product_usage` |
| Khuyến mãi (2) | `promotion`, `promotion_service` |
| Hoá đơn và thanh toán (6) | `invoice`, `invoice_line`, `invoice_line_promotion`, `payment`, `payment_allocation`, `refund` |
| Thành viên và điểm (4) | `membership_plan`, `membership_subscription`, `loyalty_account`, `loyalty_transaction` |
| Đánh giá (1) | `feedback` |
| Hệ thống (2) | `outbox_event`, `audit_log` |

---

## 5. Bốn bài toán khó

### 5.1. Hai khách đặt cùng một buồng, cùng một giờ

Kiểm tra trong mã nguồn không giải quyết được: hai yêu cầu đến cùng lúc thì cả hai đều thấy buồng trống rồi cả hai đều ghi. PostgreSQL chặn được việc này ngay tại tầng dữ liệu bằng ràng buộc loại trừ trên khoảng thời gian:

```sql
ALTER TABLE app.appointment
    ADD CONSTRAINT appointment_no_overlap_room EXCLUDE USING gist (
        room_id WITH =,
        tstzrange(start_at, end_at, '[)') WITH &&
    ) WHERE (room_id IS NOT NULL AND status NOT IN ('cancelled','no_show'));

ALTER TABLE app.appointment
    ADD CONSTRAINT appointment_no_overlap_employee EXCLUDE USING gist (
        employee_id WITH =,
        tstzrange(start_at, end_at, '[)') WITH &&
    ) WHERE (employee_id IS NOT NULL AND status NOT IN ('cancelled','no_show'));
```

Ba chi tiết quyết định ràng buộc này chạy đúng:

| Chi tiết | Vì sao |
|---|---|
| `'[)'` — đóng đầu, mở cuối | Lịch 09:00–10:00 và 10:00–11:00 **không** chồng nhau. Dùng `'[]'` thì hai lịch liền kề bị coi là trùng và không xếp được lịch nào sát nhau |
| `WHERE status NOT IN ('cancelled','no_show')` | Lịch đã huỷ phải nhường chỗ cho lịch mới. Không có điều kiện này thì một lần huỷ khoá luôn khung giờ đó |
| `EXCLUDE` chứ không phải trigger | Ràng buộc do PostgreSQL thi hành nên đúng ở mọi mức cô lập giao dịch và với mọi ứng dụng ghi vào, kể cả máy POS và công cụ quản trị |

Ứng dụng chỉ cần bắt lỗi vi phạm ràng buộc và trả về "khung giờ vừa có người đặt":

```sql
-- Ứng dụng chạy INSERT bình thường, không cần SELECT kiểm tra trước
INSERT INTO app.appointment (booking_id, booking_item_id, salon_id, room_id, employee_id, start_at, end_at)
VALUES ($1, $2, $3, $4, $5, $6, $7);
-- Nếu trùng: SQLSTATE 23P01 exclusion_violation
```

### 5.2. Cổng thanh toán gọi lại nhiều lần cho một giao dịch

Cổng thanh toán không bảo đảm gọi callback đúng một lần. Gặp mạng chậm nó gọi lại, và nếu mỗi lần gọi tạo một dòng `payment` thì tiền thu được nhân lên theo số lần gọi.

Ràng buộc `UNIQUE (gateway_code, gateway_txn_id)` khiến lần gọi thứ hai không tạo được dòng mới. Ứng dụng dùng `ON CONFLICT` để lần gọi lại trở thành vô hại:

```sql
INSERT INTO app.payment
    (payment_no, customer_id, salon_id, method_code, amount, status,
     gateway_code, gateway_txn_id, paid_at)
VALUES ($1, $2, $3, $4, $5, 'completed', $6, $7, $8)
ON CONFLICT (gateway_code, gateway_txn_id) DO NOTHING
RETURNING payment_id;
```

Không có dòng trả về nghĩa là giao dịch đã ghi trước đó — trả về thành công cho cổng thanh toán, không ghi thêm gì. Tiền mặt không có `gateway_txn_id` nên `UNIQUE` không áp vào, đúng như mong muốn: `NULL` không xung đột với `NULL` trong PostgreSQL.

### 5.3. Điểm thưởng không âm khi hai giao dịch chạy cùng lúc

Khách có 100 điểm, đồng thời tiêu 80 điểm ở hai thiết bị. Nếu mỗi giao dịch đọc số dư rồi trừ thì cả hai đều thấy 100 và cả hai đều thành công, số dư còn âm 60.

`CHECK (balance_points >= 0)` chặn được kết quả âm, nhưng phải khoá dòng để hai giao dịch không đọc cùng một số dư. Toàn bộ nằm trong một giao dịch:

```sql
BEGIN;
-- Khoá dòng số dư. Giao dịch thứ hai chờ tại đây cho tới khi giao dịch đầu xong.
SELECT balance_points FROM app.loyalty_account
 WHERE customer_id = $1 FOR UPDATE;

INSERT INTO app.loyalty_transaction (customer_id, txn_type, point_delta, invoice_id)
VALUES ($1, 'redeem', -$2, $3);

-- Nếu không đủ điểm, CHECK vi phạm và cả giao dịch bị hoàn lại
UPDATE app.loyalty_account
   SET balance_points = balance_points - $2, updated_at = now()
 WHERE customer_id = $1;

INSERT INTO app.outbox_event (aggregate_type, aggregate_id, event_type, payload)
VALUES ('customer', $1::text, 'points_redeemed',
        jsonb_build_object('customer_id', $1, 'points', $2, 'invoice_id', $3));
COMMIT;
```

Số dư và tổng biến động phải luôn bằng nhau. Kiểm tra này chạy mỗi đêm, và là quy tắc đối soát giữa cơ sở dữ liệu ứng dụng và kho phân tích:

```sql
SELECT a.customer_id, a.balance_points, COALESCE(SUM(t.point_delta), 0) AS txn_sum
FROM   app.loyalty_account a
LEFT JOIN app.loyalty_transaction t ON t.customer_id = a.customer_id
GROUP  BY a.customer_id, a.balance_points
HAVING a.balance_points <> COALESCE(SUM(t.point_delta), 0);
```

### 5.4. Gửi sự kiện ra Kafka không mất và không trùng

Ghi cơ sở dữ liệu rồi gọi Kafka là hai việc riêng. Ứng dụng chết giữa hai việc đó thì lịch hẹn đã lưu mà kho phân tích không bao giờ biết.

Cách giải là ghi sự kiện vào `outbox_event` **trong cùng giao dịch** với thay đổi nghiệp vụ. Một giao dịch thành công thì cả thay đổi và sự kiện cùng có; thất bại thì cả hai cùng không. Debezium đọc WAL của `outbox_event` và phát ra Kafka.

```sql
BEGIN;
INSERT INTO app.booking (customer_id, salon_id, channel, requested_at, status)
VALUES ($1, $2, $3, $4, 'created')
RETURNING booking_id INTO v_booking_id;

INSERT INTO app.outbox_event (aggregate_type, aggregate_id, event_type, payload)
VALUES ('booking', v_booking_id::text, 'booking_created',
        jsonb_build_object('booking_id', v_booking_id, 'customer_id', $1,
                           'salon_id', $2, 'channel', $3, 'requested_at', $4));
COMMIT;
```

Không cần cập nhật `published_at` bằng ứng dụng — Debezium đọc log nên biết đã phát tới đâu. Cột đó chỉ để dọn dữ liệu cũ:

```sql
DELETE FROM app.outbox_event WHERE occurred_at < now() - INTERVAL '7 days';
```

---

## 6. Index theo mẫu truy vấn

Cơ sở dữ liệu ứng dụng đánh index theo **câu truy vấn thật của màn hình**, không đánh cho mọi khoá ngoại.

| Màn hình | Câu truy vấn | Index phục vụ |
|---|---|---|
| Lịch trong ngày của một chi nhánh | `WHERE salon_id = ? AND start_at BETWEEN ? AND ?` | `appointment_day_idx (salon_id, start_at)` |
| Lịch sử của một khách | `WHERE customer_id = ? ORDER BY booked_at DESC` | `booking_customer_idx (customer_id, booked_at DESC)` |
| Đăng nhập | `WHERE lower(email) = ?` | `app_user_email_uq (lower(email))` |
| Tìm khách theo tên | `WHERE full_name ILIKE '%?%'` | `customer_name_trgm_idx` dùng gin trigram |
| Phiên đang hiệu lực | `WHERE user_id = ? AND revoked_at IS NULL` | `user_session_user_active_idx` có điều kiện |
| Sự kiện chưa phát | `WHERE published_at IS NULL` | `outbox_unpublished_idx` có điều kiện |
| Đối soát doanh thu ngày | `WHERE salon_id = ? AND service_at::date = ?` | `invoice_service_day_idx (salon_id, service_at)` |

Ba index có điều kiện `WHERE` chỉ chứa số dòng cần dùng: `user_session` chỉ giữ phiên còn hiệu lực, `outbox_event` chỉ giữ sự kiện chưa phát. Index đầy đủ trên hai bảng này sẽ lớn gấp nhiều lần mà không dùng tới.

---

## 7. Đưa dữ liệu sang kho phân tích

| Bảng ứng dụng | Cơ chế | Bảng `crt` tương ứng |
|---|---|---|
| `customer`, `booking`, `booking_item`, `appointment`, `invoice`, `invoice_line`, `payment` | Debezium đọc WAL, phát ra Kafka | `crt.customer`, `crt.booking`, `crt.booking_item`, `crt.appointment`, `crt.invoice`, `crt.invoice_line`, `crt.payment` |
| `outbox_event` | Debezium đọc WAL — nguồn của các sự kiện nghiệp vụ | Không nạp vào `crt`; dùng để đo độ trễ và cho bảng thời gian thực |
| `salon`, `room`, `service`, `product`, `promotion`, `employee` | Trích theo lô mỗi đêm, dữ liệu danh mục đổi chậm | `crt.salon`, `crt.room`, `crt.service`, `crt.product`, `crt.promotion`, `crt.employee` |
| `shift`, `salon_opening_hour` | Trích theo lô mỗi đêm | `lnd.hr_shift` — **mở được hai chỉ tiêu đang bị chặn**: Năng suất kỹ thuật viên và Tỷ lệ lấp buồng |
| `customer_skin_profile` | **Không đưa sang kho** | Không có — quyết định QĐ-06 |
| `user_session`, `password_reset`, `audit_log` | **Không đưa sang kho** | Không có — dữ liệu vận hành hệ thống, không phục vụ phân tích |

Cấu hình Debezium cho PostgreSQL:

```json
{
  "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
  "plugin.name": "pgoutput",
  "slot.name": "facialbar_cdc",
  "publication.autocreate.mode": "filtered",
  "table.include.list": "app.customer,app.booking,app.booking_item,app.appointment,app.invoice,app.invoice_line,app.payment,app.payment_allocation,app.loyalty_transaction,app.feedback,app.outbox_event",
  "column.exclude.list": "app.customer.date_of_birth",
  "tombstones.on.delete": "false",
  "snapshot.mode": "initial"
}
```

Hai điều cần theo dõi khi bật CDC trên PostgreSQL:

| Việc | Vì sao |
|---|---|
| Giám sát độ trễ của replication slot | Slot không được đọc thì WAL không được dọn và đĩa đầy dần. Đây là cách phổ biến nhất làm sập một cơ sở dữ liệu có CDC |
| `REPLICA IDENTITY FULL` cho bảng cần biết giá trị cũ | Mặc định `UPDATE` chỉ ghi khoá chính và cột đã đổi. Muốn kho biết giá trị trước khi đổi thì phải bật, và WAL sẽ lớn hơn |

```sql
ALTER TABLE app.booking     REPLICA IDENTITY FULL;
ALTER TABLE app.appointment REPLICA IDENTITY FULL;
```

---

## 8. Những gì chưa chốt

| Hạng mục | Cần ai quyết |
|---|---|
| Giá vốn dịch vụ — `invoice_line.unit_cost` hiện chỉ có cho sản phẩm | Kế toán. Đang chặn Lợi nhuận gộp và Giá trị vòng đời khách hàng |
| Chính sách hết hạn điểm thưởng — hiện có `txn_type = 'expire'` nhưng chưa có quy tắc sinh ra nó | Chăm sóc khách hàng |
| Có cho phép đặt lịch không chọn kỹ thuật viên rồi hệ thống tự xếp không | Vận hành. Ảnh hưởng tới việc `appointment.employee_id` có được để NULL hay không |
| Thời gian giữ `audit_log` và `user_session` | Bảo mật và Pháp chế |

---

**Tài liệu liên quan:** [README.md](README.md) tổng thiết kế · [Flow.md](Flow.md) đường đi dữ liệu · [Flow-DA.md](Flow-DA.md) luồng thiết kế phân tích · [Ban-Thiet-Ke-CSDL.md](Ban-Thiet-Ke-CSDL.md) bản trình phê duyệt
