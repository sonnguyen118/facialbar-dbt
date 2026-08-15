# Tra cứu — Quy ước đặt tên, thuật ngữ, checklist

Quy ước đặt tên đối tượng database, giải nghĩa thuật ngữ, bốn checklist: thiết kế bảng mới, chạy `CREATE TABLE`, phát hành chỉ tiêu, đưa đường ống lên môi trường chạy thật.



## 1. Quy ước đặt tên

| Đối tượng | Quy tắc | Ví dụ |
|---|---|---|
| Schema | Chữ thường, viết tắt ngắn | `lnd`, `crt`, `dm`, `svg_bi`, `ctl`, `qtn` |
| Bảng landing | `<nguồn>_<entity>` | `lnd.pos_invoice_line` |
| Bảng curated | `<entity>` (số ít) | `crt.invoice_line` |
| Dimension | `dim_<entity>` | `dm.dim_customer` |
| Fact | `fact_<nghiệp vụ>` | `dm.fact_sales_line` |
| Bảng tổng hợp | `agg_<chủ đề>_<chu kỳ>_<chiều>` | `svg_bi.agg_revenue_daily_salon` |
| View | `v_<mục đích>` | `v_pos_revenue_daily` |
| Bridge table | `bridge_<fact>_<dim>` | `dm.bridge_sales_promotion` |
| Surrogate key | `<entity>_sk` | `customer_sk` |
| Nghiệp vụ key | `<entity>_id` | `customer_id` |
| Khoá ngày | `<vai trò>_date_key` (INT: `20260814`) | `service_date_key`, `invoice_date_key`, `paid_date_key` |
| Khoá giờ | `<vai trò>_time_key` (SMALLINT: 0–1439) | `service_time_key`, `slot_time_key` |
| Cột ngày | `*_date` (DATE) | `business_date`, `last_visit_date` |
| Cột thời điểm | `*_at` (DATETIME2(3), **UTC**) | `created_at`, `occurred_at` |
| Cột boolean | `is_*` / `has_*` / `reached_*` | `is_current`, `is_first_visit`, `reached_payment` |
| Cột tiền | `*_amount` (DECIMAL(18,2)) | `net_amount`, `discount_amount` |
| Cột số lượng | `*_qty` / `*_count` / `*_minutes` | `quantity`, `visit_count`, `busy_minutes` |
| Cột kỹ thuật | Tiền tố `_` | `_run_id`, `_loaded_at`, `_src_file` |
| Ràng buộc & index | `PK_` / `UQ_` / `UX_` / `FK_` / `CK_` / `DF_` / `IX_` / `CCI_` | Bảng đầy đủ ở [mục 5.4](../03-ddl/00-init.md#4-chiến-lược-khoá-và-ràng-buộc) |
| Stored procedure | `usp_<hành động>_<đối tượng>` | `dm.usp_load_dim_customer` |
| Partition function/scheme | `pf_<cột>_<chu kỳ>` / `ps_<cột>_<chu kỳ>` | `pf_date_key_month`, `ps_date_key_month` |
| View | `v_<mục đích>` | `dm.v_fact_payment_completed` |
| Sự kiện | `<domain>_<động từ quá khứ>` | `booking_created` |
| Kafka topic | `facialbar.<domain>.v<n>` | `facialbar.booking.v1` |
| Airflow DAG | `dag_<hành động>_<đối tượng>_<chu kỳ>` | `dag_load_dwh_daily` |
| Phân vùng S3 | `<cột>=<giá trị>` (kiểu Hive) | `dt=2026-08-14/hour=09` |

> **Quy ước cho role-playing dimension:** khi một dim đóng nhiều vai trong cùng một fact, **tiền tố vai trò vào tên cột**, không đánh số. Dùng `service_date_key` / `invoice_date_key` chứ không dùng `date_key_1` / `date_key_2` — đọc câu SQL là hiểu ngay đang lọc theo ngày nào.

**Năm quy ước bắt buộc, không có ngoại lệ:**
1. **Mọi timestamp lưu ở UTC**, kiểu `DATETIME2(3)`. Chỉ đổi sang giờ Việt Nam ở tầng hiển thị. Trộn múi giờ trong dữ liệu là loại lỗi cực khó truy vết.
2. **Mọi số tiền dùng `DECIMAL(18,2)`.** Không `FLOAT` (sai số làm tròn), không `MONEY` (phép chia sai số tích luỹ).
3. **Mọi cột khoá/mã phải ghi đè `COLLATE Latin1_General_100_BIN2`** — collation `_AI` của database coi `"Lan"` và `"Làn"` là bằng nhau (xem [mục 5.2](../03-ddl/00-init.md#2-chuẩn-kiểu-dữ-liệu-và-collation)).
4. **Mọi dimension phải có dòng `-1` (Unknown member)**, seed ngay khi tạo bảng, trước khi nạp fact đầu tiên.
5. **Không dùng từ khoá SQL và không dùng tiếng Việt có dấu làm tên đối tượng.**

## 2. Thuật ngữ

| Thuật ngữ | Giải thích một câu |
|---|---|
| **ACID** | Đảm bảo giao dịch hoặc xong hẳn hoặc như chưa từng xảy ra |
| **Attribution** | Quy gán doanh thu cho kênh marketing nào |
| **Backfill** | Chạy lại đường ống dữ liệu cho một khoảng thời gian trong quá khứ |
| **Batch** | Xử lý dữ liệu theo lô, định kỳ |
| **Cardinality** | Số dòng ở bảng A ứng với một dòng ở bảng B |
| **CDC** | Bắt thay đổi của database qua log giao dịch |
| **Nhóm khách theo tháng đến lần đầu** | Nhóm khách có cùng mốc bắt đầu, theo dõi hành vi theo thời gian |
| **COGS** | Giá vốn hàng bán |
| **Columnstore** | Lưu dữ liệu theo cột để đọc phân tích nhanh |
| **Conformed dimension** | Dimension dùng chung nhiều fact, cho phép so sánh chéo |
| **hồ dữ liệu** | Kho lưu dữ liệu lớn, nhiều định dạng, gần dạng gốc |
| **kho phân tích** | Tập dữ liệu theo mô hình chiều, phục vụ phân tích |
| **Degenerate dimension** | Mã giao dịch nằm ngay trong fact |
| **Denormalize** | Gộp bảng để bớt join, đọc nhanh hơn |
| **Drilling across** | Tổng hợp từng fact về cùng độ hạt rồi mới join — cách so sánh 2 fact an toàn |
| **Fact** | Bảng chứa số đo của các sự kiện |
| **Fan-out** | Join làm nhân số dòng lên, gây sai tổng |
| **Freshness** | Độ mới của dữ liệu so với SLA |
| **độ hạt** | Một dòng trong bảng đại diện cho cái gì |
| **Hive-style partition** | Thư mục dạng `cột=giá trị` để engine lọc nhanh |
| **Iceberg** | Lớp metadata biến file Parquet trên S3 thành bảng có ACID và time travel |
| **Idempotent** | Chạy nhiều lần cho cùng một kết quả |
| **Immutable** | Đã ghi thì không sửa |
| **Thu nạp** | Đưa dữ liệu từ nguồn vào platform |
| **Late-arriving data** | Dữ liệu về muộn so với thời điểm nó xảy ra |
| **Leakage** | Dùng thông tin tương lai để dự đoán tương lai — làm model sai |
| **Least Privilege** | Chỉ cấp quyền tối thiểu cần thiết |
| **Lineage** | Đường đi của dữ liệu từ nguồn tới báo cáo |
| **LSN** | Số thứ tự thay đổi trong log của database |
| **MERGE** | Câu lệnh gộp: có thì update, không có thì insert |
| **OLAP** | Hệ thống tối ưu cho phân tích |
| **OLTP** | Hệ thống tối ưu cho giao dịch nghiệp vụ |
| **Parquet** | Định dạng file lưu theo cột, nén tốt |
| **Partition pruning** | Chỉ đọc phân vùng cần thiết |
| **PII** | Thông tin nhận diện cá nhân |
| **Vùng cách ly** | Vùng cách ly dòng dữ liệu lỗi |
| **Reconciliation** | Đối soát số liệu giữa DWH và nguồn |
| **Replay** | Nạp lại dữ liệu từ archive |
| **RFM** | Phân khúc khách theo Recency–Frequency–Monetary |
| **RPO / RTO** | Được mất tối đa bao nhiêu dữ liệu / bao lâu để phục hồi |
| **SCD** | Cơ chế xử lý dimension thay đổi theo thời gian |
| **Schema Evolution** | Đổi schema mà không làm vỡ hạ nguồn |
| **Schema Registry** | Nơi lưu và kiểm tra schema của sự kiện |
| **Snapshot** | Trạng thái nhất quán của bảng tại một thời điểm |
| **SoR (System of Record)** | Hệ thống được coi là nguồn chân lý cho một dữ liệu |
| **Star schema** | Fact ở giữa, dimension vây quanh |
| **Surrogate key** | Khoá số vô nghĩa do DWH sinh, dùng để join |
| **Time Travel** | Đọc bảng ở trạng thái quá khứ |
| **Watermark** | Dấu mốc "đã xử lý đến đâu" |

### Thuật ngữ riêng của thiết kế DB vật lý (Phần 5)

| Thuật ngữ | Giải thích một câu |
|---|---|
| **Accumulating snapshot** | Fact có 1 dòng cho cả quy trình, được UPDATE dần qua từng mốc |
| **Additive / Semi-additive / Non-additive** | Measure cộng được theo mọi chiều / trừ thời gian / không chiều nào |
| **Aligned index** | Index trên bảng phân vùng có chứa cột phân vùng → mới `SWITCH` được |
| **BCNF, 1NF, 2NF, 3NF** | Các dạng chuẩn hoá; `crt` ở 3NF, `dm` cố ý phá 3NF |
| **Bridge table** | Bảng cầu nối giải quan hệ nhiều-nhiều, có hệ số phân bổ |
| **Bus Matrix** | Bảng fact × dimension, artifact trung tâm của thiết kế chiều |
| **CCI (Clustered Columnstore Index)** | Index lưu theo cột, nén 5–10×, dùng cho fact lớn |
| **Collation** | Quy tắc so sánh và sắp xếp chuỗi; `_CI_AI` = không phân biệt hoa/thường và dấu |
| **Counting fact** | Cột luôn bằng 1 để mọi phép đếm thành `SUM`, tránh sai độ hạt |
| **Degenerate dimension** | Mã giao dịch nằm trong fact, không có dim riêng |
| **Filtered index** | Index chỉ trên tập dòng thoả điều kiện (ví dụ `WHERE is_current = 1`) |
| **Junk dimension** | Gom nhiều cờ nhỏ vào 1 dim để bớt cột nhân bản trong fact |
| **Outrigger** | Dim trỏ tới dim khác (snowflake cục bộ), dùng rất tiết chế |
| **Partition pruning** | Chỉ đọc phân vùng cần thiết |
| **Periodic snapshot** | Fact chốt trạng thái vào cuối mỗi kỳ |
| **RANGE RIGHT / LEFT** | Biên phân vùng thuộc về khoảng bên phải / bên trái |
| **Role-playing dimension** | Một dim đóng nhiều vai trong cùng fact (`service_date_key` vs `invoice_date_key`) |
| **Rowgroup** | Đơn vị nén của columnstore, cần ≥ 102.400 dòng mới hiệu quả |
| **Sliding window** | `SWITCH` phân vùng cũ ra ngoài để lưu trữ/xoá trong vài giây |
| **Temporal join** | Join fact với dim SCD2 theo khoảng `valid_from`/`valid_to` |
| **Unknown member** | Dòng `sk = -1` trong dim, dành cho khoá thiếu/không xác định |
| **Volumetrics** | Dự toán số dòng và dung lượng, cơ sở cho mọi quyết định index/partition |

## 3. Checklist

**Trước khi tạo một bảng mới (thiết kế logic):**
- [ ] Viết được độ hạt bằng đúng một câu, không có chữ "và"?
- [ ] Xác định được khoá duy nhất của độ hạt đó?
- [ ] Bảng này thuộc chặng nào trên hành trình khách hàng?
- [ ] Đã thêm vào **Bus Matrix** và kiểm tra dim nào là conformed?
- [ ] Nguồn chân lý (SoR) là hệ thống nào?
- [ ] Ai là Nghiệp vụ Owner, ai là Technical Owner?
- [ ] Chứa PII không? Nếu có, đã phân loại và bảo vệ chưa?
- [ ] Nạp lại 2 lần có ra kết quả giống nhau (idempotent)?
- [ ] Đã ghi vào danh mục dữ liệu kèm độ hạt và định nghĩa cột?

**Trước khi chạy `CREATE TABLE` (thiết kế vật lý — Phần 5):**
- [ ] Mọi cột dùng **kiểu chuẩn** ở mục 5.2? Không còn `FLOAT`/`MONEY`/`DATETIME` nào?
- [ ] Cột khoá/mã đã ghi đè `COLLATE ..._BIN2`?
- [ ] Đã phân loại từng measure là **additive / semi-additive / non-additive**? Không lưu tỷ lệ nào?
- [ ] Measure `NOT NULL DEFAULT 0`, FK `NOT NULL` dùng `-1`?
- [ ] Phân biệt đúng ba trạng thái `NULL` (chưa xảy ra) / `-1` (không xác định) / `0` (bằng không)?
- [ ] Có **UNIQUE index trên độ hạt**? Nếu bảng đã phân vùng thì index có **aligned** (chứa cột phân vùng)?
- [ ] Dimension đã seed dòng **Unknown member `-1`** chưa? (làm trước khi nạp fact đầu tiên)
- [ ] Dim SCD2 có `valid_from`/`valid_to`/`is_current`/`row_hash` + filtered UNIQUE index?
- [ ] Chọn đúng index chính theo mục 5.8? (CCI cho fact lớn, **rowstore** cho bảng bị UPDATE và bảng < 102.400 dòng)
- [ ] Khoá phân vùng là cột **được lọc nhiều nhất**, không phải cột tiện tay?
- [ ] Đã đặt tên tường minh cho **mọi** constraint và index?
- [ ] Đã dự toán volumetrics (dòng/năm và dung lượng) cho bảng này?
- [ ] Nếu dùng bridge table: đã có DQ quy tắc kiểm tra tổng `allocation_factor = 1` và đã ghi cảnh báo "cấm SUM measure của fact sau khi join bridge"?

**Trước khi phát hành một chỉ tiêu:**
- [ ] Công thức đã được nghiệp vụ ký xác nhận?
- [ ] Tính từ đúng một fact, không join 2 fact trực tiếp?
- [ ] Có cần `DISTINCT` không (kiểm tra lại độ hạt)?
- [ ] Xử lý NULL và chia cho 0 thế nào?
- [ ] Múi giờ và mốc chốt ngày (cut-off) đã rõ?
- [ ] Có mốc so sánh (cùng kỳ / mục tiêu)?
- [ ] Đã đối soát thủ công với nguồn ít nhất 1 lần?
- [ ] Đã ghi định nghĩa vào từ điển dữ liệu?

**Trước khi đưa đường ống dữ liệu lên production:**
- [ ] Có ghi `run_id` và watermark?
- [ ] Retry được, backfill được theo `business_date`?
- [ ] Có DQ quy tắc kèm phân mức BLOCK/WARN?
- [ ] Dòng lỗi đi vào vùng cách ly, không bị âm thầm bỏ qua?
- [ ] Cảnh báo gửi tới đúng người, kèm việc cần làm?
- [ ] Có runbook: hỏng thì làm gì, theo thứ tự nào?
- [ ] Đã kiểm tra dữ liệu về muộn và dữ liệu trùng?
- [ ] Đã đo thời gian chạy và đặt SLA?

---

## 4. Bản đồ từ nghiệp vụ đến bảng vật lý

Đây là bảng tổng hợp cuối cùng — đọc ngang một dòng là thấy hết đường đi của một khái niệm nghiệp vụ qua toàn bộ platform.

| Nghiệp vụ | Sự kiện | Nguồn | Thu nạp | Zone Lake | `crt` | `dm` | `svg_bi` | chỉ tiêu |
|---|---|---|---|---|---|---|---|---|
| Khách đăng ký | `customer_registered` | App, OLTP | Stream + CDC | raw/kafka, raw/cdc | `crt.customer` | `dim_customer` (SCD2) | `agg_customer_360` | Số khách mới, CAC |
| Xem dịch vụ | `service_viewed` | App, GA4 | Stream, Batch | raw/kafka, raw/ga4 | `crt.service_view` | `fact_service_view` ⏳ | `agg_funnel_daily` | Booking Conversion |
| Đặt lịch | `booking_created` | App, OLTP | Stream + CDC | raw/kafka, raw/cdc | `crt.booking`, `crt.booking_item` | `fact_booking_line` | `agg_funnel_daily` | Số booking, tỷ lệ huỷ |
| Đến salon | `customer_checked_in` | POS | Batch/CDC | raw/pos | `crt.appointment` | `fact_appointment`, `fact_booking_lifecycle` | `agg_funnel_daily` | No-show Rate, Wait Time |
| Làm dịch vụ | `treatment_completed` | POS | Batch/CDC | raw/pos | `crt.treatment` | `fact_treatment` | `agg_therapist_utilization_daily` | Năng suất, Up-sell |
| Thanh toán | `payment_completed` | POS, Gateway | Batch + API | raw/pos, raw/gateway | `crt.invoice_line`, `crt.payment` | `fact_sales_line`, `fact_payment` | `agg_revenue_daily_salon` | Net Revenue, ATV, Margin |
| Tích điểm | `points_earned` | OLTP | CDC | raw/cdc | `crt.loyalty_transaction` | `fact_loyalty_txn` | `agg_customer_360` | Tier Upgrade Rate |
| Đánh giá | `feedback_created` | App, SMS | Stream + Batch | raw/kafka | `crt.feedback` | `fact_feedback` | `agg_revenue_daily_salon` | CSAT (và NPS nếu thu) |
| Chiến dịch | `campaign_sent` | Marketing Platform, Ads | Batch API | raw/ads | `crt.campaign_send`, `crt.ad_spend` | `fact_campaign_send` ⏳, `fact_ad_spend` | `agg_cohort_retention` | ROAS, CAC |

⏳ = chưa thiết kế DDL chi tiết, thuộc Giai đoạn 7 — xem [ranh giới phạm vi ở mục 5.11](../README.md#tình-trạng-hoàn-thiện).

---

## 5. Ràng buộc thiết kế không được vi phạm

Bốn ràng buộc dưới đây áp dụng cho mọi bảng và mọi thay đổi về sau. Vi phạm bất kỳ ràng buộc nào đều tạo ra sai số **không sinh thông báo lỗi**, nên không thể phát hiện bằng kiểm thử thông thường.

| # | Ràng buộc | Thực thi bằng | Kiểm chứng |
|---|---|---|---|
| 1 | Mỗi bảng khai báo độ hạt bằng một câu, không có chữ "và" | `UNIQUE` constraint trên cột định nghĩa độ hạt ([5.4](../03-ddl/00-init.md#4-chiến-lược-khoá-và-ràng-buộc)) | DQ-UNIQ-001 |
| 2 | Quy trình nạp idempotent — chạy lại cho cùng kết quả | Delete-insert theo phân vùng hoặc `MERGE` theo khoá nghiệp vụ ([4.2](../06-platform/ho-du-lieu-va-kho.md#2-nạp-và-kiểm-soát)) | Chạy lại 3 lần, đối chiếu kết quả |
| 3 | Không lưu tỷ lệ trong fact, chỉ lưu tử số và mẫu số | Rà cột khi review DDL ([2.6](../01-erd/độ hạt.md#additivity-của-measure)) | Không có cột `*_pct`, `*_rate` trong `dm.fact_*` |
| 4 | Mọi dimension có dòng Unknown `-1`; FK trong fact `NOT NULL` | Seed dòng `-1` trước khi nạp fact đầu tiên ([5.5.3](../03-ddl/03-dm-dimension.md)) | Đếm dòng fact có `sk = -1`, ngưỡng cảnh báo 1% |

### Ba lỗi sai âm thầm thường gặp và cách chặn

| Lỗi | Biểu hiện | Chặn bằng |
|---|---|---|
| `COUNT(*)` trên bảng có độ hạt chi tiết hơn câu hỏi | Số lượt đặt lịch cao gấp số dịch vụ/lượt | Khai báo độ hạt trong catalog; dùng `COUNT(DISTINCT <khoá nghiệp vụ>)` |
| `AVG` của một tỷ lệ đã tính sẵn | Tỷ lệ lấp buồng toàn chuỗi lệch nhiều lần so với thực tế | Ràng buộc 3 ở trên |
| `INNER JOIN` với dimension thiếu khoá | Doanh thu hụt mà không có dấu vết | Ràng buộc 4 ở trên; `LEFT JOIN` + `ISNULL(..., -1)` khi nạp |

### Thứ tự thiết kế bắt buộc

Miền nghiệp vụ → Quy trình → Sự kiện → Entity → độ hạt → Bus Matrix → Star Schema → kiểu dữ liệu → index và phân vùng → lựa chọn công nghệ.

Đảo thứ tự này — chọn công nghệ hoặc dựng bảng vật lý trước khi chốt độ hạt và Bus Matrix — dẫn tới phải viết lại toàn bộ fact và mọi báo cáo phụ thuộc.
