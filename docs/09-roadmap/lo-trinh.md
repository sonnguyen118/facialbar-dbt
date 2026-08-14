# Lộ trình triển khai

Tám giai đoạn hai tuần, đi theo chiều dọc từng luồng nghiệp vụ. Bản trình phê duyệt kèm mốc kiểm soát của lãnh đạo: [../../Ban-Thiet-Ke-CSDL.md](../../Ban-Thiet-Ke-CSDL.md#5-lộ-trình-triển-khai).



**Nguyên tắc:** làm **theo chiều dọc** (một luồng nghiệp vụ chạy hết từ nguồn đến dashboard) thay vì theo chiều ngang (dựng hết ingestion cho mọi nguồn rồi mới làm modeling). Chiều dọc cho ra giá trị sử dụng được sau 4 tuần và phát hiện sớm mọi sai sót thiết kế.

| Sprint | Chủ đề | Việc làm | Điều kiện hoàn thành (DoD) |
|---|---|---|---|
| **S0** | Khám phá | Phỏng vấn business; kiểm kê nguồn; lấy mẫu dữ liệu; xác định owner | Có Source Inventory + 10 câu hỏi nghiệp vụ ưu tiên |
| **S1** | Business & Model logic | Chốt Domain, Process, Event Catalog; ERD; **Bảng khai báo Grain**; **Bus Matrix**; Star schema | Business ký xác nhận Event Catalog và định nghĩa KPI |
| **S2** | **Nền tảng DB + Lake** | S3 + phân zone; Iceberg ở cleansed; Airflow; **chuẩn kiểu dữ liệu (5.2)**; **`dim_date` + `dim_time` + partition scheme + toàn bộ bảng `ctl`** | `dim_date` đủ 10 năm kèm cờ ngày lễ/Tết; nạp 1 nguồn vào cleansed, chạy lại không sai |
| **S3** | Luồng dọc #1 — Doanh thu | POS → raw → cleansed → `lnd` → `crt` → **`dim_customer` + `dim_salon` (SCD2)** → **`fact_sales_line`** → `agg_revenue_daily_salon` → 1 dashboard | Doanh thu dashboard **khớp POS** 7 ngày liên tiếp; DQ-SCD-001/002 pass |
| **S4** | Streaming & CDC | Kafka + Schema Registry; Debezium cho OLTP; Kafka Connect S3 sink | Event `booking_created` đến cleansed trong dưới 5 phút |
| **S5** | Đủ mô hình chiều | Toàn bộ dim còn lại + `dim_booking_junk`; `fact_booking_line`, `fact_appointment`, `fact_treatment`, `fact_payment`, **`fact_booking_lifecycle`** | Chạy được đủ 4 dashboard vận hành; phễu chuyển đổi khớp số vận hành ghi tay |
| **S6** | Chất lượng & Cách ly | Bộ DQ rule 6 chiều; cổng kiểm tra; bảng `qtn`; đối soát tự động hằng ngày | Cổng chặn được lỗi cố tình gieo vào; báo cáo quarantine chạy hằng ngày |
| **S7** | Khách hàng & Marketing | Gộp định danh; `agg_customer_360`; `fact_customer_monthly_snapshot`; cohort; Ads/GA4; **chốt thiết kế `fact_campaign_send` và `fact_service_view`** (xem 5.11) | Repeat rate và CAC được business chấp nhận |
| **S8** | Vận hành & Nâng cao | Monitoring/alerting; Catalog + Lineage; bảng real-time; **sliding window + Iceberg maintenance**; ML use case đầu tiên (churn) | Runbook đầy đủ; on-call quay vòng; model churn có AUC ≥ 0,75 |

**Năm việc phải làm ngay từ Sprint 1–2, không được để sau:**
1. **Bảng khai báo Grain + Bus Matrix** — sửa grain về sau nghĩa là viết lại toàn bộ fact và mọi báo cáo.
2. **Chuẩn kiểu dữ liệu và collation (mục 5.2)** — đổi collation của database sau khi đã có dữ liệu là việc phải dựng lại toàn bộ; đổi kiểu cột đã có index là downtime.
3. **Partition scheme (mục 5.8)** — thêm phân vùng cho bảng đã có 40 triệu dòng phải ghi lại toàn bộ bảng.
4. **Bảng `ctl` (run_id, watermark, audit)** — thêm về sau nghĩa là mọi pipeline phải viết lại.
5. **Naming convention** — đổi tên về sau làm vỡ mọi dashboard và mọi câu SQL người dùng đã lưu.

> 💡 **Vì sao `dim_date` nằm ở Sprint 2 chứ không phải Sprint 5:** nó là dimension duy nhất mà **mọi** fact đều phụ thuộc (xem Bus Matrix, mục 2.7). Không có nó thì không nạp được fact nào. Cùng lý do, `partition scheme` phải có trước bảng fact đầu tiên vì `CREATE TABLE ... ON ps_...` cần scheme tồn tại sẵn.

---
---
