# SOURCE-TO-TARGET MAPPING

Bảng tra cứu ở mức cột: mỗi cột đích lấy từ nguồn nào, biến đổi ra sao, xử lý thế nào khi thiếu.

Đây là tài liệu người viết ETL tra liên tục. Mọi thay đổi ở đây phải đồng bộ với [DDL](../03-ddl/) và [DQ rules](../05-quality/dq-rules.md).

---

## Quy ước đọc bảng

| Cột | Ý nghĩa |
|---|---|
| **Nguồn** | Hệ thống · bảng/topic · cột |
| **Biến đổi** | Phép biến đổi áp dụng. `1:1` = copy nguyên |
| **Khi NULL/thiếu** | Giá trị thay thế. `-1` = Unknown member, `FAIL` = chặn dòng vào vùng cách ly |
| **DQ** | Mã quy tắc kiểm tra áp lên cột này |

**Ký hiệu hệ thống nguồn:**

| Mã | Hệ thống | Cơ chế thu nạp |
|---|---|---|
| `POS` | POS tại cửa hàng | Batch/CDC → `raw/pos/` |
| `OLTP` | DB ứng dụng đặt lịch | CDC (Debezium) → `raw/cdc/` |
| `APP` | Sự kiện từ app/web | Streaming (Kafka) → `raw/kafka/` |
| `GW` | Cổng thanh toán | API batch → `raw/gateway/` |
| `ADS` | Facebook / Google Ads | API batch → `raw/ads/` |
| `GA4` | Google Analytics 4 | Export batch → `raw/ga4/` |
| `MKT` | Marketing platform (email/SMS/Zalo) | API batch → `raw/mkt/` |
| `HR` | Bảng nhân sự | Batch file → `raw/hr/` |
| `DWH` | Tính toán trong kho, không có nguồn ngoài | — |

---

## 1. `crt.customer` ← nhiều nguồn

độ hạt: 1 dòng = 1 khách hàng đã gộp định danh.

| Cột đích | Nguồn | Biến đổi | Khi NULL/thiếu | DQ |
|---|---|---|---|---|
| `customer_id` | `DWH` | Sinh từ `crt.customer_identity_map.customer_id` sau khi gộp định danh | `FAIL` | DQ-CUS-001 |
| `phone` | `OLTP.customer.phone`, dự phòng `POS.customer.mobile` | Chuẩn E.164: bỏ ký tự không phải số, `0xxxxxxxxx` → `+84xxxxxxxxx` | `NULL` (cho phép) | DQ-CUS-002 |
| `email` | `OLTP.customer.email` | `LOWER(TRIM())` | `NULL` | DQ-CUS-003 |
| `full_name` | `OLTP.customer.full_name`, dự phòng `POS.customer.name` | `TRIM`, gộp khoảng trắng liên tiếp thành một | `N'(Chưa có tên)'` | — |
| `date_of_birth` | `OLTP.customer.dob` | Parse `dd/MM/yyyy` → `DATE` | `NULL` | DQ-CUS-004 |
| `gender` | `OLTP.customer.gender` | `Nữ/F/female` → `F`; `Nam/M/male` → `M`; khác → `OTHER` | `'UNKNOWN'` | — |
| `registration_date` | `MIN` của `OLTP.customer.created_at` và `POS.customer.first_visit_date` | `CAST(... AS DATE)`, lấy giá trị nhỏ hơn | Ngày đầu tiên có giao dịch | DQ-CUS-005 |
| `acquisition_channel` | `APP.customer_registered.channel`, dự phòng `POS` | Ánh xạ danh mục, xem [bảng ánh xạ kênh](#ánh-xạ-danh-mục) | `'UNKNOWN'` | — |
| `first_salon_id` | `POS` — salon của hoá đơn đầu tiên | `FIRST_VALUE` theo `service_at` | `-1` | — |
| `status` | `OLTP.customer.status` | `active/inactive/blacklisted`, giá trị khác → `inactive` | `'active'` | — |
| `_is_deleted` | CDC `op = 'd'` | `1` nếu CDC báo DELETE | `0` | — |

> Khách walk-in tại cửa hàng chỉ có `phone` và `full_name` từ POS, không có bản ghi OLTP. Vẫn tạo dòng `crt.customer` để hoá đơn không bị gán `-1`.

## 2. `crt.customer_identity_map` ← nhiều nguồn

độ hạt: 1 dòng = 1 danh tính ở 1 hệ thống nguồn.

| Cột đích | Nguồn | Biến đổi | Khi NULL/thiếu | DQ |
|---|---|---|---|---|
| `source_system` | Cố định theo luồng nạp | `'app'` / `'pos'` / `'ga4'` / `'crm'` | `FAIL` | — |
| `source_id` | ID gốc của hệ thống đó | `1:1` | `FAIL` | — |
| `match_key` | `phone` chuẩn E.164, dự phòng `email` lowercase | Chuẩn hoá trước khi so | `NULL` → không gộp được | — |
| `customer_id` | `DWH` | Gán theo thứ tự ưu tiên: `phone` → `email` → `(tên + ngày sinh + salon)` → thủ công | Sinh ID mới | DQ-CUS-006 |
| `match_method` | `DWH` | `exact_phone` / `exact_email` / `fuzzy_name_dob` / `manual` | `FAIL` | — |
| `match_confidence` | `DWH` | `1.00` cho exact; `0.60–0.85` cho fuzzy | `FAIL` | DQ-CUS-007 |

> `match_confidence < 0.80` **không tự động gộp** — đưa vào danh sách chờ người rà. Gộp sai hai khách thành một rất khó phát hiện và khó tách lại.

## 3. `crt.salon` ← `POS` + `HR`

| Cột đích | Nguồn | Biến đổi | Khi NULL/thiếu |
|---|---|---|---|
| `salon_id` | `POS.store.store_id` | `1:1` | `FAIL` |
| `salon_code` | `POS.store.code` | `UPPER(TRIM())` | `FAIL` |
| `salon_name` | `POS.store.name` | `TRIM` | `FAIL` |
| `city`, `district`, `address` | `POS.store.*` | `TRIM`, chuẩn theo danh mục địa giới | `N'(Chưa xác định)'` |
| `region` | `DWH` | Suy từ `city` theo [bảng ánh xạ vùng](#ánh-xạ-danh-mục) | `'UNKNOWN'` |
| `capacity_beds` | `POS.store.bed_count` | `CAST` sang `TINYINT` | `FAIL` (bắt buộc để tính lấp buồng) |
| `open_date`, `close_date` | `POS.store.*` | Parse `DATE` | `close_date` `NULL` = đang hoạt động |
| `is_active` | `DWH` | `1` nếu `close_date IS NULL` hoặc `> hôm nay` | — |

## 4. `crt.employee` ← `HR` + `POS`

| Cột đích | Nguồn | Biến đổi | Khi NULL/thiếu |
|---|---|---|---|
| `employee_id` | `HR.employee.emp_id` | `1:1` | `FAIL` |
| `employee_code` | `HR.employee.code` | `UPPER(TRIM())` | `FAIL` |
| `employee_name` | `HR.employee.full_name` | `TRIM` | `FAIL` |
| `role_name` | `HR.employee.position` | Ánh xạ về `therapist` / `receptionist` / `manager` / `other` | `'other'` |
| `skill_level` | `HR.employee.grade` | `Junior` / `Senior` / `Expert` | `'Junior'` |
| `salon_id` | `HR.employee.store_id` | `1:1` | `-1` |
| `hire_date`, `terminate_date` | `HR.employee.*` | Parse `DATE` | `terminate_date` `NULL` = đang làm |
| `tenure_band` | `DWH` | Từ `hire_date` đến ngày chạy: `<6m` / `6-12m` / `1-3y` / `3y+` | — |

> KTV nghỉ việc **không xoá** khỏi `crt` — điều trị trong quá khứ vẫn phải quy được về người thực hiện.

## 5. `crt.service` / `crt.product` / `crt.promotion` ← `POS`

| Cột đích | Nguồn | Biến đổi | Khi NULL/thiếu |
|---|---|---|---|
| `service_id` | `POS.service.service_id` | `1:1` | `FAIL` |
| `service_code` | `POS.service.code` | `UPPER(TRIM())` | `FAIL` |
| `service_name` | `POS.service.name` | `TRIM` | `FAIL` |
| `category_l1`, `category_l2` | `POS.service.category_path` | Tách theo `>`, cấp 1 và cấp 2 | `N'(Chưa phân loại)'` |
| `standard_duration_min` | `POS.service.duration` | `CAST` `SMALLINT`, đơn vị phút | `FAIL` |
| `list_price_amount` | `POS.service.price` | `CAST` `DECIMAL(18,2)` | `0` |
| `price_band` | `DWH` | Phân vị giá: `<p33` → Economy, `<p66` → Standard, còn lại Premium | — |
| `product_id` … | `POS.product.*` | Tương tự | |
| `is_retail` | `POS.product.type` | `1` nếu bán lẻ cho khách | `0` |
| `is_consumable` | `POS.product.type` | `1` nếu vật tư dùng trong buồng | `0` |
| `promotion_id` … | `POS.promotion.*` | Tương tự | |
| `promotion_type` | `POS.promotion.kind` | `percent` / `amount` / `gift` / `bundle` | `'none'` |
| `discount_value` | `POS.promotion.value` | `DECIMAL(18,2)`. Với `percent` lưu số phần trăm (20 = 20%) | `0` |

## 6. `crt.booking` + `crt.booking_item` ← `OLTP` (CDC) + `APP`

| Cột đích | Nguồn | Biến đổi | Khi NULL/thiếu | DQ |
|---|---|---|---|---|
| `booking_id` | `OLTP.booking.id` | `1:1` | `FAIL` | DQ-BKG-001 |
| `customer_id` | `OLTP.booking.customer_id` → qua identity map | Gộp định danh | `-1` | DQ-BKG-002 |
| `salon_id` | `OLTP.booking.store_id` | `1:1` | `FAIL` | |
| `booking_channel` | `OLTP.booking.source` | `app` / `web` / `hotline` / `walk_in`, khác → `unknown` | `'unknown'` | |
| `booking_status` | `OLTP.booking.status` | Ánh xạ danh mục trạng thái | `FAIL` | DQ-BKG-003 |
| `booked_at` | `OLTP.booking.created_at` | Chuyển sang **UTC** | `FAIL` | DQ-BKG-004 |
| `requested_slot_at` | `OLTP.booking.slot_time` | Chuyển sang **UTC** | `FAIL` | |
| `cancel_reason` | `OLTP.booking.cancel_note` | `TRIM` | `NULL` | |
| `promotion_id` | `OLTP.booking.promo_id` | `1:1` | `-1` | |
| `source_event_id` | `APP.booking_created.event_id` | Khớp theo `booking_id`, dùng để đo độ trễ | `NULL` | |
| `booking_item_id` | `OLTP.booking_line.id` | `1:1` | `FAIL` | |
| `service_id` | `OLTP.booking_line.service_id` | `1:1` | `-1` | |
| `quantity` | `OLTP.booking_line.qty` | `DECIMAL(9,2)` | `1` | |
| `unit_price` | `OLTP.booking_line.price` | `DECIMAL(18,2)` | `0` | |
| `discount_amount` | `OLTP.booking_line.discount` | `DECIMAL(18,2)` | `0` | |
| `line_amount` | `DWH` | `quantity * unit_price - discount_amount` | — | DQ-BKG-005 |

> Nguồn chân lý của `booking` là `OLTP`. Sự kiện từ `APP` **không** dùng để tạo dòng, chỉ dùng để đối chiếu và đo độ trễ giữa lúc khách bấm và lúc dữ liệu về kho.

## 7. `crt.appointment` ← `POS`

| Cột đích | Nguồn | Biến đổi | Khi NULL/thiếu |
|---|---|---|---|
| `appointment_id` | `POS.appointment.id` | `1:1` | `FAIL` |
| `booking_id` | `POS.appointment.booking_ref` | `1:1` | `NULL` (khách walk-in) |
| `customer_id` | `POS.appointment.customer_id` → identity map | Gộp định danh | `-1` |
| `salon_id`, `employee_id`, `room_id` | `POS.appointment.*` | `1:1` | `-1` |
| `slot_at` | `POS.appointment.scheduled_at` | Chuyển sang **UTC** | `FAIL` |
| `checkin_at` | `POS.appointment.checkin_time` | Chuyển sang **UTC** | `NULL` = chưa đến |
| `appointment_status` | `POS.appointment.status` | `scheduled` / `checked_in` / `no_show` / `cancelled` | `FAIL` |
| `scheduled_duration_min` | `POS.appointment.duration` | `SMALLINT` | Lấy `standard_duration_min` của dịch vụ |
| `is_no_show` | `DWH` | `1` khi `checkin_at IS NULL` và đã qua `slot_at` + 30 phút | `0` |
| `wait_time_min` | `DWH` | `DATEDIFF(MINUTE, checkin_at, treatment_started_at)`; âm thì gán `0` | `0` |

## 8. `crt.treatment` ← `POS`

| Cột đích | Nguồn | Biến đổi | Khi NULL/thiếu |
|---|---|---|---|
| `treatment_id` | `POS.treatment.id` | `1:1` | `FAIL` |
| `appointment_id` | `POS.treatment.appointment_ref` | `1:1` | `NULL` |
| `customer_id` | Qua `appointment`, dự phòng `POS.treatment.customer_id` | Gộp định danh | `-1` |
| `salon_id`, `employee_id`, `room_id`, `service_id` | `POS.treatment.*` | `1:1` | `-1` |
| `started_at`, `completed_at` | `POS.treatment.*` | Chuyển sang **UTC** | `FAIL` cho `started_at` |
| `busy_minutes` | `DWH` | `DATEDIFF(MINUTE, started_at, completed_at)` | `FAIL` |
| `available_minutes` | `DWH` | Từ lịch làm việc KTV trong ngày, phân bổ theo ca | `0` |
| `standard_minutes` | `crt.service.standard_duration_min` | Tra theo `service_id` | `0` |
| `overrun_minutes` | `DWH` | `MAX(0, busy_minutes - standard_minutes)` | `0` |
| `is_upsell` | `DWH` | `1` nếu `service_id` **không** có trong `booking_item` của booking tương ứng | `0` |

> `available_minutes` là mẫu số của Therapist Năng suất, phải lấy từ **lịch làm việc**, không suy từ giờ mở cửa salon. Không có lịch làm việc thì chỉ số này không tính được — cần Vận hành cung cấp.

## 9. `crt.invoice` + `crt.invoice_line` ← `POS`

Đây là nguồn của `fact_sales_line` — luồng quan trọng nhất.

| Cột đích | Nguồn | Biến đổi | Khi NULL/thiếu | DQ |
|---|---|---|---|---|
| `invoice_id` | `POS.invoice.id` | `1:1` | `FAIL` | DQ-INV-001 |
| `invoice_no` | `POS.invoice.number` | `UPPER(TRIM())` | `FAIL` | DQ-INV-002 |
| `customer_id` | `POS.invoice.customer_id` → identity map | Gộp định danh | `-1` | |
| `salon_id` | `POS.invoice.store_id` | `1:1` | `FAIL` | |
| `service_at` | `POS.invoice.service_date` | UTC. **Cột chốt kỳ doanh thu** | Dùng `invoiced_at` nếu thiếu | DQ-INV-003 |
| `invoiced_at` | `POS.invoice.created_at` | UTC | `FAIL` | |
| `invoice_status` | `POS.invoice.status` | `paid` / `unpaid` / `void` | `FAIL` | |
| `invoice_line_id` | `POS.invoice_line.id` | `1:1`. **Định nghĩa độ hạt** | `FAIL` | DQ-UNIQ-001 |
| `invoice_line_no` | `POS.invoice_line.seq` | `SMALLINT` | Số thứ tự tự sinh | |
| `line_type` | `POS.invoice_line.item_type` | `service` / `product` | `FAIL` | DQ-INV-004 |
| `service_id` | `POS.invoice_line.item_id` khi `line_type='service'` | `1:1` | `-1` | |
| `product_id` | `POS.invoice_line.item_id` khi `line_type='product'` | `1:1` | `-1` | |
| `treatment_id` | `POS.invoice_line.treatment_ref` | `1:1` | `NULL` | |
| `employee_id` | `POS.invoice_line.performed_by`, dự phòng qua `treatment` | `1:1` | `-1` | |
| `quantity` | `POS.invoice_line.qty` | `DECIMAL(9,2)` | `FAIL` | |
| `unit_price_amount` | `POS.invoice_line.unit_price` | `DECIMAL(18,2)` | `0` | |
| `gross_amount` | `DWH` | `quantity * unit_price_amount` | — | DQ-INV-005 |
| `promo_discount_amount` | `POS.invoice_line.promo_discount` | `DECIMAL(18,2)`, luôn ≥ 0 | `0` | |
| `member_discount_amount` | `POS.invoice_line.member_discount` | `DECIMAL(18,2)`, luôn ≥ 0 | `0` | |
| `discount_amount` | `DWH` | `promo_discount_amount + member_discount_amount` | — | DQ-INV-006 |
| `net_amount` | `DWH` | `gross_amount - discount_amount` | — | DQ-INV-007 |
| `tax_amount` | `POS.invoice_line.vat_amount` | `DECIMAL(18,2)` | `0` | |
| `net_excl_tax_amount` | `DWH` | `net_amount - tax_amount` | — | |
| `cogs_amount` | `DWH` | Dòng sản phẩm: `quantity * product.cost_price`. Dòng dịch vụ: tổng `treatment_product_usage` **(cách tính chờ Kế toán chốt — xem ghi chú)** | `0` | DQ-INV-008 |
| `gross_margin_amount` | `DWH` | `net_amount - cogs_amount` | — | |

> **COGS dịch vụ chưa chốt.** Hai phương án: (a) chỉ vật tư tiêu hao; (b) vật tư + phân bổ công KTV. Hai cách cho biên lãi rất khác nhau. Đang tạm dùng (a); cần Kế toán quyết trước khi phát hành báo cáo lợi nhuận gộp.

## 10. `crt.payment` ← `POS` + `GW`

| Cột đích | Nguồn | Biến đổi | Khi NULL/thiếu | DQ |
|---|---|---|---|---|
| `payment_id` | `POS.payment.id` | `1:1` | `FAIL` | |
| `invoice_id` | `POS.payment.invoice_ref` | `1:1` | `FAIL` | DQ-PAY-001 |
| `customer_id` | Qua `invoice` | — | `-1` | |
| `salon_id` | Qua `invoice` | — | `FAIL` | |
| `payment_method_code` | `POS.payment.method` | Ánh xạ về danh mục `dim_payment_method` | `'UNKNOWN'` | |
| `payment_amount` | `POS.payment.amount` | `DECIMAL(18,2)` | `FAIL` | DQ-PAY-002 |
| `paid_at` | `POS.payment.paid_time` | UTC | `FAIL` | |
| `payment_status` | `POS.payment.status` | `completed` / `failed` / `refunded` / `pending` | `FAIL` | |
| `gateway_txn_id` | `GW.transaction.txn_id` | Khớp theo mã tham chiếu POS | `NULL` (tiền mặt) | DQ-PAY-003 |
| `fee_amount` | `GW.transaction.fee` | `DECIMAL(18,2)` | `0` | |
| `refund_amount` | `POS.payment.refund_amount` | `DECIMAL(18,2)` | `0` | |
| `net_cash_amount` | `DWH` | `payment_amount - refund_amount - fee_amount` | — | |

> Giao dịch `failed` **vẫn nạp** để tính tỷ lệ thất bại theo cổng thanh toán. Vì vậy mọi truy vấn doanh thu phải qua view `dm.v_fact_payment_completed`.

## 11. `crt.loyalty_transaction` ← `OLTP` (CDC)

| Cột đích | Nguồn | Biến đổi | Khi NULL/thiếu |
|---|---|---|---|
| `loyalty_txn_id` | `OLTP.point_ledger.id` | `1:1` | `FAIL` |
| `customer_id` | `OLTP.point_ledger.customer_id` → identity map | Gộp định danh | `FAIL` |
| `salon_id` | Qua `source_payment_id` → `invoice` | — | `-1` |
| `txn_type` | `OLTP.point_ledger.type` | `earn` / `redeem` / `expire` / `adjust` | `FAIL` |
| `point_delta` | `OLTP.point_ledger.points` | `INT`. **Dấu âm cho `redeem` và `expire`** | `FAIL` |
| `point_value_amount` | `DWH` | `ABS(point_delta) * tỷ giá điểm` | `0` |
| `source_payment_id` | `OLTP.point_ledger.ref_payment_id` | `1:1` | `NULL` |
| `txn_at` | `OLTP.point_ledger.created_at` | UTC | `FAIL` |

> Lưu **biến động** (`point_delta`), không lưu số dư. Số dư là semi-additive, tính bằng tổng luỹ tiến hoặc lấy từ `fact_customer_monthly_snapshot`.

## 12. `crt.feedback` ← `APP` + `MKT`

| Cột đích | Nguồn | Biến đổi | Khi NULL/thiếu |
|---|---|---|---|
| `feedback_id` | `APP.feedback_created.feedback_id` | `1:1` | `FAIL` |
| `customer_id` | `APP.feedback_created.customer_id` → identity map | Gộp định danh | `-1` |
| `treatment_id` | `APP.feedback_created.treatment_ref` | `1:1` | `NULL` |
| `salon_id`, `employee_id`, `service_id` | Qua `treatment` | — | `-1` |
| `feedback_channel` | Cố định theo luồng nạp | `app` / `sms` / `hotline` | `FAIL` |
| `rating` | `APP.feedback_created.rating` | `TINYINT` 1–5 | `FAIL` |
| `is_satisfied` | `DWH` | `1` khi `rating >= 4` | `0` |
| `is_dissatisfied` | `DWH` | `1` khi `rating <= 2` | `0` |
| `nps_score` | `MKT.survey.nps_answer` | `TINYINT` 0–10. **Câu hỏi riêng, không suy từ `rating`** | `NULL` |
| `is_promoter` | `DWH` | `1` khi `nps_score` 9–10 | `NULL` nếu không có `nps_score` |
| `is_detractor` | `DWH` | `1` khi `nps_score` 0–6 | `NULL` nếu không có `nps_score` |
| `comment_text` | `APP.feedback_created.comment` | `TRIM`. **Không đưa lên `dm`** | `NULL` |
| `has_comment` | `DWH` | `1` khi `comment_text` không rỗng | `0` |
| `feedback_at` | `APP.feedback_created.occurred_at` | UTC | `FAIL` |

> Khử trùng lặp theo `(customer_id, treatment_id)` — cùng một lượt điều trị có thể nhận phiếu từ cả app và SMS. Giữ phiếu có `feedback_at` sớm nhất.

## 13. `crt.ad_spend` ← `ADS`

| Cột đích | Nguồn | Biến đổi | Khi NULL/thiếu |
|---|---|---|---|
| `spend_date` | `ADS.insights.date_start` | `DATE` theo múi giờ tài khoản quảng cáo, đổi sang giờ VN | `FAIL` |
| `campaign_id` | `ADS.insights.campaign_id` | `1:1` | `FAIL` |
| `platform` | Cố định theo luồng nạp | `facebook` / `google` | `FAIL` |
| `spend_amount` | `ADS.insights.spend` | `DECIMAL(18,2)`. Quy về VND theo tỷ giá ngày | `0` |
| `impression_count` | `ADS.insights.impressions` | `BIGINT` | `0` |
| `click_count` | `ADS.insights.clicks` | `BIGINT` | `0` |
| `lead_count` | `ADS.insights.leads` | `INT` | `0` |

> **Nền tảng quảng cáo điều chỉnh lại số liệu trong 7 ngày.** Mỗi lần chạy phải nạp lại 7 ngày gần nhất theo cơ chế delete-insert phân vùng, không chỉ nạp ngày hôm qua.

## 14. `crt.campaign_send` ← `MKT`

| Cột đích | Nguồn | Biến đổi | Khi NULL/thiếu |
|---|---|---|---|
| `send_id` | `MKT.send_log.id` | `1:1` | `FAIL` |
| `campaign_id` | `MKT.send_log.campaign_id` | `1:1` | `FAIL` |
| `customer_id` | `MKT.send_log.recipient_id` → identity map | Gộp định danh | `-1` |
| `channel` | `MKT.send_log.channel` | `email` / `sms` / `zalo` / `push` | `FAIL` |
| `sent_at`, `opened_at`, `clicked_at` | `MKT.send_log.*` | UTC | `NULL` = chưa xảy ra |

## 15. `crt.service_view` ← `APP` + `GA4`

| Cột đích | Nguồn | Biến đổi | Khi NULL/thiếu |
|---|---|---|---|
| `session_id` | `APP.service_viewed.session_id`, dự phòng `GA4.ga_session_id` | `1:1` | `FAIL` |
| `customer_id` | `APP.service_viewed.customer_id` → identity map | Gộp định danh | `-1` (khách chưa đăng nhập) |
| `service_id` | `APP.service_viewed.service_id` | `1:1` | `-1` |
| `viewed_at` | `APP.service_viewed.occurred_at` | UTC | `FAIL` |
| `duration_sec` | `APP.service_viewed.duration_sec` | `INT` | `0` |

> Khối lượng lớn (~2,5 triệu dòng/năm ở 20 salon). Chỉ dùng ở mức tổng hợp cho phễu chuyển đổi. Quyết định có nạp vào SQL Server hay giữ ở Iceberg thuộc Giai đoạn 7.

---

## Ánh xạ `crt` sang `dm`

Phần lớn là tra khoá đại diện. Ba dạng biến đổi cần nêu rõ:

| Dạng | Quy tắc | Áp dụng cho |
|---|---|---|
| **Tra SCD2 theo thời điểm** | `JOIN dim ON dim.<bk> = crt.<bk> AND crt.<thời điểm giao dịch> >= dim.valid_from AND crt.<thời điểm giao dịch> < dim.valid_to` | `dim_customer`, `dim_salon`, `dim_employee`, `dim_service` |
| **Tra SCD1** | `JOIN dim ON dim.<bk> = crt.<bk>` | `dim_product`, `dim_promotion`, `dim_payment_method`, `dim_room`, `dim_campaign` |
| **Tra Junk dimension** | `JOIN dim_booking_junk ON` khớp đồng thời cả 5 cột cờ | `dim_booking_junk` |

Ba quy tắc bắt buộc khi tra khoá:

1. Luôn `LEFT JOIN`, không `INNER JOIN` — thiếu dimension không được làm mất dòng fact.
2. Luôn bọc `ISNULL(<dim>.<sk>, -1)` — cột FK trong fact khai báo `NOT NULL`.
3. Cột thời điểm dùng để tra SCD2 phải là **thời điểm nghiệp vụ xảy ra** (`service_at`, `booked_at`, `paid_at`), không phải thời điểm nạp.

### Khoá ngày và giờ

| Cột đích | Công thức |
|---|---|
| `<vai trò>_date_key` | `YEAR(x)*10000 + MONTH(x)*100 + DAY(x)` |
| `<vai trò>_time_key` | `DATEPART(HOUR, x)*60 + DATEPART(MINUTE, x)` |

---

## Ánh xạ danh mục

Các bảng ánh xạ dưới đây được lưu thành bảng tham chiếu trong `ctl` để sửa được không cần đổi code.

### Kênh thu hút khách (`acquisition_channel`)

| Giá trị nguồn | Giá trị chuẩn |
|---|---|
| `fb`, `facebook`, `fb_ads`, `meta` | `fb_ads` |
| `gg`, `google`, `google_ads`, `adwords` | `google_ads` |
| `walkin`, `walk-in`, `offline`, `store` | `walk_in` |
| `ref`, `referral`, `friend` | `referral` |
| `zalo`, `zalo_oa` | `zalo` |
| rỗng hoặc không khớp | `UNKNOWN` |

### Vùng miền (`region`) suy từ `city`

| Thành phố / tỉnh | Vùng |
|---|---|
| Hà Nội, Hải Phòng, Quảng Ninh, Bắc Ninh, … | `Bắc` |
| Đà Nẵng, Huế, Nha Trang, Quảng Nam, … | `Trung` |
| TP HCM, Cần Thơ, Bình Dương, Đồng Nai, … | `Nam` |
| không khớp | `UNKNOWN` |

### Trạng thái booking

| Giá trị nguồn | Giá trị chuẩn |
|---|---|
| `new`, `created`, `pending` | `created` |
| `confirmed`, `accepted` | `confirmed` |
| `cancelled`, `canceled`, `void` | `cancelled` |
| `done`, `completed`, `finished` | `completed` |
| `moved`, `rescheduled`, `changed` | `rescheduled` |

---

## Cột không có nguồn — tính trong kho

Các cột sau **không** lấy từ hệ thống nguồn, phải tính trong kho. Ghi rõ ở đây để không ai đi tìm nguồn không tồn tại.

| Cột | Bảng | Cách tính | Phụ thuộc |
|---|---|---|---|
| `is_first_visit` | `dim_booking_junk` | `1` khi không có hoá đơn nào trước `service_at` của cùng khách | `fact_sales_line` lịch sử |
| `rfm_segment` | `dim_customer` | Phân vị R, F, M trên 12 tháng gần nhất | `agg_customer_360` |
| `age_group` | `dim_customer` | Từ `date_of_birth`: `<25` / `25-34` / `35-44` / `45+` | — |
| `price_band` | `dim_service` | Phân vị giá niêm yết | Toàn bộ `crt.service` |
| `tenure_band` | `dim_employee` | Từ `hire_date` đến ngày chạy | — |
| `salon_size_band` | `dim_salon` | Từ `capacity_beds`: `<5` Small, `5-10` Medium, `>10` Large | — |
| `is_vn_holiday`, `is_tet_season` | `dim_date` | Từ `ctl.vn_holiday`, nạp thủ công mỗi năm | Không có công thức — Tết theo âm lịch |
| `available_minutes` | `fact_treatment` | Từ lịch làm việc KTV | **Cần Vận hành cung cấp lịch làm việc** |
| `days_since_last_visit` | `agg_customer_360` | `DATEDIFF` từ hoá đơn gần nhất tới ngày chạy | Tính lại mỗi ngày |
| `churn_probability`, `predicted_clv_amount` | `agg_customer_360` | Kết quả mô hình ML | Giai đoạn 8 |

---

## Phụ thuộc bên ngoài chưa có

Ba dữ liệu dưới đây **chưa có nguồn** và sẽ chặn các chỉ số tương ứng:

| Dữ liệu cần | Chặn chỉ số nào | Bên cung cấp |
|---|---|---|
| Lịch làm việc / phân ca KTV | Therapist Năng suất, Bed Occupancy | Vận hành |
| Giờ mở cửa từng salon theo ngày | Bed Occupancy | Vận hành |
| Cách tính COGS dịch vụ | Gross Margin, CLV | Kế toán |
| Danh mục ngày lễ và ngày nghỉ bù từng năm | So sánh cùng kỳ, dự báo nhu cầu | Nhân sự / Hành chính |
| Tỷ giá quy đổi điểm thưởng sang tiền | `point_value_amount` | CRM |
