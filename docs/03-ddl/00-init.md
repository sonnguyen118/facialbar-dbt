# DDL — Khởi tạo database

Tạo database, schema, chuẩn kiểu dữ liệu, chiến lược khoá và ràng buộc, index và phân vùng, dự toán dung lượng.

Chạy **trước tiên**, trước mọi file DDL khác. Thứ tự đầy đủ: [04-etl/seed.md — kịch bản chạy toàn bộ](../04-etl/seed.md#kịch-bản-chạy-toàn-bộ).

## 1. Phạm vi thiết kế — cái gì ta thiết kế, cái gì là cho sẵn

Điều đầu tiên phải chốt trong mọi tài liệu thiết kế DB: **ranh giới trách nhiệm**.

| Đối tượng database | Ai thiết kế | Ta được làm gì |
|---|---|---|
| OLTP booking DB (PostgreSQL) | Team ứng dụng | **Chỉ đọc** qua CDC. Không đổi được schema → phải chịu đựng mọi thứ nguồn gửi sang |
| POS DB | Nhà cung cấp POS | Chỉ đọc qua export/CDC. **Rủi ro:** nhà cung cấp đổi schema không báo trước → cần DQ rule schema-drift |
| Kafka topic + Avro schema | **Ta thiết kế**, thống nhất với team app | Phần 3.3 |
| Iceberg table ở `cleansed` | **Ta thiết kế toàn bộ** | Phần 4.1 |
| `lnd`, `crt`, `dm`, `svg_bi`, `ctl`, `qtn` | **Ta thiết kế toàn bộ** | **Phần 5 này** |

> **`crt` chính là mô hình quan hệ chuẩn hoá 3NF của Facial Bar.** Nếu sau này công ty tự viết lại hệ thống booking thay cho POS mua ngoài, schema `crt` là điểm khởi đầu tốt nhất cho OLTP mới — nó đã được đối soát với thực tế nghiệp vụ trong nhiều tháng.

### Khởi tạo database và schema

```sql
CREATE DATABASE FacialBarDW
    COLLATE Vietnamese_CI_AI;          -- xem lý do ở mục 5.2
ALTER DATABASE FacialBarDW SET RECOVERY SIMPLE;   -- DWH dựng lại được từ Lake, không cần log chain đầy đủ
ALTER DATABASE FacialBarDW SET READ_COMMITTED_SNAPSHOT ON;  -- báo cáo không bị chặn bởi job nạp
GO

CREATE SCHEMA lnd;      -- Landing: vùng đệm, heap, ghi đè
CREATE SCHEMA crt;      -- Curated: 3NF, đã làm sạch, dùng để đối soát
CREATE SCHEMA dm;       -- Datamart: star schema
CREATE SCHEMA svg_bi;   -- Serving: bảng tổng hợp sẵn cho BI
CREATE SCHEMA ctl;      -- Control: run_id, watermark, DQ result
CREATE SCHEMA qtn;      -- Quarantine: dòng lỗi chờ xử lý
GO
```

`READ_COMMITTED_SNAPSHOT ON` là quyết định nhỏ nhưng quan trọng: không có nó, job nạp lúc 06:00 sẽ **chặn** dashboard của người mở lúc 06:05, và người dùng sẽ báo hệ thống không phản hồi.

---

## 2. Chuẩn kiểu dữ liệu và collation

**Nguyên tắc:** mỗi loại dữ liệu có **đúng một** kiểu chuẩn cho toàn bộ database. Không để cột `phone` chỗ thì `VARCHAR(15)`, chỗ thì `NVARCHAR(50)` — join giữa chúng sẽ sinh implicit conversion và mất index.

| Loại dữ liệu | Kiểu chuẩn | Vì sao chọn / vì sao không chọn cái khác |
|---|---|---|
| Surrogate key | `BIGINT IDENTITY(1,1)` (dim nhỏ dùng `INT`) | Join số nguyên nhanh nhất; `INT` đủ cho dim < 2,1 tỷ dòng |
| Business key (số) | `BIGINT` | — |
| Business key (chuỗi từ nguồn) | `VARCHAR(50)` | Mã POS/gateway đều ASCII, không cần Unicode |
| Khoá ngày | `INT` dạng `20260814` | Đọc được bằng mắt, dùng trực tiếp làm partition function. **Đánh đổi:** tốn 4 byte so với `DATE` 3 byte — chấp nhận |
| Khoá giờ | `SMALLINT` (0–1439 = phút trong ngày) | 2 byte, đủ biểu diễn từng phút |
| Ngày | `DATE` | 3 byte |
| Thời điểm | `DATETIME2(3)`, **luôn UTC** | 7 byte, chính xác tới ms. **Không dùng `DATETIME`**: 8 byte, độ phân giải 3,33 ms, giá trị bị làm tròn về `.000`/`.003`/`.007` giây — vừa tốn hơn 1 byte vừa kém chính xác hơn |
| Múi giờ | Không dùng `DATETIMEOFFSET` trong DWH | Đã chuẩn hoá UTC ở tầng cleansed; lưu offset lần nữa là mời gọi dữ liệu lệch múi giờ |
| Tiền | `DECIMAL(18,2)` | **Không dùng `MONEY`** (phép chia gây sai số làm tròn tích luỹ). **Không dùng `FLOAT`** (kế toán sẽ tìm ra chỗ lệch) |
| Tỷ lệ / hệ số | `DECIMAL(9,4)` hoặc `DECIMAL(9,6)` | Đủ chính xác cho hệ số phân bổ |
| Số lượng | `DECIMAL(9,2)` | Dịch vụ có thể bán nửa buổi; `INT` sẽ chặn nghiệp vụ này |
| Text tiếng Việt (tên, địa chỉ, ghi chú) | `NVARCHAR(n)` | Bắt buộc. `VARCHAR` không lưu được dấu tiếng Việt trên collation không phải UTF-8 |
| Mã kỹ thuật, status, email | `VARCHAR(n)` | ASCII thuần, tiết kiệm nửa dung lượng |
| Boolean | `BIT` | SQL Server đóng gói 8 cột `BIT` liền nhau vào 1 byte → nhóm chúng cạnh nhau khi khai báo |
| Danh mục / enum | `VARCHAR(20)` + `CHECK` + bảng tham chiếu | **Không** mã hoá thành `TINYINT`: đọc dữ liệu thô mà thấy `status = 3` thì không ai hiểu, và mọi câu SQL đều phải join thêm |
| UUID | `UNIQUEIDENTIFIER` | **Không dùng làm clustered key** — giá trị ngẫu nhiên gây phân mảnh trang nghiêm trọng |
| JSON | `NVARCHAR(MAX)` + `CHECK (ISJSON(col) = 1)` | Chỉ dùng ở `lnd` và `qtn`, không dùng ở `dm` |

### Collation — chi tiết đặc thù tiếng Việt, dễ bị bỏ sót

Chọn `Vietnamese_CI_AI` ở cấp database:
- **CI** (Case-Insensitive) — không phân biệt hoa/thường.
- **AI** (Accent-Insensitive) — không phân biệt dấu. Lễ tân gõ `nguyen thi lan` phải tìm ra `Nguyễn Thị Lan`. Đây là yêu cầu nghiệp vụ thật, không phải tuỳ chọn kỹ thuật.

**Hệ quả của `_AI`:** collation không phân biệt dấu coi `"Lan"` và `"Làn"` là **bằng nhau**. Nếu cột đó nằm trong `UNIQUE` hoặc dùng làm khoá join, hai giá trị khác nhau sẽ bị coi là trùng.

→ **Quy tắc:** mọi cột dùng làm **khoá hoặc mã** phải ghi đè collation nhị phân:

```sql
CREATE TABLE crt.customer (
    customer_id BIGINT        NOT NULL,
    phone       VARCHAR(20)   COLLATE Latin1_General_100_BIN2 NULL,  -- khoá gộp định danh: so sánh chính xác
    email       VARCHAR(255)  COLLATE Latin1_General_100_BIN2 NULL,  -- nt
    full_name   NVARCHAR(200) NULL,                                  -- theo collation DB: tìm kiếm không dấu
    ...
);
```

---

## 3. Cột kỹ thuật chuẩn và chính sách NULL

### Cột kỹ thuật bắt buộc theo từng tầng

Tiền tố `_` để phân biệt rõ với cột nghiệp vụ.

| Tầng | Cột kỹ thuật bắt buộc | Dùng để làm gì |
|---|---|---|
| `lnd` | `_src_file`, `_src_line_no`, `_run_id`, `_loaded_at` | Truy vết một dòng sai về đúng **file và dòng** trong file gốc |
| `crt` | `_src_system`, `_run_id`, `_loaded_at`, `_updated_at`, `_is_deleted` | `_is_deleted` = xoá mềm, giữ lịch sử khi CDC báo DELETE |
| `dm.dim_*` | `valid_from`, `valid_to`, `is_current`, `row_hash`, `_run_id`, `_updated_at` | Bộ điều khiển SCD2 |
| `dm.fact_*` | `_run_id`, `_loaded_at` | Biết dòng này do lần chạy nào nạp → xoá đúng khi nạp lại |
| `svg_bi.agg_*` | `_run_id`, `_refreshed_at` | Dashboard hiển thị "dữ liệu cập nhật lúc..." |

### Chính sách NULL — quyết định, không phải mặc định

| Vị trí | Chính sách | Vì sao |
|---|---|---|
| **Measure trong fact** | `NOT NULL DEFAULT 0` | `SUM(NULL)` trả NULL, và `NULL + 5 = NULL` → một dòng NULL làm cả tổng thành NULL |
| **FK trong fact** | `NOT NULL`, thiếu thì dùng `-1` (Unknown member) | NULL trong FK làm `INNER JOIN` **âm thầm xoá mất dòng doanh thu** |
| **Thuộc tính dim dùng để lọc** | `NOT NULL`, thiếu thì `N'(Không xác định)'` | NULL hiển thị thành ô trống trên BI, người dùng tưởng hệ thống lỗi |
| **Thuộc tính dim chỉ để xem** | Cho phép NULL | Ví dụ `holiday_name_vi` |
| **Ngày mốc chưa xảy ra** (accumulating snapshot) | `NULL` là **đúng** | `paid_date_key = NULL` nghĩa là "chưa trả tiền" — khác hoàn toàn với `-1` nghĩa là "không xác định" |

> ⚠️ Phân biệt ba trạng thái khác nhau, không được trộn: `NULL` = **chưa xảy ra**; `-1` = **đã xảy ra nhưng không biết là gì**; `0` = **giá trị bằng không**. Trộn ba cái này lại là nguồn của những báo cáo sai mà không ai giải thích được.

---

## 4. Chiến lược khoá và ràng buộc

### Ma trận ràng buộc theo tầng

| Ràng buộc | `lnd` | `crt` | `dm` | Quyết định & lý do |
|---|---|---|---|---|
| **PRIMARY KEY** | ❌ Không | ✅ Có | ✅ Có (trên SK) | `lnd` là heap ghi-đè, PK chỉ làm chậm việc nạp |
| **UNIQUE trên grain** | ❌ | ✅ | ✅ **Bắt buộc** | Đây là **hàng rào cứng duy nhất** chống double counting do nạp trùng |
| **FOREIGN KEY** | ❌ | ✅ Enforced | ⚠️ Có điều kiện — xem dưới | |
| **CHECK** | ❌ | ✅ | ✅ | Chặn dữ liệu vô lý ngay tại database |
| **DEFAULT** | ❌ | ✅ | ✅ | |
| **NOT NULL** | ❌ (để rộng) | ✅ | ✅ | `lnd` phải **không bao giờ fail lúc nạp**; sai kiểu để tầng sau bắt với thông báo rõ ràng |

### Quyết định về FOREIGN KEY ở tầng `dm` — có đánh đổi thật

Đây là điểm nhiều tài liệu thiết kế nói lấp lửng. Ba phương án:

| Phương án | Ưu | Nhược |
|---|---|---|
| **A. FK enforced đầy đủ** | Database tự đảm bảo không có fact mồ côi; Power BI tự nhận diện quan hệ khi import | Mỗi `INSERT` phải kiểm tra → nạp lô lớn chậm hơn rõ rệt |
| **B. Không tạo FK, dùng DQ rule** | Nạp nhanh nhất | Mất tài liệu hoá quan hệ trong chính database; phát hiện lỗi **sau khi** đã nạp |
| **C. FK có tạo nhưng `NOCHECK` trong lúc nạp** | Nhanh khi nạp, vẫn có tài liệu hoá | Bật lại `WITH CHECK` phải quét toàn bảng — với 200 triệu dòng là rất lâu |

**Quyết định cho Facial Bar: chọn A ở quy mô hiện tại (20 salon).** Volumetrics ở [mục 5.10](#6-volumetrics--dự-toán-số-dòng-và-dung-lượng) cho thấy fact lớn nhất chỉ ~421.000 dòng/năm — chi phí kiểm tra FK là không đáng kể so với lợi ích. **Chuyển sang B khi một fact vượt 100 triệu dòng**, và khi đó DQ rule "orphan check" phải được viết trước khi bỏ FK, không phải sau.

### Ràng buộc của SQL Server: UNIQUE index trên bảng đã phân vùng

Đây là chi tiết kỹ thuật rất dễ vướng khi triển khai thật.

SQL Server yêu cầu: **index unique trên bảng phân vùng phải chứa cột phân vùng** để được coi là *aligned*. Index không aligned thì **không thể `SWITCH` phân vùng** — mất luôn cơ chế xoá/lưu trữ dữ liệu cũ trong vài giây.

| Cách | Kết quả |
|---|---|
| `UNIQUE (invoice_line_id)` | ✅ Duy nhất toàn bảng — ❌ non-aligned, **chặn `SWITCH`** |
| `UNIQUE (service_date_key, invoice_line_id)` | ✅ Aligned, `SWITCH` được — ⚠️ chỉ đảm bảo duy nhất **trong cùng một ngày** |
| Không có unique index | ✅ Nạp nhanh nhất — ❌ mất hàng rào chống nạp trùng |

**Quyết định: chọn cách 2, bù bằng DQ rule kiểm tra duy nhất toàn cục.**
Lý do: một `invoice_line_id` về bản chất chỉ thuộc **một** `service_date_key`, nên trong thực tế cách 2 chặn được đúng tình huống hay gặp (nạp lại cùng một phân vùng hai lần). Trường hợp còn lại — cùng `invoice_line_id` xuất hiện dưới hai ngày khác nhau — là dấu hiệu **lỗi ở hệ nguồn**, và DQ rule sau đây sẽ bắt được:

```sql
-- DQ rule DQ-UNIQ-001 : duy nhất toàn cục của grain fact_sales_line
SELECT invoice_line_id, COUNT(*) AS dup_cnt
FROM   dm.fact_sales_line
GROUP  BY invoice_line_id
HAVING COUNT(*) > 1;
```

### Quy ước đặt tên ràng buộc và index

Bắt buộc đặt tên tường minh. Để SQL Server tự sinh tên (`PK__fact_sal__3213E83F8A4B...`) là làm cho thông báo lỗi trở nên vô nghĩa và làm script deploy không lặp lại được.

| Loại | Mẫu tên | Ví dụ |
|---|---|---|
| Primary key | `PK_<bảng>` | `PK_dim_customer` |
| Unique | `UQ_<bảng>_<cột>` | `UQ_dim_customer_bk_validfrom` |
| Unique index | `UX_<bảng>_<cột>` | `UX_dim_customer_current` |
| Foreign key | `FK_<bảng con>_<bảng cha>` | `FK_fact_sales_line_dim_customer` |
| Check | `CK_<bảng>_<quy tắc>` | `CK_fact_sales_line_amount_nonneg` |
| Default | `DF_<bảng>_<cột>` | `DF_fact_sales_line_quantity` |
| Index thường | `IX_<bảng>_<cột>` | `IX_crt_invoice_line_invoice_id` |
| Columnstore | `CCI_<bảng>` | `CCI_fact_sales_line` |

---

---

## 5. Index và Partition

### Partition function và scheme

```sql
-- Sinh danh sách biên phân vùng theo tháng cho 2024–2032
DECLARE @vals NVARCHAR(MAX) = N'', @d DATE = '2024-01-01';
WHILE @d <= '2032-12-01'
BEGIN
    SET @vals += CONVERT(VARCHAR(8), @d, 112) + N',';
    SET @d = DATEADD(MONTH, 1, @d);
END

DECLARE @sql NVARCHAR(MAX) =
    N'CREATE PARTITION FUNCTION pf_date_key_month (INT) AS RANGE RIGHT FOR VALUES ('
    + LEFT(@vals, LEN(@vals) - 1) + N');';
EXEC sys.sp_executesql @sql;

CREATE PARTITION SCHEME ps_date_key_month
    AS PARTITION pf_date_key_month ALL TO ([PRIMARY]);
```

Dùng `RANGE RIGHT` để biên `20260801` thuộc **phân vùng tháng 8**, đúng trực giác. `RANGE LEFT` sẽ đẩy ngày mùng 1 về phân vùng tháng trước — nguồn của những sai lệch 1 ngày rất khó tìm.

### Ma trận index theo loại bảng

| Loại bảng | Index chính | Index phụ | Lý do |
|---|---|---|---|
| `lnd.*` | **Không có** (heap) | — | Chỉ ghi một lần rồi đọc một lần; index chỉ làm chậm nạp |
| `crt.*` | CLUSTERED trên business key | NC trên FK + cột hay lọc; NC trên `(_run_id)` | Phục vụ đối soát và join khi build `dm` |
| `dm.dim_*` nhỏ (<100k) | CLUSTERED PK trên SK (rowstore) | UNIQUE `(bk, valid_from)`; filtered UNIQUE `WHERE is_current=1`; NC `(bk, valid_from, valid_to)` | Join theo SK; temporal join khi nạp fact |
| `dm.fact_*` lớn | **CLUSTERED COLUMNSTORE**, aligned theo phân vùng | UNIQUE NC trên `(date_key, grain_id)` | Nén 5–10×, quét nhanh; NC chặn nạp trùng |
| `dm.fact_*` nhỏ (<100k) | CLUSTERED rowstore trên PK | — | Dưới 102.400 dòng/rowgroup, columnstore **chậm hơn** rowstore |
| `dm.fact_booking_lifecycle` | CLUSTERED rowstore trên `booking_id` | NC trên `booked_date_key` INCLUDE cờ phễu | Bị UPDATE liên tục → columnstore không phù hợp |
| `svg_bi.agg_*` | CLUSTERED rowstore trên `(date_key, dim_sk)` | — | Bảng nhỏ, truy vấn theo khoảng ngày |

**Nguyên tắc: fact chỉ có tối đa 1–2 index phụ.** Mỗi index phụ trên fact làm chậm việc nạp và tăng dung lượng. Nếu dashboard chậm, giải pháp đúng là **thêm bảng tổng hợp ở `svg_bi`**, không phải thêm index vào fact.

### Sliding window — lưu trữ dữ liệu cũ trong vài giây

Khi cần đưa dữ liệu quá 25 tháng ra khỏi DWH (bước 3 ở mục 7.6):

```sql
-- 1. Bảng staging phải cùng cấu trúc, cùng filegroup, cùng loại index
CREATE TABLE dm.fact_sales_line_switchout (/* ...cấu trúc y hệt... */)
    ON ps_date_key_month (service_date_key);

-- 2. Chuyển cả phân vùng ra ngoài: chỉ là đổi metadata, KHÔNG di chuyển dữ liệu
ALTER TABLE dm.fact_sales_line
    SWITCH PARTITION 14 TO dm.fact_sales_line_switchout PARTITION 14;

-- 3. Dữ liệu đã có bản gốc bất biến ở S3 → xoá bảng staging là an toàn
DROP TABLE dm.fact_sales_line_switchout;
```

`SWITCH` chạy trong vài giây bất kể phân vùng có bao nhiêu dòng, vì nó chỉ đổi con trỏ metadata. So sánh: `DELETE FROM ... WHERE service_date_key < 20240101` trên 40 triệu dòng sẽ chạy hàng chục phút, làm transaction log tăng mạnh và chặn mọi truy vấn khác.

> Đây là ví dụ rõ nhất cho thấy **`raw` zone bất biến trên S3 là điều kiện tiên quyết** để `SWITCH` rồi `DROP` mà không lo mất dữ liệu. Hai quyết định thiết kế ở hai tầng khác nhau nhưng phụ thuộc lẫn nhau.

---

---

## 6. Volumetrics — dự toán số dòng và dung lượng

không có volumetrics thì mọi quyết định về index, partition và chọn DBMS đều là phỏng đoán. Đây cũng là bước kiểm chứng lại lựa chọn công nghệ ở [mục 7.1](../08-operations/van-hanh.md#1-lựa-chọn-công-nghệ).

### Giả định nghiệp vụ

| Tham số | Giá trị | Nguồn giả định |
|---|---|---|
| Số salon (hiện tại) | 20 | Thực tế |
| Lượt treatment / salon / ngày | 45 | 10 buồng × 6 slot × ~75% lấp buồng |
| Ngày hoạt động / năm | 350 | Nghỉ Tết |
| Dịch vụ / lần đến | 1,25 | Tỷ lệ up-sell hiện tại |
| Tỷ lệ no-show | 12% | Thực tế ngành |
| Tỷ lệ hoá đơn có bán lẻ sản phẩm | 30%, 1,4 dòng | Thực tế ngành |
| Tỷ lệ gửi feedback | 35% | Thực tế ngành |

### Dự toán số dòng mỗi năm

| Bảng | Công thức | 20 salon | 2.000 salon (×100) |
|---|---|---|---|
| `fact_treatment` | 20 × 45 × 350 | **315.000** | 31,5 tr |
| `fact_appointment` | treatment ÷ 1,25 ÷ 0,88 | **286.000** | 28,6 tr |
| `fact_booking_line` | appointment × 1,3 | **372.000** | 37,2 tr |
| `fact_sales_line` | 315.000 + (0,30 × 252.000 × 1,4) | **421.000** | 42,1 tr |
| `fact_payment` | 252.000 × 1,15 | **290.000** | 29,0 tr |
| `fact_loyalty_txn` | 252.000 × 2 | **504.000** | 50,4 tr |
| `fact_feedback` | 315.000 × 0,35 | **110.000** | 11,0 tr |
| `fact_booking_lifecycle` | = số booking | **286.000** | 28,6 tr |
| `fact_customer_monthly_snapshot` | khách active × 12 | **~420.000** | 42,0 tr |
| `fact_ad_spend` | 350 × 30 campaign × 2 platform | **21.000** | 21.000 |
| *App event (chỉ ở Lake, không vào DWH)* | — | *~2,5 tr* | *250 tr* |

### Dự toán dung lượng `dm` (fact lớn nhất)

| | 20 salon | 2.000 salon |
|---|---|---|
| `fact_sales_line` — dòng/năm | 421.000 | 42,1 triệu |
| Byte/dòng (rowstore, ~140 B) | 59 MB/năm | 5,9 GB/năm |
| Byte/dòng (**columnstore**, nén ~8×) | **~7 MB/năm** | **~740 MB/năm** |
| Sau 5 năm | ~35 MB | ~3,7 GB |
| **Toàn bộ schema `dm` sau 5 năm** (mọi fact + dim) | **~150 MB** | **~15 GB** |

### Ba kết luận thiết kế từ volumetrics

**1. Ở quy mô 20 salon, dung lượng không phải vấn đề gì cả.** Toàn bộ datamart nằm gọn trong RAM của một server tầm trung. Điều này **xác nhận** quyết định giữ FK enforced ở mục 5.4 và quyết định chọn SQL Server ở mục 7.1.

**2. Ngay ở quy mô 2.000 salon, `dm` cũng chỉ ~15 GB.** Fact lớn nhất đạt ~210 triệu dòng sau 5 năm — vẫn dưới ngưỡng 1 tỷ dòng ở bước 3 của mục 7.6. Nghĩa là **bước 4 (di trú sang MPP) rất có thể không bao giờ cần đến**. Đường thoát vẫn phải giữ, nhưng không nên đầu tư trước cho nó.

**3. Bảng lớn nhất của toàn hệ thống là app event (250 triệu dòng/năm ở quy mô 2.000 salon) — và nó không nằm trong DWH.** Nó nằm ở S3/Iceberg, nơi lưu trữ rẻ và tính toán tách rời. Đây chính là lý do kiến trúc **Lake + Warehouse** thay vì chỉ một trong hai: dữ liệu hành vi khối lượng lớn ở Lake, dữ liệu giao dịch cần join nhanh ở Warehouse.

> **Điểm nghẽn thật sự không phải dung lượng mà là THỜI GIAN NẠP.** Ở quy mô 2.000 salon, mỗi đêm phải nạp ~1,2 triệu dòng fact trong cửa sổ 05:00–06:40. Đó là lý do các quyết định về idempotent, phân vùng và `TRUNCATE PARTITION` quan trọng hơn nhiều so với việc tiết kiệm vài GB.

---
