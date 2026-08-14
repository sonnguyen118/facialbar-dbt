# Bảng khai báo Grain

Grain là độ hạt của bảng: một dòng đại diện cho cái gì. Đây là khai báo bắt buộc cho mọi bảng, và phải được thực thi bằng `UNIQUE` constraint trong DDL — không chỉ ghi trong tài liệu.

Ràng buộc thực thi: [03-ddl/00-init.md mục 4](../03-ddl/00-init.md#4-chiến-lược-khoá-và-ràng-buộc). Quy tắc kiểm tra: [05-quality/dq-rules.md mục 4](../05-quality/dq-rules.md#4-uniqueness--duy-nhất).



Grain (độ hạt) là câu trả lời cho câu hỏi **"MỘT DÒNG trong bảng này đại diện cho điều gì?"** — trả lời bằng đúng một câu, không có chữ "và".

grain quyết định mọi phép đếm và mọi phép tổng. Sai grain → `COUNT(*)` và `SUM()` sai → toàn bộ báo cáo sai, nhưng **không có lỗi nào báo ra**. Loại sai này không sinh thông báo lỗi nên không phát hiện được bằng kiểm thử thông thường.

### Bảng khai báo Grain (bắt buộc có cho mọi bảng)

| Bảng | 1 dòng = | Khoá duy nhất | Đo được gì |
|---|---|---|---|
| `dim_customer` | 1 **phiên bản** của 1 khách hàng | customer_id + valid_from | — (dimension) |
| `dim_salon` | 1 phiên bản của 1 salon | salon_id + valid_from | — |
| `dim_service` | 1 dịch vụ | service_id | — |
| `fact_booking_line` | 1 **dịch vụ được đặt** trong 1 booking | booking_item_id | số dịch vụ đặt, giá trị đặt |
| `fact_appointment` | 1 **lịch hẹn** | appointment_id | số lịch hẹn, no-show, thời gian chờ |
| `fact_treatment` | 1 **dịch vụ đã thực hiện** | treatment_id | số lượt làm, phút phục vụ |
| `fact_sales_line` | 1 **dòng hoá đơn** | invoice_line_id | **doanh thu**, giảm giá, COGS |
| `fact_payment` | 1 **lần chuyển tiền** | payment_id | tiền thực thu |
| `fact_loyalty_txn` | 1 lần điểm biến động | loyalty_txn_id | điểm tích, điểm tiêu |
| `fact_feedback` | 1 phiếu đánh giá | feedback_id | rating, CSAT, NPS (nếu có thu) |
| `fact_ad_spend` | 1 ngày × chiến dịch × nền tảng | (date_key, campaign_sk, platform) | chi phí, impression, click |

### Double counting — minh hoạ bằng số

Khách **Lan** đặt 1 booking gồm 2 dịch vụ: *Hydrafacial 1.200.000đ* và *Massage vai gáy 300.000đ*.

Bảng `fact_booking_line` (grain = 1 dịch vụ đặt) sẽ có **2 dòng**:

| booking_id | booking_item_id | customer | service | line_amount |
|---|---|---|---|---|
| B-001 | BI-001 | Lan | Hydrafacial | 1.200.000 |
| B-001 | BI-002 | Lan | Massage vai gáy | 300.000 |

| Câu hỏi nghiệp vụ | Câu SQL **SAI** | Kết quả sai | Câu SQL **ĐÚNG** | Kết quả đúng |
|---|---|---|---|---|
| Có bao nhiêu booking? | `COUNT(*)` | **2** ❌ | `COUNT(DISTINCT booking_id)` | **1** ✅ |
| Bao nhiêu khách đã đặt? | `COUNT(customer_id)` | **2** ❌ | `COUNT(DISTINCT customer_id)` | **1** ✅ |
| Tổng giá trị đặt? | `SUM(line_amount)` | **1.500.000** ✅ | `SUM(line_amount)` | 1.500.000 ✅ |

→ Cùng một bảng: `SUM` thì đúng, `COUNT` thì sai. **Đó chính là hệ quả của grain.**

**Trường hợp nghiêm trọng hơn — join làm nhân dòng (fan-out):**

Nếu join `fact_sales_line` (3 dòng hoá đơn) với `fact_payment` (khách trả 2 lần: 1 lần thẻ, 1 lần tiền mặt) mà không qua bảng phân bổ:

```
3 dòng × 2 lần trả = 6 dòng  →  SUM(revenue) bị nhân lên 2 lần
```

**Cách phòng 3 lớp:**
1. **Không bao giờ join 2 fact trực tiếp với nhau.** Muốn so sánh thì tổng hợp từng fact về cùng grain trước, rồi mới join (kỹ thuật *drilling across*).
2. Luôn đi qua bảng phân bổ (`payment_allocation`) khi quan hệ là N:N.
3. Ghi rõ grain vào comment của bảng và vào data catalog.

### Ba loại Fact — chọn đúng loại theo câu hỏi nghiệp vụ

| Loại Fact | Là gì | Ví dụ Facial Bar | Khi nào dùng |
|---|---|---|---|
| **Transaction Fact** | 1 dòng = 1 sự kiện xảy ra | `fact_sales_line`, `fact_payment` | Đo lượng, đo tiền theo thời gian |
| **Periodic Snapshot** | 1 dòng = trạng thái của 1 đối tượng vào cuối mỗi kỳ | `fact_customer_monthly_snapshot` (số dư điểm, hạng thẻ cuối tháng) | Đo trạng thái tích luỹ, đo "bao nhiêu khách đang active" |
| **Accumulating Snapshot** | 1 dòng = 1 quy trình, **cập nhật dần** qua các mốc | `fact_booking_lifecycle` (booked_at → confirmed_at → checkin_at → treatment_at → paid_at) | Đo **thời gian giữa các bước** và tỷ lệ rơi ở từng bước |

> 💡 `fact_booking_lifecycle` là bảng đắt giá nhất cho vận hành: nó cho ra ngay funnel *đặt lịch → xác nhận → đến → làm → trả tiền*, kèm số ngày/giờ ở mỗi chặng. Đây là bảng duy nhất được phép **UPDATE** trong datamart.

---

---

## Additivity của measure

Additivity cho biết một số đo có được phép `SUM` theo từng chiều hay không.

nó quyết định **cột nào được lưu trong fact**. Chọn sai kiểu measure thì mọi báo cáo tổng hợp về sau đều sai, mà lại là sai âm thầm.

| Loại | Cộng được theo | Ví dụ Facial Bar | Cách tổng hợp đúng |
|---|---|---|---|
| **Additive** | **Mọi** chiều | `net_amount`, `discount_amount`, `quantity`, `cogs_amount`, `busy_minutes` | `SUM()` bình thường |
| **Semi-additive** | Mọi chiều **trừ thời gian** | `point_balance` (số dư điểm), `active_member_count`, `inventory_qty` | Lấy **giá trị cuối kỳ**, không `SUM` qua các tháng |
| **Non-additive** | **Không** chiều nào | `rating`, `discount_pct`, `utilization_pct`, `nps` | Lưu **tử số + mẫu số**, tính lại tỷ lệ khi tổng hợp |

### Quy tắc thiết kế #1: KHÔNG lưu tỷ lệ trong fact

```sql
-- ❌ SAI: lưu sẵn tỷ lệ
CREATE TABLE dm.fact_treatment (treatment_id BIGINT, utilization_pct DECIMAL(5,2), ...);

-- ✅ ĐÚNG: lưu tử số và mẫu số, để tỷ lệ được tính lại ở mọi mức tổng hợp
CREATE TABLE dm.fact_treatment (treatment_id BIGINT, busy_minutes INT, available_minutes INT, ...);
-- Tổng hợp:  SUM(busy_minutes) * 1.0 / NULLIF(SUM(available_minutes), 0)
```

**Con số chứng minh** — hai salon có quy mô rất khác nhau:

| Salon | busy_minutes | available_minutes | utilization |
|---|---|---|---|
| Q1 (mới mở, 1 buồng) | 90 | 100 | 90,0% |
| Q7 (10 buồng) | 10 | 900 | 1,1% |

- Cách sai: `AVG(90,0% ; 1,1%)` = **45,6%**
- Cách đúng: `(90+10) / (100+900)` = **10,0%**

Lệch **4,5 lần**. Và không có thông báo lỗi nào.

### Quy tắc thiết kế #2: lưu BIẾN ĐỘNG, không lưu SỐ DƯ

Số dư điểm là semi-additive nên rất dễ dùng sai. Khách có 500 điểm cuối tháng 1, 700 cuối tháng 2, 900 cuối tháng 3 → `SUM = 2.100 điểm` là con số vô nghĩa; đáp án đúng là **900**.

→ Vì vậy `fact_loyalty_txn` lưu `point_delta` (`+150`, `−500`) là **additive**, và số dư được tính bằng tổng luỹ tiến:

```sql
SELECT customer_sk, date_key,
       SUM(point_delta) OVER (PARTITION BY customer_sk ORDER BY date_key
                              ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS point_balance
FROM   dm.fact_loyalty_txn;
```

Số dư cuối tháng vẫn được chốt sẵn vào `fact_customer_monthly_snapshot` để báo cáo khỏi phải tính luỹ tiến trên toàn bộ lịch sử — đây chính là lý do tồn tại của **periodic snapshot fact**.

---
