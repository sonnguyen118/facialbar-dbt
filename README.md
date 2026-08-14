# Facial Bar — Data Platform

Thiết kế data platform và data warehouse cho chuỗi Facial Bar, tiếp cận theo góc nhìn Data Analyst / Data Architect: đi từ **Business → Data → Technology**.

## Tài liệu thiết kế

📄 **[Flow.md](Flow.md)** — tài liệu thiết kế đầy đủ (9 phần). Kèm sơ đồ kiến trúc gốc tại [Flow.jpg](Flow.jpg).

| Phần | Nội dung | Sản phẩm |
|---|---|---|
| 0 | Bức tranh tổng thể | Sơ đồ kiến trúc end-to-end |
| 1 | Business Layer | Domain, Process, Event Catalog |
| 2 | Data Modeling (logic) | ERD, bảng khai báo Grain, Bus Matrix, Star schema |
| 3 | Data Source & Flow | Source mapping, ma trận Batch/CDC/Streaming |
| 4 | Data Platform | Phân zone Data Lake, 4 tầng DWH |
| 5 | **Thiết kế DB vật lý** | **DDL đầy đủ 26 bảng, index & partition, volumetrics** |
| 6 | Analytics Layer | KPI dictionary, dashboard spec, ML use case |
| 7 | Architecture | Tech stack, security, data quality, governance |
| 8 | Roadmap | Kế hoạch 8 sprint |
| 9 | Phụ lục | Naming convention, glossary, checklist |

## Kiến trúc

```
Nguồn (POS · App · Ads · GA4 · Tổng đài)
        │
        ├── ETL theo lô ─────────────┐
        └── Kafka + Schema Registry ─┤
                                     ▼
                    S3 Data Lake (raw → cleansed → archive, Iceberg)
                                     │
                          Nạp và kiểm soát (watermark, idempotent)
                                     ▼
              SQL Server:  lnd → crt → [Cổng DQ] → dm → svg_bi
                                          │              │
                                    qtn (cách ly)   Superset / Power BI
```

Điều phối bằng **Airflow**. Chi tiết từng chặng ở [Flow.md](Flow.md).

## Mô hình dữ liệu

Star schema gồm **13 dimension + 10 fact + 1 bridge + 2 aggregate**, đủ cả 3 loại fact:

- **Transaction** — `fact_sales_line`, `fact_payment`, `fact_treatment`, `fact_booking_line`, `fact_appointment`, `fact_loyalty_txn`, `fact_feedback`, `fact_ad_spend`
- **Accumulating snapshot** — `fact_booking_lifecycle` (phễu đặt lịch → thanh toán, kèm thời gian mỗi chặng)
- **Periodic snapshot** — `fact_customer_monthly_snapshot` (chốt số dư điểm, hạng thẻ cuối tháng)

Dimension theo dõi lịch sử bằng **SCD Type 2**: `dim_customer`, `dim_salon`, `dim_employee`, `dim_service`.

DDL đầy đủ ở [docs/03-ddl/](docs/03-ddl/).

## Ba nguyên tắc xuyên suốt

1. **Grain là nền móng** — mỗi bảng trả lời "một dòng là gì" bằng đúng một câu, và câu đó được ghi thành `UNIQUE` constraint trong DDL.
2. **Idempotent là điều kiện sống còn** — chạy lại pipeline phải ra cùng kết quả.
3. **Business đi trước công nghệ** — Domain → Process → Event → Grain → Bus Matrix → Star schema → rồi mới đến kiểu dữ liệu, index, và công nghệ.

## Trạng thái

| Hạng mục | Trạng thái |
|---|---|
| Tài liệu thiết kế | ✅ Hoàn thành |
| DDL cho `dm` / `svg_bi` / `ctl` / `qtn` | ✅ Hoàn thành trong tài liệu |
| Triển khai dbt | ⏳ Chưa bắt đầu — xem [Roadmap Sprint 1–8](docs/09-roadmap/lo-trinh.md) |
| `fact_campaign_send`, `fact_service_view` | ⏳ Sprint 7 — xem [ranh giới phạm vi](docs/README.md) |

## Tech stack dự kiến

Airflow · Kafka + Schema Registry · Debezium · Kafka Connect · Amazon S3 · Apache Iceberg · Spark/Glue · SQL Server · dbt · Superset / Power BI

Lý do chọn từng công nghệ (kèm phương án đã cân nhắc và loại bỏ) ở [docs/08-operations/van-hanh.md](docs/08-operations/van-hanh.md#1-lựa-chọn-công-nghệ).
