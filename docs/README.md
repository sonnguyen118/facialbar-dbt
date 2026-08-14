# Tài liệu thiết kế — Facial Bar Data Platform

Chỉ mục điều hướng. Tài liệu luồng tổng thể ở [../Flow.md](../Flow.md); bản trình phê duyệt ở [../Ban-Thiet-Ke-CSDL.md](../Ban-Thiet-Ke-CSDL.md).

## Cấu trúc

| Thư mục | Nội dung | Đối tượng đọc |
|---|---|---|
| [01-erd/](01-erd/) | ERD nghiệp vụ, star schema, bảng khai báo grain, Bus Matrix | Data Analyst |
| [02-mapping/](02-mapping/) | Source-to-Target Mapping ở mức cột | Data Engineer, Data Analyst |
| [03-ddl/](03-ddl/) | DDL đầy đủ theo từng schema | DBA, Data Engineer |
| [04-etl/](04-etl/) | Quy trình nạp, script seed | Data Engineer |
| [05-quality/](05-quality/) | Catalog quy tắc kiểm soát chất lượng | Data Analyst, Data Engineer |

## Thứ tự đọc khi bắt đầu triển khai

| # | Tài liệu | Việc |
|---|---|---|
| 1 | [../Flow.md](../Flow.md) Phần 0–2 | Hiểu kiến trúc, nghiệp vụ và mô hình logic |
| 2 | [02-mapping/source-to-target.md](02-mapping/source-to-target.md) | Biết dữ liệu từng cột lấy từ đâu, biến đổi ra sao |
| 3 | [03-ddl/](03-ddl/) | Dựng database theo đúng thứ tự |
| 4 | [04-etl/seed.md](04-etl/seed.md) | Nạp dữ liệu khởi tạo **trước** khi chạy pipeline |
| 5 | [05-quality/dq-rules.md](05-quality/dq-rules.md) | Nạp catalog quy tắc, cấu hình cổng kiểm tra |

## Danh mục tài liệu

### 01-erd
| File | Nội dung | Trạng thái |
|---|---|---|
| `erd-logic.md` | ERD nghiệp vụ, cardinality | ⏳ Đang ở [Flow.md 2.2](../Flow.md#22-relationship--cardinality) |
| `grain.md` | Bảng khai báo grain toàn bộ bảng | ⏳ Đang ở [Flow.md 2.3](../Flow.md#23-grain--độ-hạt-của-bảng) |
| `star-schema.md` | Sơ đồ star schema, SCD, dimension đặc biệt | ⏳ Đang ở [Flow.md 2.4](../Flow.md#24-star-schema--mô-hình-chiều) |
| `bus-matrix.md` | Ma trận fact × dimension | ⏳ Đang ở [Flow.md 2.7](../Flow.md#27-bus-matrix) |

### 02-mapping
| File | Nội dung | Trạng thái |
|---|---|---|
| [source-to-target.md](02-mapping/source-to-target.md) | Mapping mức cột cho 15 bảng `crt`, mapping `crt → dm`, bảng ánh xạ danh mục, cột tính trong kho, phụ thuộc chưa có | ✅ |

### 03-ddl
| File | Nội dung | Trạng thái |
|---|---|---|
| `00-init.md` | Tạo database, schema, chuẩn kiểu dữ liệu, collation, partition scheme | ⏳ Đang ở [Flow.md 5.1–5.2, 5.8](../Flow.md#51-phạm-vi-thiết-kế--cái-gì-ta-thiết-kế-cái-gì-là-cho-sẵn) |
| [01-lnd.md](03-ddl/01-lnd.md) | 28 bảng landing, script sinh DDL tự động | ✅ |
| [02-crt.md](03-ddl/02-crt.md) | 25 bảng + 1 view, đầy đủ khoá/FK/CHECK/index, thứ tự tạo | ✅ |
| `03-dm-dimension.md` | 13 dimension | ⏳ Đang ở [Flow.md 5.5](../Flow.md#55-ddl--dimension) |
| `04-dm-fact.md` | 10 fact + 1 bridge | ⏳ Đang ở [Flow.md 5.6](../Flow.md#56-ddl--fact) — **còn thiếu FK cho 7 fact** |
| `05-svg-bi.md` | 6 bảng tổng hợp | ⏳ 2 bảng ở [Flow.md 5.7](../Flow.md#57-ddl--bridge-table-và-aggregate-table), **4 bảng chưa thiết kế** |
| [06-ctl-qtn.md](03-ddl/06-ctl-qtn.md) | 9 bảng + 1 view điều khiển và cách ly | ✅ |

### 04-etl
| File | Nội dung | Trạng thái |
|---|---|---|
| [seed.md](04-etl/seed.md) | Unknown member 12 dim, `dim_date`, `dim_time` 1.440 dòng, `dim_booking_junk` 80 tổ hợp, danh mục cố định, cấu hình `ctl`, ngày lễ | ✅ |
| `load-dimension.md` | Quy trình nạp 13 dimension | ⏳ 1/13 ở [Flow.md 5.9](../Flow.md#59-thủ-tục-nạp--scd2-và-fact) |
| `load-fact.md` | Quy trình nạp 10 fact + accumulating snapshot | ⏳ Mẫu ở [Flow.md 4.2](../Flow.md#42-ingestion--loading-layer--nạp-và-kiểm-soát) |

### 05-quality
| File | Nội dung | Trạng thái |
|---|---|---|
| [dq-rules.md](05-quality/dq-rules.md) | 56 quy tắc (44 BLOCK, 11 WARN, 1 INFO) trên 7 nhóm, kèm SQL kiểm tra | ✅ |

## Tình trạng hoàn thiện

| Hạng mục | Tiến độ |
|---|---|
| `lnd` — 28 bảng | ✅ Xong |
| `crt` — 25 bảng + 1 view | ✅ Xong |
| `ctl` + `qtn` — 9 bảng + 1 view | ✅ Xong |
| `dm` — 26 bảng | ⚠️ Có DDL nhưng thiếu FK cho 7 fact |
| `svg_bi` — 6 bảng | ⚠️ 2/6 |
| Source-to-Target Mapping | ✅ Xong |
| Script seed | ✅ Xong |
| Catalog DQ rule | ✅ Xong |
| Quy trình nạp | ⚠️ 1/23 |
| Tách ERD ra file riêng | ⏳ Chưa |

## Phụ thuộc bên ngoài chưa có

Năm dữ liệu sau chưa có nguồn và sẽ chặn các chỉ số tương ứng:

| Cần | Chặn | Bên cung cấp |
|---|---|---|
| Lịch làm việc / phân ca KTV | Therapist Utilization, Bed Occupancy | Vận hành |
| Giờ mở cửa từng salon theo ngày | Bed Occupancy | Vận hành |
| Quy tắc tính COGS dịch vụ | Gross Margin, CLV | Kế toán |
| Bảng số liệu đối chiếu doanh thu từ POS | `DQ-RECON-001` — tiêu chí nghiệm thu số 1 | Nhà cung cấp POS |
| Tỷ giá quy đổi điểm thưởng | `point_value_amount` | CRM |
