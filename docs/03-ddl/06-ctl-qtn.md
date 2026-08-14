# DDL — Schema `ctl` (Control) và `qtn` (Quarantine)

Hai schema không chứa dữ liệu nghiệp vụ. `ctl` chứa thông tin về **chính pipeline**; `qtn` chứa **dòng dữ liệu bị từ chối**.

Không có hai schema này thì khi được hỏi *"số liệu hôm nay đã đủ chưa"* sẽ không có câu trả lời nào ngoài phỏng đoán.

---

## 1. `ctl` — Bảng điều khiển

### `ctl.pipeline_run`

Grain: 1 dòng = 1 lần chạy 1 task.

```sql
CREATE TABLE ctl.pipeline_run (
    run_id        UNIQUEIDENTIFIER NOT NULL,
    dag_id        VARCHAR(100)   NOT NULL,
    task_id       VARCHAR(100)   NOT NULL,
    business_date DATE           NOT NULL,
    attempt_no    TINYINT        NOT NULL CONSTRAINT DF_ctl_run_attempt DEFAULT (1),
    started_at    DATETIME2(3)   NOT NULL CONSTRAINT DF_ctl_run_start   DEFAULT (SYSUTCDATETIME()),
    ended_at      DATETIME2(3)   NULL,
    duration_sec  AS DATEDIFF(SECOND, started_at, ended_at),
    status        VARCHAR(20)    NOT NULL,
    rows_read     BIGINT         NOT NULL CONSTRAINT DF_ctl_run_read   DEFAULT (0),
    rows_written  BIGINT         NOT NULL CONSTRAINT DF_ctl_run_write  DEFAULT (0),
    rows_rejected BIGINT         NOT NULL CONSTRAINT DF_ctl_run_reject DEFAULT (0),
    error_message NVARCHAR(MAX)  NULL,
    CONSTRAINT PK_ctl_pipeline_run PRIMARY KEY CLUSTERED (run_id),
    CONSTRAINT CK_ctl_run_status CHECK (status IN ('RUNNING','SUCCESS','FAILED','SKIPPED','BLOCKED')),
    -- Task đã kết thúc thì phải có mốc kết thúc
    CONSTRAINT CK_ctl_run_ended  CHECK (status = 'RUNNING' OR ended_at IS NOT NULL)
);

CREATE INDEX IX_ctl_run_bizdate ON ctl.pipeline_run (business_date, dag_id)
    INCLUDE (task_id, status, rows_written);
CREATE INDEX IX_ctl_run_status  ON ctl.pipeline_run (status, started_at)
    WHERE status IN ('RUNNING','FAILED','BLOCKED');
```

`duration_sec` là cột tính toán — không cần ETL ghi, và luôn khớp với hai mốc thời gian.

### `ctl.watermark`

Grain: 1 dòng = 1 cặp (nguồn, entity).

```sql
CREATE TABLE ctl.watermark (
    source_name    VARCHAR(50)  NOT NULL,
    entity_name    VARCHAR(100) NOT NULL,
    watermark_type VARCHAR(20)  NOT NULL,
    last_value     VARCHAR(100) NOT NULL,
    last_run_id    UNIQUEIDENTIFIER NULL,
    updated_at     DATETIME2(3) NOT NULL CONSTRAINT DF_ctl_wm_upd DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT PK_ctl_watermark PRIMARY KEY CLUSTERED (source_name, entity_name),
    CONSTRAINT CK_ctl_wm_type CHECK (watermark_type IN ('TIMESTAMP','LSN','PARTITION','FILE_LIST')),
    CONSTRAINT FK_ctl_wm_run  FOREIGN KEY (last_run_id) REFERENCES ctl.pipeline_run(run_id)
);
```

Giá trị khởi tạo cho luồng doanh thu:

```sql
INSERT INTO ctl.watermark (source_name, entity_name, watermark_type, last_value)
VALUES ('pos',  'invoice',       'PARTITION', '1900-01-01'),
       ('pos',  'invoice_line',  'PARTITION', '1900-01-01'),
       ('pos',  'payment',       'PARTITION', '1900-01-01'),
       ('pos',  'treatment',     'PARTITION', '1900-01-01'),
       ('pos',  'appointment',   'PARTITION', '1900-01-01'),
       ('oltp', 'booking',       'LSN',       '0'),
       ('oltp', 'booking_line',  'LSN',       '0'),
       ('oltp', 'customer',      'LSN',       '0'),
       ('oltp', 'point_ledger',  'LSN',       '0'),
       ('ads',  'insights',      'PARTITION', '1900-01-01'),
       ('app',  'service_viewed','TIMESTAMP', '1900-01-01T00:00:00'),
       ('app',  'feedback',      'TIMESTAMP', '1900-01-01T00:00:00');
```

### `ctl.load_audit`

Grain: 1 dòng = 1 file đã nạp. Đây là hàng rào chống nạp lại cùng một file hai lần.

```sql
CREATE TABLE ctl.load_audit (
    audit_id     BIGINT IDENTITY(1,1) NOT NULL,
    run_id       UNIQUEIDENTIFIER NOT NULL,
    source_name  VARCHAR(50)    NOT NULL,
    entity_name  VARCHAR(100)   NOT NULL,
    file_path    VARCHAR(1000)  COLLATE Latin1_General_100_BIN2 NOT NULL,
    file_hash    VARCHAR(64)    COLLATE Latin1_General_100_BIN2 NOT NULL,
    file_size_byte BIGINT       NOT NULL,
    rows_in_file BIGINT         NOT NULL,
    loaded_at    DATETIME2(3)   NOT NULL CONSTRAINT DF_ctl_audit_load DEFAULT (SYSUTCDATETIME()),
    status       VARCHAR(20)    NOT NULL,
    CONSTRAINT PK_ctl_load_audit PRIMARY KEY CLUSTERED (audit_id),
    -- Cùng nội dung file không được nạp hai lần
    CONSTRAINT UQ_ctl_audit_hash UNIQUE (source_name, entity_name, file_hash),
    CONSTRAINT CK_ctl_audit_status CHECK (status IN ('LOADED','ARCHIVED','FAILED')),
    CONSTRAINT FK_ctl_audit_run FOREIGN KEY (run_id) REFERENCES ctl.pipeline_run(run_id)
);

CREATE INDEX IX_ctl_audit_path ON ctl.load_audit (file_path);
```

> `UQ_ctl_audit_hash` dùng **hash nội dung**, không dùng đường dẫn. Nguồn đổi tên file rồi gửi lại cùng nội dung vẫn bị chặn. Đây là lớp phòng thứ hai, sau `UNIQUE` trên grain của bảng fact.

### `ctl.dq_result`

Grain: 1 dòng = 1 lần chạy 1 quy tắc.

```sql
CREATE TABLE ctl.dq_result (
    dq_result_id    BIGINT IDENTITY(1,1) NOT NULL,
    run_id          UNIQUEIDENTIFIER NOT NULL,
    rule_id         VARCHAR(50)    NOT NULL,
    entity_name     VARCHAR(100)   NOT NULL,
    dimension       VARCHAR(30)    NOT NULL,
    severity        VARCHAR(10)    NOT NULL,
    business_date   DATE           NOT NULL,
    metric_value    DECIMAL(18,4)  NULL,
    threshold_value DECIMAL(18,4)  NULL,
    failed_row_cnt  BIGINT         NOT NULL CONSTRAINT DF_ctl_dq_rows DEFAULT (0),
    status          VARCHAR(10)    NOT NULL,
    detail_message  NVARCHAR(1000) NULL,
    checked_at      DATETIME2(3)   NOT NULL CONSTRAINT DF_ctl_dq_at DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT PK_ctl_dq_result PRIMARY KEY CLUSTERED (dq_result_id),
    CONSTRAINT CK_ctl_dq_dimension CHECK (dimension IN
        ('Completeness','Accuracy','Consistency','Uniqueness','Validity','Freshness',
         'DimensionalModel')),
    CONSTRAINT CK_ctl_dq_severity CHECK (severity IN ('BLOCK','WARN','INFO')),
    CONSTRAINT CK_ctl_dq_status   CHECK (status   IN ('PASS','FAIL','ERROR')),
    CONSTRAINT FK_ctl_dq_run  FOREIGN KEY (run_id)  REFERENCES ctl.pipeline_run(run_id),
    CONSTRAINT FK_ctl_dq_rule FOREIGN KEY (rule_id) REFERENCES ctl.dq_rule(rule_id)
);

CREATE INDEX IX_ctl_dq_bizdate ON ctl.dq_result (business_date, status)
    INCLUDE (rule_id, entity_name, severity);
```

### `ctl.dq_rule`

Danh mục quy tắc. Tách khỏi `dq_result` để sửa ngưỡng không cần đổi code.

```sql
CREATE TABLE ctl.dq_rule (
    rule_id         VARCHAR(50)    NOT NULL,
    rule_name       NVARCHAR(200)  NOT NULL,
    entity_name     VARCHAR(100)   NOT NULL,
    dimension       VARCHAR(30)    NOT NULL,
    severity        VARCHAR(10)    NOT NULL,
    threshold_value DECIMAL(18,4)  NULL,
    check_sql       NVARCHAR(MAX)  NOT NULL,   -- câu SQL trả về số dòng vi phạm
    owner_role      VARCHAR(50)    NOT NULL,
    is_active       BIT            NOT NULL CONSTRAINT DF_ctl_rule_active DEFAULT (1),
    updated_at      DATETIME2(3)   NOT NULL CONSTRAINT DF_ctl_rule_upd DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT PK_ctl_dq_rule PRIMARY KEY CLUSTERED (rule_id),
    -- Phải có đủ 7 nhóm. Thiếu 'DimensionalModel' thì 8 quy tắc DQ-SCD-* và
    -- DQ-ALLOC-001 không insert được vào catalog, và cổng chất lượng mất nhóm
    -- thi hành nguyên tắc "báo cáo kỳ cũ in lại phải ra cùng số".
    CONSTRAINT CK_ctl_rule_dimension CHECK (dimension IN
        ('Completeness','Accuracy','Consistency','Uniqueness','Validity','Freshness',
         'DimensionalModel')),
    CONSTRAINT CK_ctl_rule_severity  CHECK (severity IN ('BLOCK','WARN','INFO'))
);
```

Nội dung khởi tạo: [Catalog DQ rule](../05-quality/dq-rules.md).

### `ctl.vn_holiday`

Danh mục ngày lễ Việt Nam. Nạp thủ công mỗi năm một lần vì Tết theo âm lịch, không có công thức.

```sql
CREATE TABLE ctl.vn_holiday (
    holiday_date DATE          NOT NULL,
    holiday_name NVARCHAR(50)  NOT NULL,
    is_tet       BIT           NOT NULL,
    is_observed  BIT           NOT NULL,   -- 1 = ngày nghỉ bù do Chính phủ công bố
    CONSTRAINT PK_ctl_vn_holiday PRIMARY KEY CLUSTERED (holiday_date)
);
```

### `ctl.code_mapping`

Bảng ánh xạ danh mục, thay cho việc hard-code trong ETL.

```sql
CREATE TABLE ctl.code_mapping (
    mapping_group VARCHAR(50)  NOT NULL,   -- acquisition_channel / region / booking_status / payment_method
    source_value  NVARCHAR(100) COLLATE Latin1_General_100_BIN2 NOT NULL,
    target_value  VARCHAR(50)  NOT NULL,
    updated_at    DATETIME2(3) NOT NULL CONSTRAINT DF_ctl_map_upd DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT PK_ctl_code_mapping PRIMARY KEY CLUSTERED (mapping_group, source_value)
);
```

Nội dung khởi tạo lấy từ [bảng ánh xạ danh mục trong STM](../02-mapping/source-to-target.md#ánh-xạ-danh-mục).

### `ctl.metric_definition`

Từ điển chỉ tiêu. Đây là bảng chốt định nghĩa để mọi báo cáo dùng cùng một công thức.

```sql
CREATE TABLE ctl.metric_definition (
    metric_code     VARCHAR(50)    NOT NULL,
    metric_name_vi  NVARCHAR(200)  NOT NULL,
    metric_group    VARCHAR(30)    NOT NULL,   -- finance / operation / customer / marketing
    formula_text    NVARCHAR(1000) NOT NULL,
    source_table    VARCHAR(100)   NOT NULL,
    grain_note      NVARCHAR(500)  NULL,       -- cảnh báo về DISTINCT, additivity
    period_required BIT            NOT NULL,   -- 1 = bắt buộc nêu kỳ tính
    approved_by     VARCHAR(100)   NULL,
    approved_at     DATE           NULL,
    CONSTRAINT PK_ctl_metric_definition PRIMARY KEY CLUSTERED (metric_code),
    CONSTRAINT CK_ctl_metric_group CHECK (metric_group IN ('finance','operation','customer','marketing'))
);
```

> `approved_by` để trống nghĩa là chỉ tiêu **chưa được nghiệp vụ ký xác nhận** — không được đưa lên báo cáo trình lãnh đạo.

---

## Thủ tục chạy cổng chất lượng — `ctl.usp_run_dq_group`

Thủ tục này là thứ mà mọi tác vụ cổng trong Airflow gọi tới. Nó đọc `check_sql` của từng quy tắc đang bật trong nhóm, chạy, ghi kết quả vào `ctl.dq_result`, và **báo lỗi ra ngoài nếu có quy tắc mức `BLOCK` thất bại** — nhờ đó Airflow đánh tác vụ là thất bại và nhánh phía sau không chạy.

```sql
CREATE OR ALTER PROCEDURE ctl.usp_run_dq_group
    @group         VARCHAR(30),          -- 'SCD', 'Completeness', 'Accuracy', ...
    @run_id        UNIQUEIDENTIFIER,
    @business_date DATE
AS
BEGIN
    SET NOCOUNT ON;   -- KHÔNG dùng XACT_ABORT: một quy tắc lỗi cú pháp không được
                      -- làm mất kết quả của các quy tắc đã chạy xong trước đó.

    DECLARE @rule_id VARCHAR(50), @sql NVARCHAR(MAX), @entity VARCHAR(100),
            @dim VARCHAR(30), @sev VARCHAR(10), @thr DECIMAL(18,4),
            @failed BIGINT, @status VARCHAR(10), @msg NVARCHAR(1000);

    DECLARE c CURSOR LOCAL FAST_FORWARD FOR
        SELECT rule_id, check_sql, entity_name, dimension, severity, threshold_value
        FROM   ctl.dq_rule
        WHERE  is_active = 1
          AND (dimension = @group OR rule_id LIKE 'DQ-' + @group + '-%');

    OPEN c;
    FETCH NEXT FROM c INTO @rule_id, @sql, @entity, @dim, @sev, @thr;

    WHILE @@FETCH_STATUS = 0
    BEGIN
        SET @failed = NULL; SET @msg = NULL;

        BEGIN TRY
            -- check_sql phải trả về đúng một cột, một dòng: SỐ DÒNG VI PHẠM
            DECLARE @out TABLE (failed_row_cnt BIGINT);
            DELETE FROM @out;
            INSERT INTO @out (failed_row_cnt)
            EXEC sys.sp_executesql @sql, N'@business_date DATE', @business_date;
            SELECT @failed = failed_row_cnt FROM @out;
            SET @status = CASE WHEN @failed <= ISNULL(@thr, 0) THEN 'PASS' ELSE 'FAIL' END;
        END TRY
        BEGIN CATCH
            -- Quy tắc chạy lỗi được ghi là ERROR, không phải PASS. Coi lỗi cú pháp
            -- là "đạt" chính là cách một cổng chất lượng âm thầm mất tác dụng.
            SET @status = 'ERROR';
            SET @failed = -1;
            SET @msg = LEFT(ERROR_MESSAGE(), 1000);
        END CATCH

        INSERT INTO ctl.dq_result
            (run_id, rule_id, entity_name, dimension, severity, business_date,
             metric_value, threshold_value, failed_row_cnt, status, detail_message)
        VALUES (@run_id, @rule_id, @entity, @dim, @sev, @business_date,
                @failed, @thr, ISNULL(@failed, 0), @status, @msg);

        FETCH NEXT FROM c INTO @rule_id, @sql, @entity, @dim, @sev, @thr;
    END

    CLOSE c; DEALLOCATE c;

    -- Cổng chặn: chỉ quy tắc mức BLOCK mới dừng nhánh. WARN và INFO ghi nhận rồi đi tiếp.
    -- ERROR cũng chặn, vì không biết dữ liệu đạt hay không thì không được cho đi tiếp.
    DECLARE @blocked INT = (
        SELECT COUNT(*) FROM ctl.dq_result
        WHERE run_id = @run_id AND business_date = @business_date
          AND severity = 'BLOCK' AND status IN ('FAIL','ERROR'));

    IF @blocked > 0
    BEGIN
        DECLARE @err NVARCHAR(400) =
            N'Cổng chất lượng nhóm ' + @group + N' thất bại: '
            + CAST(@blocked AS NVARCHAR(10)) + N' quy tắc mức BLOCK không đạt. '
            + N'Chi tiết trong ctl.dq_result theo run_id.';
        THROW 50001, @err, 1;
    END
END
```

**Ba quyết định trong thủ tục này:**

| Quyết định | Lý do |
|---|---|
| Không dùng `XACT_ABORT ON` | Một quy tắc lỗi cú pháp không được xoá kết quả của các quy tắc đã chạy xong. Người vận hành cần thấy toàn bộ bức tranh, không chỉ quy tắc lỗi đầu tiên |
| Quy tắc chạy lỗi ghi `ERROR` và **cũng chặn** | Coi lỗi cú pháp là "đạt" là cách một cổng chất lượng âm thầm mất tác dụng mà không ai biết |
| `THROW` thay vì trả mã lỗi | Airflow nhận biết tác vụ thất bại qua ngoại lệ; trả mã lỗi thì tác vụ vẫn xanh và nhánh sau vẫn chạy |

Ba quy tắc mức `BLOCK` của nhóm `SCD` được gọi ngay sau khi dựng dim: [04-etl/load-dimension.md](../04-etl/load-dimension.md).

---

## 2. `qtn` — Vùng cách ly

### `qtn.reject_row`

Grain: 1 dòng = 1 dòng dữ liệu bị từ chối bởi một quy tắc.

```sql
CREATE TABLE qtn.reject_row (
    reject_id      BIGINT IDENTITY(1,1) NOT NULL,
    run_id         UNIQUEIDENTIFIER NOT NULL,
    rule_id        VARCHAR(50)    NOT NULL,
    entity_name    VARCHAR(100)   NOT NULL,
    business_key   VARCHAR(200)   NULL,
    business_date  DATE           NULL,
    reject_reason  NVARCHAR(500)  NOT NULL,
    payload        NVARCHAR(MAX)  NOT NULL,   -- JSON của cả dòng gốc, để sửa và nạp lại
    src_file       VARCHAR(1000)  NULL,
    src_line_no    BIGINT         NULL,
    status         VARCHAR(20)    NOT NULL CONSTRAINT DF_qtn_status DEFAULT ('NEW'),
    assigned_to    VARCHAR(100)   NULL,
    resolution_note NVARCHAR(1000) NULL,
    created_at     DATETIME2(3)   NOT NULL CONSTRAINT DF_qtn_created DEFAULT (SYSUTCDATETIME()),
    resolved_at    DATETIME2(3)   NULL,
    CONSTRAINT PK_qtn_reject_row PRIMARY KEY CLUSTERED (reject_id),
    CONSTRAINT CK_qtn_status  CHECK (status IN ('NEW','INVESTIGATING','FIXED','IGNORED','REPROCESSED')),
    CONSTRAINT CK_qtn_payload CHECK (ISJSON(payload) = 1),
    -- Đã xử lý thì phải có mốc xử lý và ghi chú
    CONSTRAINT CK_qtn_resolved CHECK (
        status IN ('NEW','INVESTIGATING')
     OR (resolved_at IS NOT NULL AND resolution_note IS NOT NULL)),
    CONSTRAINT FK_qtn_run  FOREIGN KEY (run_id)  REFERENCES ctl.pipeline_run(run_id),
    CONSTRAINT FK_qtn_rule FOREIGN KEY (rule_id) REFERENCES ctl.dq_rule(rule_id)
);

CREATE INDEX IX_qtn_status ON qtn.reject_row (status, created_at)
    INCLUDE (entity_name, rule_id) WHERE status IN ('NEW','INVESTIGATING');
CREATE INDEX IX_qtn_entity ON qtn.reject_row (entity_name, business_date);
```

`src_file` + `src_line_no` được sao xuống đây từ `lnd` để người xử lý truy được về đúng dòng trong file gốc.

### View theo dõi vùng cách ly

```sql
-- Báo cáo hằng ngày gửi người phụ trách từng lĩnh vực
CREATE OR ALTER VIEW qtn.v_quarantine_summary AS
SELECT r.entity_name,
       r.rule_id,
       d.rule_name,
       d.owner_role,
       d.severity,
       COUNT(*)                                              AS reject_cnt,
       MIN(r.created_at)                                     AS oldest_at,
       DATEDIFF(DAY, MIN(r.created_at), SYSUTCDATETIME())    AS oldest_age_days,
       SUM(CASE WHEN r.status = 'NEW' THEN 1 ELSE 0 END)     AS new_cnt
FROM      qtn.reject_row r
JOIN      ctl.dq_rule    d ON d.rule_id = r.rule_id
WHERE     r.status IN ('NEW','INVESTIGATING')
GROUP BY  r.entity_name, r.rule_id, d.rule_name, d.owner_role, d.severity;
```

**Cam kết xử lý: 3 ngày làm việc.** `oldest_age_days > 3` sinh cảnh báo mức P2. Vùng cách ly không có người rà sẽ tích luỹ dòng lỗi mà không ai định lượng được phần doanh thu bị bỏ sót.

---

## 3. Thứ tự tạo và chính sách lưu

### Thứ tự tạo

```
1. ctl.pipeline_run
2. ctl.dq_rule
3. ctl.watermark, ctl.load_audit, ctl.dq_result   (phụ thuộc 1 và 2)
4. ctl.vn_holiday, ctl.code_mapping, ctl.metric_definition
5. qtn.reject_row                                  (phụ thuộc 1 và 2)
6. qtn.v_quarantine_summary
```

### Chính sách lưu

| Bảng | Giữ | Cách dọn |
|---|---|---|
| `ctl.pipeline_run` | 13 tháng | Xoá theo `business_date`, chạy hằng tháng |
| `ctl.load_audit` | **Không xoá** | Là hàng rào chống nạp trùng, xoá là mất tác dụng |
| `ctl.dq_result` | 13 tháng | Xoá theo `business_date` |
| `qtn.reject_row` | 13 tháng sau khi `status` khác `NEW`/`INVESTIGATING` | Không xoá dòng chưa xử lý |
| `ctl.watermark`, `ctl.dq_rule`, `ctl.code_mapping`, `ctl.metric_definition`, `ctl.vn_holiday` | **Không xoá** | Bảng cấu hình |

## Danh mục bảng

| # | Bảng | Grain | Dòng/năm (20 salon) |
|---|---|---|---|
| 1 | `ctl.pipeline_run` | 1 lần chạy 1 task | ~15.000 |
| 2 | `ctl.watermark` | 1 (nguồn, entity) | ~30 (không tăng) |
| 3 | `ctl.load_audit` | 1 file đã nạp | ~40.000 |
| 4 | `ctl.dq_result` | 1 lần chạy 1 rule | ~50.000 |
| 5 | `ctl.dq_rule` | 1 quy tắc | ~60 (không tăng) |
| 6 | `ctl.vn_holiday` | 1 ngày lễ | ~15 |
| 7 | `ctl.code_mapping` | 1 cặp ánh xạ | ~200 |
| 8 | `ctl.metric_definition` | 1 chỉ tiêu | ~30 |
| 9 | `qtn.reject_row` | 1 dòng bị từ chối | biến động |
| 10 | `qtn.v_quarantine_summary` | View | — |

**Tổng: 9 bảng + 1 view.**
