# Chỉ tiêu, báo cáo và mô hình dự báo

Từ điển chỉ tiêu bốn nhóm, bộ báo cáo, các bài toán dự báo. Định nghĩa chỉ tiêu chốt tại `ctl.metric_definition` — xem [../03-ddl/06-ctl-qtn.md](../03-ddl/06-ctl-qtn.md#ctlmetric_definition).



> **Mục tiêu:** Biến Data Mart / Serving Data thành **Business Insight**, hỗ trợ ra quyết định và Machine Learning.

## 1. Từ điển chỉ tiêu

**Nguyên tắc:** Câu hỏi nghiệp vụ có trước → KPI → công thức → dữ liệu cần → bảng cần dựng.
Làm ngược lại (nhìn bảng có gì rồi vẽ chart) sẽ ra hàng chục dashboard không ai dùng.

Bắt đầu bằng câu hỏi thật của ban lãnh đạo: *"Các salon đang kinh doanh hiệu quả như thế nào? Facial Bar kiếm được bao nhiêu tiền?"*

### KPI Dictionary

**Nhóm Tài chính**

| KPI | Là gì | Công thức | Nguồn | Lưu ý grain |
|---|---|---|---|---|
| **Net Revenue** | Doanh thu sau giảm giá | `SUM(net_amount)` | `fact_sales_line` | Không cộng thêm `fact_payment` — sẽ đếm 2 lần |
| **Cash Collected** | Tiền thực thu | `SUM(payment_amount)` | `fact_payment` | Khác Net Revenue khi có trả trước/nợ |
| **ATV** (Average Ticket Value) | Giá trị hoá đơn bình quân | `SUM(net_amount) / COUNT(DISTINCT invoice_no)` | `fact_sales_line` | **Phải** có `DISTINCT` — grain là dòng hoá đơn, không phải hoá đơn |
| **ARPU** | Doanh thu bình quân/khách | `SUM(net_amount) / COUNT(DISTINCT customer_sk)` | `fact_sales_line` | Chốt rõ kỳ tính |
| **Gross Margin** | Lãi gộp | `SUM(net_amount − cogs_amount)` | `fact_sales_line` | **Cần Kế toán chốt** cách tính COGS dịch vụ: chỉ vật tư, hay vật tư + phân bổ công KTV. Hai cách cho biên lãi rất khác nhau |
| **Discount Rate** | Tỷ lệ giảm giá | `SUM(discount_amount) / SUM(gross_amount)` | `fact_sales_line` | Cảnh báo nếu > 20% |

**Nhóm Vận hành**

| KPI | Là gì | Công thức | Nguồn |
|---|---|---|---|
| **Booking Conversion** | Tỷ lệ xem → đặt | Tổng hợp **từng fact về cùng grain `ngày × salon`** rồi mới chia: `booking_cnt / session_cnt`. **Không** chia trực tiếp giữa hai fact khác grain | `agg_funnel_daily` |
| **No-show Rate** | Tỷ lệ khách không đến | `COUNT(no-show) / COUNT(appointment)` | `fact_appointment` |
| **Cancellation Rate** | Tỷ lệ huỷ | `COUNT(cancelled) / COUNT(booking)` | `fact_booking_line` |
| **Therapist Utilization** | Tỷ lệ giờ KTV có khách | `SUM(actual_duration) / SUM(scheduled_hours)` | `fact_treatment` + lịch làm việc |
| **Bed Occupancy** | Tỷ lệ lấp buồng | `giờ buồng có khách / giờ buồng mở cửa` | `fact_treatment` + `dim_room` |
| **Up-sell Rate** | Tỷ lệ bán thêm tại chỗ | `COUNT(treatment không có trong booking) / COUNT(treatment)` | `fact_treatment` |
| **Avg Wait Time** | Thời gian chờ bình quân | `AVG(treatment_start − checkin_at)` | `fact_booking_lifecycle` |

**Nhóm Khách hàng**

| KPI | Là gì | Công thức | Nguồn |
|---|---|---|---|
| **New vs Returning** | Cơ cấu khách mới/cũ | Đếm theo cờ `is_first_visit` | `fact_sales_line` |
| **Repeat Rate** | Tỷ lệ khách quay lại | `khách có ≥ 2 lượt TRONG KỲ / khách có ≥ 1 lượt trong kỳ`. **Bắt buộc nêu kỳ** (đề xuất: 12 tháng gần nhất) | `fact_sales_line` |
| **Cohort Retention** | Giữ chân theo nhóm tháng đầu | Ma trận cohort × tháng thứ N | `agg_cohort_retention` |
| **Churn Rate** | Tỷ lệ mất khách | `khách vượt ngưỡng vắng / khách active`. **Ngưỡng phải suy ra từ dữ liệu**, xem ghi chú dưới | `agg_customer_360` |
| **CSAT** | Tỷ lệ hài lòng (thang 1–5 sao) | `SUM(is_satisfied) / SUM(response_count)` với hài lòng = rating ≥ 4 | `fact_feedback` |
| **NPS** | Chỉ số thiện cảm (thang 0–10) | `%promoter − %detractor`. **Chỉ tính trên phiếu có `nps_score`**, xem ghi chú dưới | `fact_feedback` |
| **Tier Upgrade Rate** | Tỷ lệ nâng hạng | `số nâng hạng trong kỳ / số thành viên đầu kỳ` | `fact_loyalty_txn` |

**Ba ghi chú phương pháp cho nhóm chỉ số khách hàng** — đây là phần dễ làm sai nhất và thuộc trách nhiệm của DA, không phải của kỹ thuật:

**1. NPS và CSAT là hai thang đo khác nhau, không quy đổi được cho nhau.**
NPS được định nghĩa trên thang **0–10** (promoter 9–10, detractor 0–6, còn lại là passive). Thang **1–5 sao** dùng để tính **CSAT**, không tính được NPS. Nếu công ty muốn có NPS thì phải **thêm một câu hỏi riêng** *"Bạn có sẵn sàng giới thiệu Facial Bar cho bạn bè?"* trên thang 0–10 — vì vậy `fact_feedback` có cột `nps_score` riêng, cho phép NULL.
→ Báo cáo hài lòng theo sao phải gọi là **CSAT**. Gọi là NPS là sai định nghĩa và sẽ không so sánh được với số liệu ngành.

**2. Ngưỡng churn phải suy ra từ phân bố khoảng cách giữa hai lượt đến, không được chọn theo cảm tính.**
Cách làm: tính khoảng cách ngày giữa các lượt liên tiếp của toàn bộ khách có ≥ 2 lượt, lấy **phân vị 80–90%** làm ngưỡng.

```sql
WITH gaps AS (
    SELECT customer_sk,
           DATEDIFF(DAY,
               LAG(f.full_date) OVER (PARTITION BY s.customer_sk ORDER BY f.full_date),
               f.full_date) AS gap_days
    FROM   dm.fact_sales_line s
    JOIN   dm.dim_date f ON f.date_key = s.service_date_key
)
SELECT DISTINCT
       PERCENTILE_CONT(0.80) WITHIN GROUP (ORDER BY gap_days) OVER () AS p80_days,
       PERCENTILE_CONT(0.90) WITHIN GROUP (ORDER BY gap_days) OVER () AS p90_days
FROM   gaps WHERE gap_days IS NOT NULL;
```
Con số 90 ngày chỉ là **giá trị khởi tạo tạm**; phải thay bằng kết quả truy vấn trên và **rà lại mỗi 6 tháng**. Ghi rõ ngưỡng đang dùng vào Data Dictionary.

**3. CLV phải tính trên lãi gộp, không tính trên doanh thu.**
Công thức doanh thu × tần suất × tuổi thọ trả lời câu "khách mang lại bao nhiêu **doanh thu**", không trả lời "khách mang lại bao nhiêu **giá trị**" — và sẽ khiến ngân sách marketing bị phân bổ sai về phía nhóm khách chi nhiều nhưng biên lãi mỏng.

| Mức | Công thức | Khi nào dùng |
|---|---|---|
| Thô | `ATV × tần suất/năm × số năm dự kiến` | Chỉ để so sánh tương đối giữa các phân khúc |
| **Chuẩn tối thiểu** | `Lãi gộp bình quân/lượt × tần suất/năm × số năm dự kiến` | **Khuyến nghị dùng cho báo cáo** |
| Có chiết khấu dòng tiền | Cộng dồn lãi gộp từng năm, chiết khấu về hiện tại | Khi dùng cho quyết định đầu tư dài hạn |

"Số năm dự kiến" cũng không được đặt tuỳ ý — suy ra từ `1 / tỷ lệ churn năm`.

**Nhóm Marketing**

| KPI | Là gì | Công thức | Nguồn |
|---|---|---|---|
| **CAC** | Chi phí thu hút 1 khách mới | `SUM(ad_spend) / COUNT(khách mới)` | `fact_ad_spend` + `fact_sales_line` |
| **ROAS** | Doanh thu trên 1 đồng quảng cáo | `revenue quy gán / ad_spend` | `fact_ad_spend` + `fact_sales_line` |
| **CTR** | Tỷ lệ bấm vào | `clicks / impressions` | `fact_ad_spend` |
| **Campaign Conversion** | Tỷ lệ chiến dịch ra booking | `booking từ campaign / số gửi` | `fact_campaign_send` |

> ⚠️ **CAC và ROAS cần "Attribution" (quy gán) — khái niệm dễ gây tranh chấp nhất.** Khách thấy Facebook Ads ngày 1, tìm Google ngày 3, đặt lịch ngày 7. Doanh thu tính cho ai? Phải chốt **một** mô hình quy gán và ghi vào catalog: `last_non_direct_click` là mặc định hợp lý cho chuỗi bán lẻ. Không chốt → marketing và tài chính sẽ tranh luận mãi không xong.

---

## 2. Báo cáo và phân tích

**Bốn cấp độ phân tích:**

| Cấp | Tên | Câu hỏi | Ví dụ Facial Bar | Công cụ |
|---|---|---|---|---|
| 1 | **Descriptive** (mô tả) | Chuyện gì đã xảy ra? | Doanh thu tháng 7 là 4,2 tỷ | Dashboard |
| 2 | **Diagnostic** (chẩn đoán) | Vì sao lại như vậy? | Giảm 12% vì salon Q7 mất 2 KTV | Drill-down, phân rã |
| 3 | **Predictive** (dự đoán) | Sắp tới sẽ thế nào? | Tháng 9 dự báo 3,9 tỷ | ML |
| 4 | **Prescriptive** (đề xuất) | Nên làm gì? | Gửi voucher cho 850 khách nguy cơ rời | ML + rule |

### Bộ Dashboard

| Dashboard | Người dùng | Nội dung chính | Tần suất |
|---|---|---|---|
| **Executive Overview** | CEO, COO | Revenue, Margin, so cùng kỳ, xếp hạng salon | Hằng ngày |
| **Salon Performance** | Quản lý salon | Doanh thu, lấp buồng, no-show, CSAT theo chi nhánh | Hằng ngày |
| **Service & Product Mix** | Sản phẩm | Dịch vụ bán chạy, gắn kèm, biên lãi | Hằng tuần |
| **Therapist Productivity** | Vận hành, HR | Utilization, doanh thu/KTV, điểm đánh giá | Hằng tuần |
| **Customer & Retention** | CRM, Marketing | Cohort, churn, CLV, phân khúc RFM | Hằng tuần |
| **Marketing ROI** | Marketing | CAC, ROAS theo kênh, hiệu quả chiến dịch | Hằng ngày |
| **Data Quality Ops** | Data team | Trạng thái DQ, số dòng quarantine, độ trễ | Hằng ngày |
| **Live Salon Monitor** | Lễ tân, quản lý | Khách trong salon, buồng trống, doanh thu tạm tính | Real-time |

**Chuẩn tối thiểu cho mọi dashboard:**
- Ghi rõ **thời điểm cập nhật cuối** ("Dữ liệu đến 06:40, 14/08/2026").
- Ghi rõ **định nghĩa KPI** (tooltip hoặc trang chú thích), khớp với catalog.
- Cho phép **xuất dữ liệu** để người dùng tự kiểm tra.
- Không quá **7 biểu đồ** trên một trang.
- Luôn có **mốc so sánh**: cùng kỳ năm trước, kỳ trước, hoặc mục tiêu.

---

## 3. Bài toán dự báo

Không chỉ nhìn dữ liệu quá khứ, mà dùng dữ liệu để **dự đoán** hoặc **tự động ra quyết định**.

| # | Use case | Câu hỏi nghiệp vụ | Loại bài toán | Nhãn (label) | Đặc trưng chính | Hành động tương ứng |
|---|---|---|---|---|---|---|
| 1 | **Churn Prediction** | Khách nào có khả năng không quay lại? | Phân loại nhị phân | Không có giao dịch trong ngưỡng vắng đã hiệu chỉnh | Ngày kể từ lần cuối, tần suất, hạng thẻ, CSAT, tỷ lệ huỷ | Chiến dịch giữ khách có mục tiêu |
| 2 | **CLV Prediction** | Khách này mang lại bao nhiêu doanh thu tương lai? | Hồi quy | Tổng doanh thu 12 tháng tới | Lịch sử chi tiêu, cơ cấu dịch vụ, kênh thu hút | Phân bổ ngân sách theo giá trị khách |
| 3 | **Next Best Service** | Nên gợi ý dịch vụ nào? | Gợi ý | Dịch vụ mua ở lần kế tiếp | Chuỗi dịch vụ đã dùng, loại da, mùa | Gợi ý trong app, kịch bản up-sell |
| 4 | **No-show Prediction** | Lịch hẹn nào có nguy cơ khách không đến? | Phân loại nhị phân | Đã no-show | Lịch sử no-show, thời gian đặt trước, khung giờ, thời tiết | Nhắc thêm, overbook có kiểm soát |
| 5 | **Demand Forecast** | Ngày mai cần bao nhiêu KTV? | Chuỗi thời gian | Số lượt treatment theo ngày × salon | Tính mùa, ngày lễ, chi quảng cáo | Xếp ca làm việc |
| 6 | **Customer Segmentation** | Có mấy nhóm khách? | Phân cụm / RFM | — (không giám sát) | Recency, Frequency, Monetary | Cá nhân hoá truyền thông |

### Nguyên tắc dữ liệu cho ML

**1. Data Leakage (rò rỉ dữ liệu):** dùng thông tin của tương lai để dự đoán tương lai.
Ví dụ: dự đoán churn mà lại đưa vào feature "tổng chi tiêu cả năm" — trong đó có cả chi tiêu **sau** mốc dự đoán. Model sẽ đạt độ chính xác 95% khi thử nghiệm và thất bại hoàn toàn khi lên production.
→ **Cách phòng:** mọi feature phải kèm mốc thời gian, và chỉ được dùng dữ liệu có `occurred_at <= feature_cutoff_date`.

**2. Training/Serving Skew (lệch giữa huấn luyện và triển khai):** feature lúc train tính bằng SQL trên warehouse, lúc chạy thật tính bằng Python trên app → hai công thức khác nhau chút ít → dự đoán sai.
→ **Cách phòng:** tính feature **một lần duy nhất** ở `svg_bi.agg_customer_360` (feature store đơn giản), cả train và serve đều đọc từ đó.

---
---
