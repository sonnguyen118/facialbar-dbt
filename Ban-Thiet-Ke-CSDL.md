# Bản trình phê duyệt — Hệ thống dữ liệu chuỗi Facial Bar

**Trình:** Ban Tổng Giám đốc · **Đơn vị soạn:** Bộ phận Dữ liệu · **Ngày:** …

Tài liệu này chỉ chứa những nội dung cần Ban Tổng Giám đốc quyết. Đặc tả kỹ thuật đầy đủ ở [README.md](README.md).

---

## 1. Tóm tắt

**Đề xuất:** xây một hệ thống dữ liệu tập trung, nơi mọi báo cáo của chuỗi lấy số từ một nguồn duy nhất và số đó đã được đối soát tự động với POS mỗi ngày.

**Vấn đề đang có**

| Hiện trạng | Hệ quả kinh doanh |
|---|---|
| Mỗi bộ phận tự lấy số từ POS, ứng dụng, bảng tính riêng | Cùng câu hỏi "doanh thu tháng trước" ra ba con số khác nhau; họp giao ban mất thời gian đối chiếu thay vì ra quyết định |
| Không có định nghĩa chung cho "doanh thu", "khách quay lại" | Không so sánh được kết quả giữa các chi nhánh |
| Báo cáo làm tay | Phụ thuộc cá nhân: người phụ trách nghỉ thì báo cáo không được cập nhật |
| Không đối soát tự động với POS | Sai lệch chỉ phát hiện khi kế toán khoá sổ, thường sau vài tuần |

**Giá trị mang lại**

| Nhóm | Kết quả cụ thể |
|---|---|
| Ban điều hành | Doanh thu thuần, tiền thực thu, xếp hạng chi nhánh — cập nhật trước 08:00 mỗi ngày, đã đối soát POS |
| Tài chính | Đối soát tự động hằng ngày giữa hệ thống, POS và cổng thanh toán; lệch quá 0,1% là cảnh báo ngay |
| Vận hành | Tỷ lệ khách không đến, tỷ lệ huỷ lịch, thời gian phục vụ theo từng chi nhánh |
| Marketing | Chi phí thu hút một khách mới và doanh thu trên mỗi đồng quảng cáo, theo từng kênh |
| Chăm sóc khách hàng | Nhận diện sớm nhóm khách có nguy cơ không quay lại |

Bốn chỉ tiêu — lợi nhuận gộp, tỷ lệ lấp buồng, năng suất kỹ thuật viên, giá trị vòng đời khách hàng — **chưa tính được ngay** vì thiếu dữ liệu từ Kế toán và Vận hành. Chi tiết ở [mục 6](#6-rủi-ro-và-điều-kiện-tiên-quyết).

### 1.1. Quy mô đầu tư

| Hạng mục | Ước tính |
|---|---|
| Thời gian | **18 tuần** — 9 giai đoạn, mỗi giai đoạn 2 tuần |
| Nhân lực | **16,95 người-tháng** — chi tiết [mục 3.2](#32-nhân-lực) |
| Bản quyền phần mềm | **0 đồng phát sinh** — dùng SQL Server và Power BI đã có; phần còn lại là mã nguồn mở |
| Hạ tầng | Dung lượng 150 MB ở kho và 3 GB ở hồ dữ liệu cho quy mô 20 chi nhánh. Dự toán tiền: [mục 3.3](#33-dự-toán-chi-phí) |

**Phần lớn chi phí nằm ở nhân lực.** Đây là căn cứ khi cân nhắc phương án: cắt nhân sự để mua công nghệ đắt hơn không mang lại hiệu quả trong trường hợp này.

### 1.2. Ba rủi ro cần Ban Tổng Giám đốc can thiệp

Ba việc dưới đây bộ phận kỹ thuật không tự giải quyết được, và nếu không được xử lý thì tiến độ hoặc phạm vi bị ảnh hưởng trực tiếp.

| # | Rủi ro | Ảnh hưởng nếu không xử lý | Cần ai |
|---|---|---|---|
| 1 | Nhà cung cấp POS chưa cam kết xuất bảng số liệu đối chiếu doanh thu | **Tiêu chí nghiệm thu số 1 vô hiệu** — không chứng minh được số liệu đúng | Chỉ đạo đàm phán với nhà cung cấp POS |
| 2 | Kế toán chưa có cách tính giá vốn dịch vụ | Mất 2 trong 24 chỉ tiêu, gồm lợi nhuận gộp theo dịch vụ | Giao Kế toán chốt trước tuần 8 |
| 3 | Chưa có người chịu trách nhiệm dữ liệu cho từng lĩnh vực nghiệp vụ | Không ai ký định nghĩa chỉ tiêu, mốc M1 không đạt | Chỉ định tại [mục 4.3](#43-phân-công-trách-nhiệm-dữ-liệu) |

### 1.3. Đề nghị

1. **Phê duyệt** phương án thiết kế và kế hoạch 18 tuần với nguồn lực 16,95 người-tháng.
2. **Quyết định** 8 nội dung chính sách tại [mục 2](#2-tám-quyết-định-cần-phê-duyệt). Đây là các định nghĩa mang tính chuẩn công ty, bộ phận kỹ thuật không tự quyết được.
3. **Chỉ định** người chịu trách nhiệm dữ liệu cho từng lĩnh vực nghiệp vụ, và **chỉ đạo** ba việc ở mục 1.2.

---

## 2. Tám quyết định cần phê duyệt

Mỗi quyết định dưới đây ảnh hưởng tới cách tính số trên **mọi** báo cáo về sau. Đổi định nghĩa sau khi hệ thống chạy nghĩa là tính lại toàn bộ lịch sử.

### QĐ-01. Định nghĩa "doanh thu" của công ty

Bốn cách hiểu đang cùng tồn tại trong chuỗi:

| Cách hiểu | Nội dung | Bộ phận đang dùng | Khuyến nghị |
|---|---|---|---|
| Doanh thu gộp | Giá niêm yết, chưa trừ gì | Kinh doanh | |
| **Doanh thu thuần** | Giá niêm yết trừ mọi khoản giảm giá | Vận hành | ✔ |
| Doanh thu chưa thuế | Doanh thu thuần trừ thuế giá trị gia tăng | Kế toán | |
| Tiền thực thu | Tiền thực tế vào tài khoản trong kỳ | Tài chính | |

Hệ thống **tính và lưu cả bốn**, mỗi cái một tên cột riêng. Quyết định cần Ban Tổng Giám đốc: khi nói "doanh thu" trong báo cáo điều hành thì hiểu là cái nào. Không chốt thì mỗi báo cáo hiểu một kiểu, và chênh lệch giữa gộp và thuần ở chuỗi spa thường 10–20%.

**Cần phê duyệt:** Giám đốc Tài chính.

### QĐ-02. Ghi nhận doanh thu theo thời gian nào

| Phương án | Nội dung | Ảnh hưởng | Khuyến nghị |
|---|---|---|---|
| **A. Theo ngày thực hiện dịch vụ** | Ghi nhận khi dịch vụ được làm | Doanh thu phân bổ đúng cho chi nhánh phục vụ; cần theo dõi số dư gói trả trước | ✔ |
| B. Theo ngày thu tiền | Ghi nhận khi khách trả | Đơn giản hơn, nhưng khách mua gói 10 buổi thì toàn bộ doanh thu rơi vào tháng bán gói, các tháng sau chi nhánh phục vụ mà không ghi nhận đồng nào | |

**Cần phê duyệt:** Giám đốc Tài chính, kèm xác nhận tương thích với nguyên tắc ghi nhận trên sổ sách.

### QĐ-03. Quy gán doanh thu cho kênh marketing

Khách xem quảng cáo Facebook ngày 1, tìm Google ngày 3, đặt lịch ngày 7. Doanh thu tính cho kênh nào?

| Phương án | Cách quy gán | Hệ quả cho phân bổ ngân sách | Khuyến nghị |
|---|---|---|---|
| **A. Kênh có trả phí gần nhất trước khi đặt** | Toàn bộ về kênh tiếp xúc cuối | Ngân sách dồn về kênh chốt đơn. Rủi ro: cắt kênh nhận diện thương hiệu vì nó không bao giờ được ghi công | ✔ |
| B. Kênh tiếp xúc đầu tiên | Toàn bộ về kênh gặp đầu tiên | Ngân sách dồn về kênh nhận diện; đánh giá thấp kênh đang tạo đơn | |
| C. Chia đều cho mọi kênh có trả phí | Chia theo số điểm tiếp xúc | Cân bằng hơn, nhưng không kênh nào có tín hiệu đủ rõ để tăng hay giảm chi | |

Chênh lệch chi phí thu hút khách mới giữa A và B sẽ được đo trên dữ liệu 3 tháng đầu và trình lại tại mốc M1.

**Cần phê duyệt:** Giám đốc Marketing và Giám đốc Tài chính.

### QĐ-04. Nguồn chân lý cho từng loại dữ liệu

Cùng một sự việc được ghi ở nhiều hệ thống. Khi lệch nhau, tin hệ thống nào:

| Dữ liệu | Nguồn chân lý đề xuất | Vì sao |
|---|---|---|
| Doanh thu, hoá đơn | POS | Là chứng từ pháp lý |
| Lịch hẹn, buồng, kỹ thuật viên | Hệ thống đặt lịch | Là nơi thực tế xếp lịch |
| Thông tin khách | CRM | Là nơi được cập nhật thường xuyên nhất |
| Chi phí quảng cáo | Nền tảng quảng cáo | Là nơi phát sinh chi phí |

Không chốt thì mỗi lần lệch số lại phải họp để quyết tin bên nào.

**Cần phê duyệt:** Giám đốc Vận hành và Giám đốc Tài chính.

### QĐ-05. Thời gian lưu dữ liệu

| Tầng | Đề xuất | Hệ quả nếu chọn ngắn hơn |
|---|---|---|
| Dữ liệu thô ở hồ dữ liệu | 3 năm | Không dựng lại được lịch sử khi phát hiện lỗi tính toán cũ |
| Kho phân tích | 7 năm | Không phân tích được chu kỳ khách dài hơn 3 năm; giá trị vòng đời khách hàng mất ý nghĩa |
| Vết kiểm toán | 10 năm | Không đáp ứng yêu cầu tra soát chứng từ |

Dung lượng tiết kiệm được khi rút ngắn là không đáng kể so với năng lực phân tích mất đi — toàn bộ kho ở quy mô 2.000 chi nhánh chỉ khoảng 15 GB.

**Cần phê duyệt:** Giám đốc Tài chính và Pháp chế.

### QĐ-06. Dữ liệu nhạy cảm của khách hàng

| Loại dữ liệu | Cách xử lý đề xuất |
|---|---|
| Số điện thoại, email | Đưa vào kho ở dạng che một phần; bản đầy đủ chỉ ở tầng đối soát, phân quyền riêng |
| Ngày sinh | Chỉ đưa nhóm tuổi, không đưa ngày cụ thể |
| **Tình trạng da, ảnh trước và sau** | **Không đưa vào kho phân tích**; lưu ở khu vực riêng, chỉ kỹ thuật viên phụ trách xem được |

**Hệ quả của việc loại tình trạng da và ảnh:** không gợi ý được dịch vụ theo loại da, không đánh giá được hiệu quả điều trị theo tình trạng da. Dự báo khách rời bỏ và cá nhân hoá theo hành vi giao dịch **vẫn chạy được**. Nếu sau này cần phân tích theo tình trạng da, phương án thay thế là chỉ đưa vào kho **nhóm loại da đã mã hoá** (5 nhóm), không đưa mô tả tự do và ảnh.

**Cần phê duyệt:** Ban Tổng Giám đốc và Pháp chế.

### QĐ-07. Cam kết mức độ sẵn sàng của số liệu

**Đề xuất:** dữ liệu ngày N sẵn sàng trước **08:00 ngày N+1**, đạt tối thiểu **99% số ngày trong quý** — tương đương tối đa 1 ngày trễ mỗi quý.

Nếu yêu cầu mức cao hơn: sẵn sàng trước 06:00 cần dịch chuyển toàn bộ cửa sổ nạp sớm 2 giờ và bố trí người trực từ 03:00, phát sinh thêm khoảng 0,5 người-tháng mỗi tháng vận hành. Đạt 99,9% — tối đa 0,1 ngày lỗi mỗi quý thay vì 1 ngày — cần môi trường dự phòng nóng cho máy chủ.

**Cần phê duyệt:** Giám đốc Vận hành.

### QĐ-08. Nền tảng công nghệ cho kho dữ liệu

| Phương án | Ưu | Nhược | Chi phí | Khuyến nghị |
|---|---|---|---|---|
| **A. SQL Server đang có** | Tận dụng bản quyền và kỹ năng sẵn có; Power BI kết nối tự nhiên | Phải tự quản trị hạ tầng | 0 đồng bản quyền phát sinh | ✔ |
| B. Kho dữ liệu trên nền đám mây | Không phải quản trị hạ tầng, mở rộng tự động | Phụ thuộc nhà cung cấp; đội hiện tại chưa có kinh nghiệm | Chi phí theo mức sử dụng, phát sinh hằng tháng | |

Dự toán ở [mục 3.3](#33-dự-toán-chi-phí) cho thấy SQL Server đáp ứng được ngay cả ở quy mô 2.000 chi nhánh. Thiết kế vẫn giữ khả năng chuyển sang phương án B mà không phải viết lại toàn bộ.

**Cần phê duyệt:** Giám đốc Công nghệ Thông tin.

---

## 3. Lộ trình, nhân lực, chi phí

### 3.1. Chín giai đoạn — 18 tuần

Nguyên tắc: làm trọn vẹn **một luồng nghiệp vụ** từ nguồn đến báo cáo trước, thay vì làm dàn trải mọi nguồn cùng lúc. Cách này cho ra kết quả dùng được ngay ở tuần 8 và phát hiện sớm sai sót khi chi phí sửa còn thấp.

| Giai đoạn | Tuần | Nội dung | Điều kiện nghiệm thu |
|---|---|---|---|
| 0 | 1–2 | Khảo sát, kiểm kê nguồn dữ liệu | Xác định đủ nguồn và người phụ trách |
| 1 | 3–4 | Chốt mô hình nghiệp vụ và định nghĩa chỉ tiêu | **Các bộ phận ký xác nhận định nghĩa** |
| 2 | 5–6 | Dựng nền tảng lưu trữ và điều phối | Nạp thử một nguồn, chạy lại không sai số |
| 3 | 7–8 | **Luồng doanh thu hoàn chỉnh đầu tiên** | **Số khớp POS 7 ngày liên tiếp** |
| 4 | 9–10 | Kết nối dữ liệu thời gian thực | Sự kiện về hệ thống dưới 5 phút |
| 5 | 11–12 | Hoàn thiện mô hình phân tích | Phễu chuyển đổi khớp số vận hành ghi tay |
| 6 | 13–14 | Kiểm soát chất lượng và đối soát tự động | Chặn được lỗi trong kiểm thử có chủ đích |
| 7 | 15–16 | Phân tích khách hàng và marketing | Các chỉ số được nghiệp vụ chấp nhận |
| 8 | 17–18 | Vận hành và phân tích nâng cao | Có quy trình xử lý sự cố và phân ca trực |

**Bốn mốc Ban Tổng Giám đốc kiểm soát**

| Mốc | Thời điểm | Nội dung |
|---|---|---|
| M1 | Cuối tuần 4 | Trình duyệt định nghĩa chỉ tiêu — **cần chữ ký các bộ phận** |
| **M2** | **Cuối tuần 8** | **Trình diễn báo cáo doanh thu và kết quả đối soát POS** |
| M3 | Cuối tuần 14 | Trình diễn hệ thống kiểm soát chất lượng và cảnh báo |
| M4 | Cuối tuần 18 | Nghiệm thu tổng thể, bàn giao vận hành |

**M2 là mốc quyết định.** Nếu đến tuần 8 số chưa khớp POS, cần dừng lại rà soát chất lượng dữ liệu nguồn trước khi đầu tư tiếp.

### 3.2. Nhân lực

| Vai trò | Mức tham gia | Người-tháng |
|---|---|---|
| Kiến trúc sư dữ liệu | 50% | 2,25 |
| Kỹ sư dữ liệu | 2 người, toàn thời gian | 9,00 |
| Chuyên viên phân tích | Toàn thời gian từ giai đoạn 3 (tuần 7–18) | 3,00 |
| Chuyên viên nghiệp vụ | 30% | 1,35 |
| Quản trị hạ tầng | 30% | 1,35 |
| | **Tổng** | **16,95** |

Cơ sở tính: 18 tuần bằng 4,5 tháng. Riêng Chuyên viên phân tích tham gia 12 tuần, tức 3 tháng.

**Phụ thuộc từ bộ phận khác** — không bố trí thì chậm tiến độ: Kế toán xác nhận nguyên tắc doanh thu và giá vốn (giai đoạn 1); Vận hành cung cấp lịch phân ca và giờ mở cửa (giai đoạn 5); nhà cung cấp POS mở kết nối và cấp bảng đối chiếu (giai đoạn 3).

### 3.3. Dự toán chi phí

Bảng tách rõ phần đã xác định được từ thiết kế và phần cần đơn giá nội bộ. **Bộ phận kỹ thuật không tự điền đơn giá nhân sự và hạ tầng** — hai ô đó cần Tài chính và Nhân sự cung cấp trước khi trình ký.

| Hạng mục | Khối lượng đã xác định | Đơn giá | Thành tiền |
|---|---|---|---|
| Nhân lực triển khai, 18 tuần | 16,95 người-tháng | *Cần Nhân sự cung cấp theo từng vai trò* | *Chờ đơn giá* |
| Bản quyền SQL Server, Power BI | 0 — dùng giấy phép hiện có | 0 | **0** |
| Thành phần mã nguồn mở | 0 phí bản quyền | 0 | **0** |
| Lưu trữ hồ dữ liệu | 20 chi nhánh: ~3 GB sau 5 năm. 2.000 chi nhánh: ~300 GB | *Cần đơn giá theo hợp đồng đám mây hiện tại* | *Chờ đơn giá* |
| Máy chủ điều phối và luồng sự kiện | 1 máy 4 vCPU / 16 GB cho quy mô 20 chi nhánh | *Cần đơn giá hạ tầng nội bộ* | *Chờ đơn giá* |
| Vận hành thường xuyên sau bàn giao | 0,5 người-tháng mỗi tháng | *Cần Nhân sự cung cấp* | *Chờ đơn giá* |

**Ba kết luận không phụ thuộc đơn giá**

1. Bản quyền bằng 0; dung lượng ở quy mô 20 chi nhánh chỉ 150 MB trong kho và 3 GB ở hồ dữ liệu.
2. Gấp 100 lần số chi nhánh chỉ làm kho lên khoảng 15 GB — vẫn trong khả năng của một máy chủ SQL Server đơn. **Chi phí hạ tầng tăng theo khối lượng dữ liệu, không theo số chi nhánh.**
3. Chi phí vận hành sau bàn giao thấp hơn chi phí triển khai khoảng 7 lần mỗi tháng: 0,5 so với 3,77 người-tháng mỗi tháng trong lúc triển khai.

---

## 4. Bảo mật, quản trị và chất lượng

### 4.1. Bảo mật

| Nội dung | Biện pháp |
|---|---|
| Dữ liệu cá nhân | Số điện thoại và email vào kho ở dạng che một phần; ngày sinh chỉ giữ nhóm tuổi |
| Truy cập | Phân quyền theo vai trò; quản lý chi nhánh chỉ xem số của chi nhánh mình |
| Mã hoá | Mã hoá khi lưu và khi truyền trên toàn bộ đường đi |
| Vết kiểm toán | Ghi lại ai truy cập dữ liệu nhạy cảm, giữ 10 năm |
| Tách môi trường | Môi trường phát triển dùng dữ liệu đã làm mờ, không dùng dữ liệu thật |

Ma trận quyền chi tiết theo từng vai trò và từng tầng dữ liệu: [docs/08-operations/van-hanh.md](docs/08-operations/van-hanh.md).

### 4.2. Chất lượng số liệu

Hệ thống có **58 quy tắc kiểm tra** chạy tự động mỗi ngày trên bảy nhóm: đầy đủ, chính xác, nhất quán, duy nhất, hợp lệ, kịp thời, và mô hình chiều. Danh mục đầy đủ: [docs/05-quality/dq-rules.md](docs/05-quality/dq-rules.md).

Cơ chế xử lý khi phát hiện lỗi:

| Mức | Số quy tắc | Xử lý |
|---|---|---|
| Chặn | 45 | Dừng **nhánh** dữ liệu đó, không dừng cả hệ thống. Dòng lỗi đưa vào vùng cách ly, báo người phụ trách trong 15 phút |
| Cảnh báo | 12 | Vẫn nạp, gắn dấu hiệu trên báo cáo, tổng hợp báo hằng ngày |
| Ghi nhận | 1 | Chỉ ghi nhật ký |

Dòng lỗi phải được xử lý trong **3 ngày làm việc**, quá hạn thì báo lên cấp quản lý. Một loại dữ liệu lỗi chỉ dừng nhánh nạp tương ứng; các báo cáo còn lại vẫn cập nhật bình thường.

**Đối soát tự động hằng ngày:** hệ thống so doanh thu theo từng chi nhánh, từng ngày giữa kho và POS, và giữa POS với cổng thanh toán. Sai lệch vượt **0,1%** được cảnh báo ngay trong ngày.

### 4.3. Phân công trách nhiệm dữ liệu

Đề nghị Ban Tổng Giám đốc chỉ định người chịu trách nhiệm cho từng lĩnh vực. Người này có quyền và nghĩa vụ chốt định nghĩa chỉ tiêu, và là nơi hệ thống báo tới khi dữ liệu lĩnh vực đó có lỗi.

| Lĩnh vực | Bộ phận đề xuất |
|---|---|
| Khách hàng, thành viên, điểm thưởng | Chăm sóc khách hàng |
| Chi nhánh, buồng, lịch hẹn, điều trị | Vận hành |
| Nhân viên, phân ca | Nhân sự |
| Dịch vụ, sản phẩm, giá | Sản phẩm |
| Hoá đơn, thanh toán | Tài chính |
| Chiến dịch, chi phí quảng cáo | Marketing |

---

## 5. Tiêu chí nghiệm thu

Dự án chỉ được coi là hoàn thành khi đạt **toàn bộ** các tiêu chí sau.

| # | Tiêu chí | Cách đo |
|---|---|---|
| 1 | Số liệu đúng | Doanh thu ngày × chi nhánh khớp POS trong sai số **0,1%**, liên tục **30 ngày** |
| 2 | Số liệu đúng hạn | Sẵn sàng trước 08:00, đạt từ 99% số ngày trong quý |
| 3 | Chất lượng dữ liệu | 100% quy tắc mức Chặn đạt trong 30 ngày liên tiếp; từ 95% quy tắc mức Cảnh báo đạt, bình quân 30 ngày |
| 4 | Định nghĩa thống nhất | 24 chỉ tiêu có định nghĩa được các bộ phận **ký xác nhận** |
| 5 | Nền tảng cho phân tích nâng cao | Mô hình dự báo khách rời bỏ đạt **AUC từ 0,75** trên tập kiểm định 3 tháng gần nhất |
| 6 | **Được sử dụng thật** | Từ **80% quản lý chi nhánh** mở báo cáo ít nhất một lần mỗi tuần |
| 7 | Vận hành được | Có quy trình xử lý sự cố, phân ca trực, và tài liệu bàn giao |

**Tiêu chí 6 là quan trọng nhất về hiệu quả đầu tư.** Một hệ thống chạy đúng kỹ thuật nhưng không ai dùng thì không tạo ra giá trị. Đề nghị đưa tiêu chí này vào đánh giá kết quả công việc của cả bộ phận Dữ liệu và các quản lý chi nhánh.

---

## 6. Rủi ro và điều kiện tiên quyết

| # | Rủi ro | Khả năng | Ảnh hưởng | Biện pháp |
|---|---|---|---|---|
| 1 | POS không cấp bảng số liệu đối chiếu | Trung bình | **Cao** | Đưa vào điều khoản hợp đồng với nhà cung cấp; xem mục 1.2 |
| 2 | Gộp nhầm hai khách thành một | Cao | Trung bình | Độ tin cậy dưới 0,80 không tự động gộp, đưa vào danh sách người rà soát |
| 3 | Chất lượng dữ liệu nguồn kém hơn dự kiến | Cao | Trung bình | Giai đoạn 0 lấy mẫu và đánh giá trước; vùng cách ly giữ dòng lỗi |
| 4 | Các bộ phận không thống nhất được định nghĩa chỉ tiêu | Trung bình | **Cao** | Mốc M1 buộc có chữ ký; việc không chốt được trình lên Ban Tổng Giám đốc |
| 5 | Nhân sự kỹ thuật thay đổi giữa dự án | Trung bình | Trung bình | Tài liệu thiết kế đầy đủ; không để kiến thức chỉ nằm ở một người |
| 6 | Nguồn dữ liệu bên ngoài đổi hoặc bị giới hạn | Trung bình | Trung bình | Giữ bản sao dữ liệu quảng cáo 90 ngày trong hồ dữ liệu |
| 7 | Phạm vi mở rộng dần trong lúc làm | Cao | Trung bình | Mọi bổ sung ngoài phạm vi phải qua mốc kiểm soát |
| 8 | Người dùng không sử dụng báo cáo | Trung bình | **Cao** | Tiêu chí nghiệm thu số 6; đào tạo ở giai đoạn 7 |

### Năm dữ liệu chưa có nguồn

Các chỉ tiêu tương ứng **không tính được** nếu không được cung cấp:

| Dữ liệu cần | Chặn chỉ tiêu nào | Bên cung cấp | Hạn |
|---|---|---|---|
| Lịch làm việc và phân ca kỹ thuật viên | Năng suất kỹ thuật viên, Tỷ lệ lấp buồng | Vận hành | Tuần 10 |
| Giờ mở cửa từng chi nhánh theo ngày | Tỷ lệ lấp buồng | Vận hành | Tuần 10 |
| Cách tính giá vốn dịch vụ | Lợi nhuận gộp, Giá trị vòng đời khách hàng | Kế toán | Tuần 8 |
| Bảng số liệu đối chiếu doanh thu từ POS | **Tiêu chí nghiệm thu số 1** | Nhà cung cấp POS | Tuần 6 |
| Tỷ giá quy đổi điểm thưởng sang tiền | Giá trị điểm thưởng | Chăm sóc khách hàng | Tuần 14 |

---

## 7. Đề nghị phê duyệt

**Một — Phê duyệt phương án và lộ trình.** Thông qua thiết kế trình bày tại tài liệu này và kế hoạch 18 tuần với nguồn lực 16,95 người-tháng.

**Hai — Quyết định 8 nội dung chính sách tại mục 2.** Đây là các định nghĩa mang tính chuẩn công ty, ảnh hưởng trực tiếp đến mọi báo cáo về sau. Bộ phận kỹ thuật đã nêu phương án và khuyến nghị cho từng nội dung.

**Ba — Chỉ định người chịu trách nhiệm dữ liệu** theo mục 4.3, và **chỉ đạo ba việc** ở mục 1.2.

### Bảng ký duyệt

| Vai trò | Họ tên | Nội dung phê duyệt | Ngày | Chữ ký |
|---|---|---|---|---|
| Tổng Giám đốc | | Toàn bộ phương án, QĐ-06 | | |
| Giám đốc Tài chính | | QĐ-01, QĐ-02, QĐ-04, QĐ-05 | | |
| Giám đốc Vận hành | | QĐ-04, QĐ-07 | | |
| Giám đốc Marketing | | QĐ-03 | | |
| Giám đốc Công nghệ Thông tin | | QĐ-08 | | |

### Phụ lục — Tài liệu kỹ thuật

| Tài liệu | Nội dung |
|---|---|
| [README.md](README.md) | Tổng thiết kế: kiến trúc, 94 bảng kho phân tích, chuẩn thiết kế, trạng thái |
| [Thiet-Ke-DB-WebApp.md](Thiet-Ke-DB-WebApp.md) | Cơ sở dữ liệu vận hành của ứng dụng: 40 bảng, chống trùng lịch, lũy đẳng thanh toán |
| [Flow.md](Flow.md) | Đường đi của dữ liệu từ nguồn đến báo cáo |
| [Flow-DA.md](Flow-DA.md) | Trình tự thiết kế theo góc nhìn phân tích |
| [docs/](docs/) | Đặc tả chi tiết: DDL, ánh xạ nguồn, quy trình nạp, quy tắc chất lượng |
