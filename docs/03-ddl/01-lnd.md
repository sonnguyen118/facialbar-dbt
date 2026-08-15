# DDL — Schema `lnd` (Landing)

Vùng đệm tiếp nhận dữ liệu từ `cleansed` trên S3 vào SQL Server.

| Đặc tính | Quy tắc | Lý do |
|---|---|---|
| Cấu trúc lưu | **Heap** — không index, không PK | Chỉ ghi một lần rồi đọc một lần; index chỉ làm chậm nạp |
| Chế độ nạp | **Ghi đè** toàn bộ theo mỗi lần chạy | Lịch sử đã nằm ở S3, giữ lại đây là trùng lặp |
| Kiểu dữ liệu | `NVARCHAR` cho **mọi** cột nghiệp vụ | Bảng này **không bao giờ được không đạt lúc nạp**. Sai kiểu để tầng `crt` bắt với thông báo rõ ràng |
| Ràng buộc | Không có `NOT NULL`, `CHECK`, `FK` | Cùng lý do trên |
| Recovery | Bảng có thể `TRUNCATE` bất kỳ lúc nào | Dựng lại được từ `cleansed` |

> Nguyên tắc cốt lõi của tầng này: **tách lỗi vận chuyển khỏi lỗi dữ liệu.** Nạp vào `lnd` thất bại nghĩa là lỗi hạ tầng. Nạp vào `crt` thất bại nghĩa là lỗi dữ liệu. Trộn hai loại lỗi này làm việc chẩn đoán sự cố lúc 3 giờ sáng trở nên bất khả thi.

---

## Cột kỹ thuật bắt buộc

Mọi bảng `lnd` có đúng 5 cột kỹ thuật này, đặt ở cuối bảng:

```sql
    _src_file    VARCHAR(1000)    NULL,   -- đường dẫn S3 của file nguồn
    _src_line_no BIGINT           NULL,   -- số dòng trong file, để truy vết dòng lỗi
    _run_id      UNIQUEIDENTIFIER NOT NULL,
    _loaded_at   DATETIME2(3)     NOT NULL,
    _lsn         BIGINT           NULL    -- chỉ với nguồn CDC, dùng khử trùng lặp
```

`_src_file` + `_src_line_no` là cặp cho phép truy một dòng sai ngược về **đúng file và đúng dòng** trong file gốc trên S3. Thiếu cặp này thì khi nguồn gửi 200.000 dòng và 3 dòng sai, không có cách nào chỉ ra dòng nào.

---

## Bảng mẫu đầy đủ

```sql
CREATE TABLE lnd.pos_invoice_line (
    -- Cột nghiệp vụ: giữ nguyên tên nguồn, kiểu NVARCHAR
    id                NVARCHAR(50),
    invoice_id        NVARCHAR(50),
    seq               NVARCHAR(20),
    item_type         NVARCHAR(20),
    item_id           NVARCHAR(50),
    treatment_ref     NVARCHAR(50),
    performed_by      NVARCHAR(50),
    qty               NVARCHAR(50),
    unit_price        NVARCHAR(50),
    promo_discount    NVARCHAR(50),
    member_discount   NVARCHAR(50),
    vat_amount        NVARCHAR(50),
    -- Cột kỹ thuật
    _src_file         VARCHAR(1000)    NULL,
    _src_line_no      BIGINT           NULL,
    _run_id           UNIQUEIDENTIFIER NOT NULL,
    _loaded_at        DATETIME2(3)     NOT NULL,
    _lsn              BIGINT           NULL
);
```

Không có `CREATE INDEX`, không có `CONSTRAINT` — đúng thiết kế.

---

## Danh mục bảng `lnd`

Tên bảng theo quy ước `lnd.<nguồn>_<entity>`. Cột nghiệp vụ **giữ nguyên tên của hệ thống nguồn**, không đổi sang tên chuẩn — việc đổi tên diễn ra ở bước nạp `lnd → crt`, và giữ tên gốc ở đây giúp đối chiếu với nguồn nhanh hơn.

| # | Bảng | Nguồn | Cột nghiệp vụ (tên gốc từ nguồn) | Có `_lsn` |
|---|---|---|---|---|
| 1 | `lnd.pos_store` | POS | `store_id, code, name, city, district, address, bed_count, manager_id, open_date, close_date` | |
| 2 | `lnd.pos_room` | POS | `room_id, store_id, name, type, active` | |
| 3 | `lnd.pos_service` | POS | `service_id, code, name, category_path, duration, price, signature_flag, active` | |
| 4 | `lnd.pos_product` | POS | `product_id, code, name, category, brand, unit, price, cost, type, active` | |
| 5 | `lnd.pos_promotion` | POS | `promo_id, code, name, kind, value, date_from, date_to` | |
| 6 | `lnd.pos_promotion_service` | POS | `promo_id, service_id` | |
| 7 | `lnd.pos_customer` | POS | `customer_id, mobile, name, first_visit_date` | |
| 8 | `lnd.pos_appointment` | POS | `id, booking_ref, customer_id, store_id, employee_id, room_id, status, scheduled_at, checkin_time, duration` | |
| 9 | `lnd.pos_treatment` | POS | `id, appointment_ref, customer_id, store_id, employee_id, room_id, service_id, promo_id, start_time, end_time` | |
| 10 | `lnd.pos_treatment_product` | POS | `id, treatment_id, product_id, qty, unit_cost` | |
| 11 | `lnd.pos_invoice` | POS | `id, number, customer_id, store_id, status, service_date, created_at, campaign_ref` | |
| 12 | **`lnd.pos_invoice_line`** | POS | xem bảng mẫu ở trên | |
| 13 | `lnd.pos_payment` | POS | `id, invoice_ref, method, status, amount, refund_amount, paid_time, gateway_ref, error_code` | |
| 14 | `lnd.pos_payment_allocation` | POS | `id, payment_id, invoice_id, amount` | |
| 15 | `lnd.cdc_customer` | OLTP | `id, phone, email, full_name, dob, gender, created_at, status, _op` | ✓ |
| 16 | `lnd.cdc_booking` | OLTP | `id, customer_id, store_id, source, status, created_at, slot_time, cancel_note, promo_id, campaign_ref, _op` | ✓ |
| 17 | `lnd.cdc_booking_line` | OLTP | `id, booking_id, service_id, qty, price, discount, _op` | ✓ |
| 18 | `lnd.cdc_point_ledger` | OLTP | `id, customer_id, type, points, ref_payment_id, created_at, _op` | ✓ |
| 19 | `lnd.cdc_membership` | OLTP | `id, customer_id, tier, valid_from, valid_to, amount, purchased_at, _op` | ✓ |
| 20 | `lnd.app_service_viewed` | APP | `event_id, session_id, customer_id, service_id, duration_sec, occurred_at, received_at` | |
| 21 | `lnd.app_feedback` | APP | `feedback_id, customer_id, treatment_ref, rating, comment, occurred_at, received_at` | |
| 22 | `lnd.gw_transaction` | GW | `txn_id, pos_ref, amount, fee, status, txn_time` | |
| 23 | `lnd.ads_insights` | ADS | `date_start, campaign_id, campaign_name, platform, spend, impressions, clicks, leads, currency` | |
| 24 | `lnd.ga4_session` | GA4 | `ga_session_id, user_pseudo_id, event_date, source, medium, campaign` | |
| 25 | `lnd.mkt_send_log` | MKT | `id, campaign_id, recipient_id, channel, sent_at, opened_at, clicked_at` | |
| 26 | `lnd.mkt_survey` | MKT | `id, customer_id, nps_answer, surveyed_at` | |
| 27 | `lnd.hr_employee` | HR | `emp_id, code, full_name, position, grade, store_id, hire_date, terminate_date` | |
| 28 | `lnd.hr_shift` | HR | `id, employee_id, store_id, work_date, shift_start, shift_end` | |
| 29 | `lnd.pos_revenue_control` | POS | `store_id, service_date, invoice_cnt, gross_amount, discount_amount, net_amount, currency` | Bảng số liệu đối chiếu do POS tự tính |

**Tổng: 29 bảng.**

### `lnd.pos_revenue_control` — bảng đối chiếu, không sinh tự động

Bảng này không nằm trong nhóm sinh tự động từ schema Iceberg vì nó không phải bản sao một bảng nguồn, mà là **số tổng do chính POS tính ra**. Đây là đầu vào duy nhất của `DQ-RECON-001` — tiêu chí nghiệm thu số 1. Không có nó thì việc đối soát chỉ là so `crt` với `lnd`, tức so dữ liệu với chính nó.

```sql
CREATE TABLE lnd.pos_revenue_control (
    -- Cột nghiệp vụ: giữ nguyên tên và kiểu chuỗi như mọi bảng `lnd` khác
    store_id          NVARCHAR(50),
    service_date      NVARCHAR(30),
    invoice_cnt       NVARCHAR(50),
    gross_amount      NVARCHAR(50),
    discount_amount   NVARCHAR(50),
    net_amount        NVARCHAR(50),
    currency          NVARCHAR(10),
    -- Cột kỹ thuật
    _src_file         VARCHAR(1000)    NULL,
    _src_line_no      BIGINT           NULL,
    _run_id           UNIQUEIDENTIFIER NOT NULL,
    _loaded_at        DATETIME2(3)     NOT NULL,
    _lsn              BIGINT           NULL
);
```

> **Trạng thái: chờ nhà cung cấp POS.** Cần yêu cầu POS xuất tệp tổng doanh thu theo ngày × chi nhánh, cùng cách tính doanh thu thuần mà họ dùng. Nếu POS chỉ xuất được doanh thu gộp thì phải thống nhất lại công thức trước khi đối soát, vì so doanh thu gộp của POS với doanh thu thuần của kho sẽ luôn lệch. Đây là điều kiện tiên quyết đã nêu ở [bản trình phê duyệt mục 11](../../Ban-Thiet-Ke-CSDL.md#6-rủi-ro-và-điều-kiện-tiên-quyết).

> `lnd.hr_shift` là nguồn của `available_minutes` — mẫu số của Therapist Năng suất và Bed Occupancy. Đây là dữ liệu **chưa có** ở thời điểm viết tài liệu; xem [phụ thuộc bên ngoài](../02-mapping/source-to-target.md#phụ-thuộc-bên-ngoài-chưa-có).

---

## Script sinh DDL

28 bảng còn lại có cấu trúc đồng dạng nên viết tay từng bảng là việc lặp không cần thiết và dễ sai sót. DDL được **sinh tự động từ schema Iceberg** của tầng `cleansed`:

```python
# Chạy trong task Airflow: dag_load_dwh → task generate_lnd_ddl
TECH_COLS = """    _src_file    VARCHAR(1000)    NULL,
    _src_line_no BIGINT           NULL,
    _run_id      UNIQUEIDENTIFIER NOT NULL,
    _loaded_at   DATETIME2(3)     NOT NULL,
    _lsn         BIGINT           NULL"""

def gen_lnd_ddl(table_name: str, columns: list[str]) -> str:
    """Sinh CREATE TABLE cho một bảng landing. Mọi cột nghiệp vụ đều NVARCHAR."""
    body = ",\n".join(f"    {c:<24} NVARCHAR(4000)" for c in columns)
    return (
        f"IF OBJECT_ID('lnd.{table_name}') IS NOT NULL DROP TABLE lnd.{table_name};\n"
        f"CREATE TABLE lnd.{table_name} (\n{body},\n{TECH_COLS}\n);"
    )
```

Độ rộng `NVARCHAR(4000)` là cố ý: bảng này không được không đạt vì dữ liệu dài quá dự kiến. Chi phí dung lượng không đáng kể vì bảng bị ghi đè mỗi lần chạy.

## Nạp và dọn

```sql
-- Trước mỗi lần nạp: xoá sạch, không giữ lịch sử
TRUNCATE TABLE lnd.pos_invoice_line;

-- Sau khi nạp xong sang crt: có thể truncate ngay để giải phóng dung lượng.
-- Thực tế nên giữ đến lần chạy kế tiếp để phục vụ chẩn đoán sự cố.
```

Không dùng `DELETE` — `TRUNCATE` nhanh hơn nhiều và không làm tăng transaction log.
