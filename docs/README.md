# Đặc tả thiết kế — Facial Bar Data Platform

Tài liệu luồng tổng thể: [../Flow.md](../Flow.md) · Bản trình phê duyệt: [../Ban-Thiet-Ke-CSDL.md](../Ban-Thiet-Ke-CSDL.md)

`Flow.md` chỉ mô tả luồng. Mọi nội dung đi sâu nằm ở đây.

## Cấu trúc

| Thư mục | Nội dung | Đối tượng đọc |
|---|---|---|
| [00-business/](00-business/) | Hành trình khách hàng, 14 miền nghiệp vụ, 6 quy trình, danh mục sự kiện | Phân tích dữ liệu, Nghiệp vụ |
| [01-erd/](01-erd/) | Thực thể, quan hệ, độ hạt, star schema, bus matrix | Phân tích dữ liệu |
| [02-mapping/](02-mapping/) | Ánh xạ nguồn sang đích ở mức cột | Kỹ sư dữ liệu, Phân tích dữ liệu |
| [03-ddl/](03-ddl/) | DDL 92 bảng theo từng schema | Quản trị cơ sở dữ liệu, Kỹ sư dữ liệu |
| [04-etl/](04-etl/) | Quy trình nạp, dữ liệu khởi tạo | Kỹ sư dữ liệu |
| [05-quality/](05-quality/) | 56 quy tắc kiểm soát chất lượng | Phân tích dữ liệu, Kỹ sư dữ liệu |
| [06-platform/](06-platform/) | Nguồn dữ liệu, thu nạp, hồ dữ liệu, kho, điều phối | Kỹ sư dữ liệu |
| [07-analytics/](07-analytics/) | Từ điển chỉ tiêu, báo cáo, mô hình dự báo | Phân tích dữ liệu |
| [08-operations/](08-operations/) | Công nghệ, bảo mật, quản trị, giám sát, mở rộng | Kiến trúc, Vận hành hệ thống |
| [09-roadmap/](09-roadmap/) | Lộ trình 9 giai đoạn (GĐ 0–8), 18 tuần | Quản lý dự án |
| [99-reference/](99-reference/) | Quy ước đặt tên, thuật ngữ, checklist | Chung |

## Thứ tự đọc khi bắt đầu triển khai

| # | Tài liệu | Việc |
|---|---|---|
| 1 | [../Flow.md](../Flow.md) | Hiểu toàn bộ luồng trong 15 phút |
| 2 | [00-business/nghiep-vu.md](00-business/nghiep-vu.md) | Hiểu nghiệp vụ sinh ra dữ liệu |
| 3 | [01-erd/](01-erd/) | Hiểu mô hình dữ liệu và độ hạt từng bảng |
| 4 | [02-mapping/source-to-target.md](02-mapping/source-to-target.md) | Biết từng cột lấy từ đâu, biến đổi ra sao |
| 5 | [03-ddl/00-init.md](03-ddl/00-init.md) → [06-ctl-qtn.md](03-ddl/06-ctl-qtn.md) | Dựng database theo đúng thứ tự |
| 6 | [04-etl/seed.md](04-etl/seed.md) | Nạp dữ liệu khởi tạo **trước** khi chạy pipeline |
| 7 | [05-quality/dq-rules.md](05-quality/dq-rules.md) | Nạp catalog quy tắc, cấu hình cổng kiểm tra |
| 8 | [04-etl/load-dimension.md](04-etl/load-dimension.md) → [load-fact.md](04-etl/load-fact.md) | Viết quy trình nạp |

## Danh mục tài liệu

| File | Nội dung |
|---|---|
| [00-business/nghiep-vu.md](00-business/nghiep-vu.md) | Hành trình khách hàng, miền nghiệp vụ, quy trình, sự kiện và thuộc tính bắt buộc |
| [01-erd/erd-logic.md](01-erd/erd-logic.md) | Thực thể master và transaction, ba loại khoá, ERD, cardinality, xử lý quan hệ nhiều-nhiều |
| [01-erd/grain.md](01-erd/grain.md) | Khai báo độ hạt toàn bộ bảng, double counting, fan-out, additivity |
| [01-erd/star-schema.md](01-erd/star-schema.md) | Fact và dim, ba loại Fact, SCD, dim đặc biệt, chuẩn hoá và phi chuẩn hoá |
| [01-erd/bus-matrix.md](01-erd/bus-matrix.md) | Ma trận Fact × dim, conformed dimension, drilling across |
| [02-mapping/source-to-target.md](02-mapping/source-to-target.md) | Mapping mức cột: 15 mục, phủ 19 bảng `crt`, mapping `crt → dm`, ánh xạ danh mục, cột tính trong kho |
| [03-ddl/00-init.md](03-ddl/00-init.md) | Tạo database và schema, chuẩn kiểu dữ liệu, collation, chính sách NULL, khoá và ràng buộc, index và phân vùng, dự toán dung lượng |
| [03-ddl/01-lnd.md](03-ddl/01-lnd.md) | 28 bảng vùng đệm, script sinh DDL tự động |
| [03-ddl/02-crt.md](03-ddl/02-crt.md) | 25 bảng + 1 view tầng đối soát, thứ tự tạo bảng |
| [03-ddl/03-dm-dimension.md](03-ddl/03-dm-dimension.md) | 13 dim: `dim_date`, `dim_time`, 4 dim SCD2, 6 dim SCD1, junk dimension |
| [03-ddl/04-dm-fact.md](03-ddl/04-dm-fact.md) | 10 Fact đủ ba loại |
| [03-ddl/05-svg-bi.md](03-ddl/05-svg-bi.md) | 6 bảng tổng hợp phục vụ báo cáo + bảng cầu nối `dm.bridge_sales_promotion` |
| [03-ddl/06-ctl-qtn.md](03-ddl/06-ctl-qtn.md) | 9 bảng + 1 view điều khiển và cách ly |
| [04-etl/seed.md](04-etl/seed.md) | 16 script khởi tạo, kịch bản chạy toàn bộ |
| [04-etl/load-dimension.md](04-etl/load-dimension.md) | Nạp 13 dim: khuôn SCD2 bốn bước, khuôn SCD1, thuộc tính phái sinh |
| [04-etl/load-fact.md](04-etl/load-fact.md) | Nạp 10 Fact, accumulating snapshot, periodic snapshot, 6 bảng tổng hợp |
| [05-quality/dq-rules.md](05-quality/dq-rules.md) | 56 quy tắc trên 7 nhóm, kèm SQL kiểm tra và kịch bản chạy |
| [06-platform/nguon-va-thu-nap.md](06-platform/nguon-va-thu-nap.md) | 4 nhóm nguồn, 3 cơ chế thu nạp, Kafka và Schema Registry |
| [06-platform/ho-du-lieu-va-kho.md](06-platform/ho-du-lieu-va-kho.md) | Phân vùng hồ dữ liệu, Iceberg, nạp và kiểm soát, 4 tầng kho, cổng chất lượng, Airflow |
| [07-analytics/chi-tieu-va-bao-cao.md](07-analytics/chi-tieu-va-bao-cao.md) | Từ điển chỉ tiêu 4 nhóm, bộ báo cáo, 6 bài toán dự báo |
| [08-operations/van-hanh.md](08-operations/van-hanh.md) | Lựa chọn công nghệ, bảo mật, chất lượng dữ liệu, quản trị, giám sát, mở rộng và phục hồi |
| [09-roadmap/lo-trinh.md](09-roadmap/lo-trinh.md) | 9 giai đoạn (GĐ 0–8), việc phải làm sớm, mốc kiểm soát |
| [99-reference/tra-cuu.md](99-reference/tra-cuu.md) | Quy ước đặt tên, thuật ngữ, 4 checklist, bản đồ nghiệp vụ đến bảng |

## Tình trạng hoàn thiện

| Hạng mục | Tiến độ |
|---|---|
| `lnd` — 28 bảng | Xong |
| `crt` — 25 bảng + 1 view | Xong |
| `dm` — 13 dim + 10 Fact + 1 cầu nối | Xong, đủ khoá ngoại |
| `svg_bi` — 6 bảng | Xong |
| `ctl` + `qtn` — 9 bảng + 1 view | Xong |
| Ánh xạ nguồn sang đích | Xong |
| Script khởi tạo | Xong |
| Catalog quy tắc chất lượng — 56 quy tắc | Xong |
| Quy trình nạp — 8 procedure mẫu, đủ khuôn cho 23 bảng | Xong |

### Hai bảng chưa thiết kế chi tiết

| Bảng | Vì sao | Thuộc giai đoạn |
|---|---|---|
| `fact_campaign_send` | Độ hạt phụ thuộc cách nền tảng marketing xuất dữ liệu, chưa kiểm kê xong | 7 |
| `fact_service_view` | Khối lượng lớn (~2,5 triệu dòng/năm) và chỉ dùng ở mức tổng hợp — có thể giữ nguyên ở Iceberg, không nạp vào SQL Server. Cần đo trước khi quyết | 7 |

## Năm dữ liệu chưa có nguồn

Các chỉ số tương ứng sẽ không tính được nếu không được cung cấp:

| Cần | Chặn | Bên cung cấp |
|---|---|---|
| Lịch làm việc / phân ca kỹ thuật viên | Năng suất kỹ thuật viên, tỷ lệ lấp buồng | Vận hành |
| Giờ mở cửa từng salon theo ngày | Tỷ lệ lấp buồng | Vận hành |
| Cách tính giá vốn dịch vụ | Lợi nhuận gộp, giá trị vòng đời khách hàng | Kế toán |
| Bảng số liệu đối chiếu doanh thu từ POS | `DQ-RECON-001` — tiêu chí nghiệm thu số 1 | Nhà cung cấp POS |
| Tỷ giá quy đổi điểm thưởng | Giá trị điểm thưởng | CRM |
