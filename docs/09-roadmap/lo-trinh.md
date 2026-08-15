# Lộ trình triển khai

Chín giai đoạn (GĐ 0 đến GĐ 8), mỗi giai đoạn hai tuần, tổng 18 tuần, đi theo chiều dọc từng luồng nghiệp vụ. Bản trình phê duyệt kèm mốc kiểm soát của lãnh đạo: [../../Ban-Thiet-Ke-CSDL.md](../../Ban-Thiet-Ke-CSDL.md#3-lộ-trình-nhân-lực-chi-phí).



**Nguyên tắc:** làm **theo chiều dọc** một luồng nghiệp vụ chạy hết từ nguồn đến báo cáo, thay vì theo chiều ngang là dựng xong thu nạp cho mọi nguồn rồi mới mô hình hoá. Chiều dọc cho ra giá trị sử dụng được sau 8 tuần (hết GĐ 3) và phát hiện sớm mọi sai sót thiết kế.

| Giai đoạn | Chủ đề | Việc làm | Điều kiện hoàn thành |
|---|---|---|---|
| **S0** | Khám phá | Phỏng vấn các bộ phận nghiệp vụ; kiểm kê nguồn; lấy mẫu dữ liệu; xác định người chịu trách nhiệm dữ liệu | Có danh mục nguồn dữ liệu và [10 câu hỏi nghiệp vụ ưu tiên](../00-business/nghiep-vu.md#3b-mười-câu-hỏi-nghiệp-vụ-ưu-tiên) được các bộ phận xếp thứ tự và ký |
| **S1** | Nghiệp vụ và mô hình logic | Chốt miền, quy trình, danh mục sự kiện; ERD; **bảng khai báo độ hạt**; **Bus Matrix**; star schema | Các bộ phận nghiệp vụ ký xác nhận danh mục sự kiện và định nghĩa chỉ tiêu |
| **S2** | **Nền tảng kho và hồ dữ liệu** | S3 và phân vùng; Iceberg ở `cleansed`; Airflow; **chuẩn kiểu dữ liệu (5.2)**; **`dim_date` + `dim_time` + partition scheme + toàn bộ bảng `ctl`** | `dim_date` đủ 10 năm kèm cờ ngày lễ/Tết; nạp 1 nguồn vào cleansed, chạy lại không sai |
| **S3** | Luồng dọc thứ nhất — Doanh thu | POS → raw → cleansed → `lnd` → `crt` → **`dim_customer` + `dim_salon` (SCD2)** → **`fact_sales_line`** → `agg_revenue_daily_salon` → 1 báo cáo | Doanh thu trên báo cáo **khớp POS** 7 ngày liên tiếp; `DQ-SCD-001` và `DQ-SCD-002` đạt |
| **S4** | Luồng sự kiện và bắt biến động dữ liệu | Kafka + Schema Registry; Debezium cho OLTP; Kafka Connect S3 sink | Sự kiện `booking_created` tới `cleansed` trong dưới 5 phút |
| **S5** | Đủ mô hình chiều | Toàn bộ dim còn lại + `dim_booking_junk`; `fact_booking_line`, `fact_appointment`, `fact_treatment`, `fact_payment`, **`fact_booking_lifecycle`** | Chạy được đủ 4 báo cáo vận hành; phễu chuyển đổi khớp số vận hành ghi tay |
| **S6** | Chất lượng và cách ly | 58 quy tắc chất lượng trên 7 nhóm; cổng kiểm tra; bảng `qtn`; đối soát tự động hằng ngày | Cổng chặn được lỗi cố tình gieo vào; báo cáo vùng cách ly chạy hằng ngày |
| **S7** | Khách hàng & Marketing | Gộp định danh; `agg_customer_360`; `fact_customer_monthly_snapshot`; nhóm khách theo tháng đến lần đầu; Ads/GA4; **chốt thiết kế `fact_campaign_send` và `fact_service_view`** (xem 5.11) | Tỷ lệ khách quay lại và chi phí thu hút khách mới được nghiệp vụ chấp nhận |
| **S8** | Vận hành và phân tích nâng cao | Giám sát và cảnh báo; danh mục dữ liệu và truy vết nguồn gốc; bảng thời gian thực; **cửa sổ trượt phân vùng và bảo trì Iceberg**; bài toán học máy đầu tiên là dự báo khách rời bỏ | Có sổ tay vận hành đầy đủ; phân ca trực quay vòng; mô hình dự báo khách rời bỏ đạt AUC ≥ 0,75 |

**Năm việc phải làm ngay từ Giai đoạn 1–2, không được để sau:**
1. **Bảng khai báo độ hạt và Bus Matrix** — sửa độ hạt về sau nghĩa là viết lại toàn bộ Fact và mọi báo cáo.
2. **Chuẩn kiểu dữ liệu và collation (mục 5.2)** — đổi collation của database sau khi đã có dữ liệu là việc phải dựng lại toàn bộ; đổi kiểu cột đã có index đòi dừng hệ thống.
3. **Sơ đồ phân vùng (mục 5.8)** — thêm phân vùng cho bảng đã có 40 triệu dòng phải ghi lại toàn bộ bảng.
4. **Bảng `ctl` — mã lần chạy, mốc thu nạp, vết kiểm toán** — thêm về sau nghĩa là mọi đường ống phải viết lại.
5. **Naming convention** — đổi tên về sau làm vỡ mọi báo cáo và mọi câu SQL người dùng đã lưu.

> **Căn cứ — `dim_date` nằm ở Giai đoạn 2 chứ không phải Giai đoạn 5:** nó là dimension duy nhất mà **mọi** fact đều phụ thuộc (xem Bus Matrix, mục 2.7). Không có nó thì không nạp được fact nào. Cùng lý do, `partition scheme` phải có trước bảng fact đầu tiên vì `CREATE TABLE ... ON ps_...` cần scheme tồn tại sẵn.

---
---
