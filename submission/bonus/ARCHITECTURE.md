# Quyết định Kiến trúc — Lakehouse Quan sát LLM ở mức 1 Tỷ Yêu cầu/Ngày

**Chủ đề:** A — Quan sát LLM ở Quy mô lớn  
**Tác giả:** Bài nộp Thử thách Bonus Ngày 18  
**Ngày:** 2026-05-04  
**Trạng thái:** Đề xuất (sẵn sàng cho đánh giá thiết kế cấp cao)

---

## 1. Tuyên bố Vấn đề

Một nhóm API mô hình nền tảng ghi lại mọi yêu cầu và phản hồi từ một nền tảng suy luận đa khách thuê (multi-tenant). Luồng dữ liệu thô (firehose) là **1 tỷ yêu cầu/ngày × ~5 KB/yêu cầu = 5 TB/ngày chưa nén**. Bốn ràng buộc cứng định hình mọi quyết định trong tài liệu này:

1. **Phân tích gần thời gian thực (Near-real-time)**: dashboard chi phí và độ trễ trên mỗi khách thuê phải làm mới sau mỗi 5 phút. Dữ liệu cũ quá 10 phút là vi phạm SLA.
2. **Chính sách lưu giữ kép**: các cặp prompt+response đầy đủ được giữ trong **7 ngày** (để xem xét sự cố, gỡ lỗi mô hình), sau đó **chỉ lưu giữ dữ liệu tổng hợp trong 1 năm** (để tuân thủ, giải quyết tranh chấp thanh toán). Các hàng dữ liệu thô sẽ bị xóa — không phải lưu trữ (archive).
3. **Bảo mật ưu tiên PII (PII-first)**: prompt của người dùng chứa tên, số điện thoại, số ID quốc gia, dữ liệu tài chính. Không có PII thô nào được phép chạm tới bất kỳ giao diện truy vấn nào. Quá trình mã hóa (tokenization) phải diễn ra tại thời điểm nhập liệu (ingestion), trước khi bất kỳ con người hay tiến trình nào có thể đọc dữ liệu.
4. **Ngân sách lưu trữ cứng**: tổng chi phí lưu trữ ≤ **$5,000/tháng**. Chi phí tính toán (compute) là một ngân sách riêng.

Khó khăn là: bốn ràng buộc này mâu thuẫn trực tiếp với nhau. Việc lưu giữ toàn vẹn tối ưu cho xem xét sự cố nhưng lại tối đa hóa chi phí lưu trữ. Việc nén tối đa tối ưu cho chi phí nhưng có thể làm chậm quá trình hiển thị dashboard 5 phút. Việc mã hóa PII tại thời điểm nhập liệu làm tăng độ trễ và có thể vượt qua ngân sách của micro-batch streaming. Tài liệu này giải thích cách chúng tôi giải quyết từng mâu thuẫn.

---

## 2. Sơ đồ Kiến trúc

```
┌──────────────────────────────────────────────────────────────────────────┐
│  CÁC NHÀ SẢN XUẤT (PRODUCERS)                                              │
│  Máy chủ LLM API (đa vùng)       →  Kafka topic: llm-events                │
│  Phân vùng theo: tenant_id (512 phân vùng)                                 │
│  Lưu giữ: 24 giờ  |  Thông lượng: duy trì ~58 MB/s, đỉnh điểm 200 MB/s     │
└─────────────────────────────────┬────────────────────────────────────────┘
                                   │
                    Spark Structured Streaming
                    micro-batch, trigger 30s
                    + Mã hóa PII bằng Presidio (inline)
                    + Khóa FPE theo từng khách thuê từ AWS Secrets Manager
                                   │
┌──────────────────────────────────▼────────────────────────────────────────┐
│  BRONZE  s3://lakehouse/delta_bronze/                                      │
│  Lược đồ: request_id, tenant_id, model_id, ts, input_tokens,              │
│          output_tokens, latency_ms, cost_usd,                              │
│          prompt_tokenized (An toàn PII), response_tokenized (An toàn PII), │
│          error_code, region                                                 │
│  Phân vùng: date=YYYY-MM-DD / hour=HH                                      │
│  Z-order:   (tenant_id, model_id)                                          │
│  Định dạng: Delta Lake + Parquet/Zstd                                      │
│  Lưu giữ: 7 ngày → Xóa tự động bằng S3 Lifecycle (+ Delta VACUUM hàng đêm) │
│  Kích thước: ~500 GB/ngày nén → ~3.5 TB cửa sổ trực tiếp                   │
└──────────────────────────────────┬────────────────────────────────────────┘
                                   │
                    Spark batch job, chạy mỗi 5 phút
                    (đọc Bronze CDF, khử trùng lặp (dedup), xác thực,
                     bổ sung chi phí từ danh mục giá)
                                   │
┌──────────────────────────────────▼────────────────────────────────────────┐
│  SILVER  s3://lakehouse/delta_silver/                                      │
│  Lược đồ: giống Bronze + quality_flag, cost_usd_validated,                │
│          dedup_hash, late_arrival_flag                                     │
│  Phân vùng: date / hour                                                    │
│  Z-order:   (tenant_id, model_id, latency_ms)                             │
│  Định dạng: Delta Lake + Parquet/Zstd                                      │
│  Lưu giữ: 7 ngày (cửa sổ xem xét sự cố)                                    │
│  Kích thước: ~600 GB/ngày → ~4.2 TB cửa sổ trực tiếp                       │
│  Watermark: 2 giờ (xử lý các sự kiện đến muộn từ các vùng biên)            │
└──────────────────────────────────┬────────────────────────────────────────┘
                                   │
                    Spark batch job, mỗi 5 phút
                    MERGE INTO gold (upsert, lũy đẳng - idempotent)
                                   │
┌──────────────────────────────────▼────────────────────────────────────────┐
│  GOLD  s3://lakehouse/delta_gold/                                          │
│                                                                            │
│  tenant_metrics_5m                                                         │
│    Phân vùng: tenant_id / date                                             │
│    Độ mịn (Granularity): Gộp theo 5 phút                                   │
│    Cột: request_count, error_count, total_cost_usd,                       │
│             latency_p50_ms, latency_p95_ms, latency_p99_ms,               │
│             input_tokens_sum, output_tokens_sum                            │
│    Định dạng: Delta + Parquet/LZ4  (đọc nhanh cho dashboard)               │
│    Lưu giữ: 13 tháng                                                       │
│                                                                            │
│  tenant_daily_agg                                                          │
│    Cuộn (Rollup) từ tenant_metrics_5m → theo ngày                          │
│    Phân vùng: date                                                         │
│    Lưu giữ: 13 tháng (chuyển S3-IA sau 30 ngày)                            │
│    Kích thước: ~1 GB/ngày → 365 GB/năm                                     │
└──────────────────────────────────┬────────────────────────────────────────┘
                                   │
         ┌─────────────────────────┼───────────────────────────┐
         ▼                         ▼                           ▼
  Dashboard (5 phút)       Xem xét Sự cố                  Thanh toán / Tuân thủ
  Trino → Gold             Spark → Silver                 Spark → Gold
  tenant_metrics_5m        (Cửa sổ 7 ngày,                tenant_daily_agg
  p95 < 2s                 cổng phê duyệt PII)            (1 năm)
```

**Lớp Phủ Danh mục & Quản trị (Unity Catalog):**
- Bronze/Silver: bảo mật cấp độ cột (column-level security) che dấu `prompt_tokenized` / `response_tokenized` — chỉ vai trò `incident-review` mới có thể đọc chúng, và mọi thao tác đọc đều được ghi vào `audit.pii_access`
- Gold: không có cột PII; mở cho vai trò `analyst` (nhà phân tích)

---

## 3. Các Quyết định Chính

### Quyết định 1 — Định dạng Bảng: Delta Lake thay vì Apache Iceberg và Apache Hudi

**Tôi chọn Delta Lake.**

Tôi đã loại bỏ **Apache Iceberg** vì: (a) Cụm Z-order gốc (native Z-order clustering) không có trong đặc tả của Iceberg — nó chỉ tồn tại dưới dạng một gợi ý `ORDER BY` đặc thù của Spark, không ghi các thống kê min/max trên mỗi file giống như cách Delta OPTIMIZE thực hiện; ở mức 500 GB/ngày với 10,000 khách thuê, việc loại bỏ (pruning) truy vấn lọc theo khách thuê hoàn toàn dựa vào thống kê Z-order, khiến đây là một sự phụ thuộc bắt buộc. (b) Xóa vector (Deletion vectors) — cơ chế để xóa mềm các hàng PII mà không cần ghi lại toàn bộ file — là tính năng riêng của Delta (DBR 7.0+, OSS Delta 2.0+); phương pháp tương đương của Iceberg (`delete files`) sử dụng xóa theo vị trí (positional deletes) yêu cầu thao tác merge-on-read, làm tăng độ khuếch đại đọc (read amplification) tại thời điểm truy vấn lớp Gold. (c) Luồng dữ liệu thay đổi (Change Data Feed - CDF), mà thiết kế này sử dụng để chạy tính toán tăng dần Bronze→Silver mỗi 5 phút, là tính năng gốc của Delta và sẽ cần phải triển khai lại bằng các bản snapshot của Iceberg — khả thi, nhưng tốn công hơn.

Tôi đã loại bỏ **Apache Hudi** vì: Các bảng Merge-on-Read (MOR) của Hudi phân bổ chi phí ghi bằng cách trì hoãn việc nén (compaction), nhưng điều này tạo ra khuếch đại đọc — mọi truy vấn đều phải gộp các file gốc (base files) và nhật ký thay đổi (delta logs) vào thời điểm quét. Đối với tốc độ làm mới dashboard 5 phút trên lớp Gold, độ trễ đọc là ràng buộc chính, chứ không phải thông lượng ghi. Copy-on-Write (COW) của Hudi loại bỏ điều này nhưng lại gây ra việc phải ghi lại toàn bộ file trong mỗi lần cập nhật, với 500 GB/ngày, điều này sẽ tiêu tốn hết ngân sách tính toán. Z-order của Hudi (`clustering`) kém tiện dụng hơn OPTIMIZE ZORDER của Delta và yêu cầu một HoodieClusteringJob riêng biệt, làm tăng độ phức tạp trong vận hành.

**Delta cung cấp: Các đảm bảo ACID, Z-order với các thống kê ở cấp độ tệp, Deletion Vectors cho việc gỡ bỏ PII tuân thủ GDPR, Change Data Feed cho các luồng đường ống tăng dần, và du hành thời gian 7 ngày (7-day time travel) cho việc xem xét sự cố — tất cả nằm trong một định dạng.**

---

### Quyết định 2 — Danh mục (Catalog): Unity Catalog thay vì AWS Glue và Apache Hive Metastore

**Tôi chọn Unity Catalog (tự lưu trữ OSS hoặc quản lý bởi Databricks).**

Tôi đã loại bỏ **AWS Glue** vì nó không có bảo mật cấp độ cột. Không thể che giấu cột `prompt_tokenized` ở lớp Silver theo từng vai trò trong Glue — việc áp dụng quyền truy cập bắt buộc phải kiểm tra ở cấp độ ứng dụng, dễ bị lỗi và vượt qua (bypass) được các truy vấn Trino. Glue cũng không có nguồn gốc dữ liệu (data lineage) nội tại; chúng ta sẽ cần một tích hợp OpenLineage riêng biệt để theo dõi hàng dữ liệu Gold nào xuất phát từ phân vùng Bronze nào, làm tăng chi phí công cụ (~$300/tháng cho triển khai Marquez) và bề mặt vận hành.

Tôi đã loại bỏ **Apache Hive Metastore** vì nó không có lớp xác thực ngoài Kerberos (đắt đỏ về mặt vận hành), không có che dấu cột, không có nguồn gốc dữ liệu, và thực tế là một sự lựa chọn lỗi thời cho các triển khai dự án mới tinh vào thời điểm 2025+.

Unity Catalog cung cấp: (a) **che giấu cấp độ cột (column-level masking)** — `prompt_tokenized` trả về `'[MASKED]'` cho tất cả các vai trò trừ `incident-review`; (b) **bảo mật cấp độ hàng (row-level security)** — các nhà phân tích của khách thuê chỉ có thể nhìn thấy các hàng của riêng họ trong lớp Gold mà không cần lọc tại lớp ứng dụng; (c) **nhật ký kiểm toán tích hợp (built-in audit log)** — mọi `SELECT` trên các cột PII ở lớp Bronze/Silver sẽ ghi lại một hàng vào `system.access.audit`, thỏa mãn yêu cầu "ghi log mọi hoạt động đọc PII" mà không cần code tùy chỉnh; (d) **nguồn gốc dữ liệu (data lineage)** — nguồn gốc cấp độ cột từ Bronze đến Gold cho phép nhóm tuân thủ trả lời câu hỏi "chỉ số đo lường lớp Gold này xuất phát từ phân vùng Bronze nào" trong một giao diện nhấp chuột (point-and-click UI).

---

### Quyết định 3 — Token hóa PII: Presidio + FPE (Vault Transit) thay vì Che dấu bằng Regex và Mã hóa Cột

**Tôi chọn Microsoft Presidio (phát hiện dựa trên NER) + Mã hóa giữ nguyên định dạng (Format-Preserving Encryption - FPE) thông qua HashiCorp Vault Transit.**

Nhận thức cốt lõi là các prompt LLM là văn bản tự do — regex cứng nhắc sẽ bắt được các mẫu đã biết (điện thoại: `\d{3}-\d{4}`) nhưng sẽ bỏ lỡ PII đã được diễn giải khác ("số của tôi là không-chín-tám..."), các thực thể đa ngôn ngữ (tên tiếng Việt, số CCCD), và PII phụ thuộc ngữ cảnh (một prompt nói "bệnh nhân của tôi John Smith có..."). Nhận dạng Thực thể Có tên (Named Entity Recognition - NER) dựa trên ML của Presidio nắm bắt các trường hợp này với độ thu hồi (recall) ~92% trên tiếng Anh, ~85% trên tiếng Việt sau khi tinh chỉnh (fine-tuning).

Tôi đã loại bỏ **băm SHA-256 đơn giản** vì: (a) nó chỉ có một chiều — nếu một điều tra viên sự cố hợp pháp cần liên kết các token của prompt qua các yêu cầu để tái tạo một cuộc trò chuyện, việc băm không cung cấp liên kết nào; (b) các token hash là các ký tự hex 64 phá vỡ phân tích cú pháp văn bản và làm tăng kích thước cột; (c) luân chuyển khóa (key rotation) yêu cầu băm lại toàn bộ tập dữ liệu.

Tôi đã loại bỏ **mã hóa ở cấp độ cột (ví dụ: Parquet encrypt)** vì: (a) Mã hóa cấp độ Parquet ngăn chặn predicate pushdown — `WHERE tenant_id = 'X'` trên cột `tenant_id` đã mã hóa sẽ yêu cầu giải mã hoàn toàn trước khi lọc, với khối lượng 500 GB/ngày thì tốc độ này quá chậm; (b) luân chuyển khóa yêu cầu viết lại toàn bộ file.

**FPE thông qua Vault Transit** duy trì định dạng của token (một số điện thoại đã token hóa vẫn là chuỗi 10 chữ số, một email đã token hóa vẫn trông giống `xxxx@yyyy.zz`), cho phép các bộ phân tích cú pháp văn bản ở hạ nguồn (downstream) hoạt động mà không cần sửa đổi. Các khóa Vault riêng cho từng khách thuê có nghĩa là sự cố vi phạm của một khách thuê không làm lộ PII của khách thuê khác. Việc luân chuyển khóa là một hoạt động của Vault (gói lại - re-wrap) — không cần phải viết lại dữ liệu.

**Ngân sách độ trễ Presidio**: tại micro-batch 30 giây xử lý ~1.7M bản ghi/batch (1 tỷ/ngày ÷ 48,000 batch/ngày... đợi đã, 86400s/30s = 2880 micro-batch, 1 tỷ/2880 = ~347K bản ghi/batch), Presidio phải token hóa ở tốc độ ≥11,600 bản ghi/giây. Một node thực thi Spark 16 lõi chạy Presidio qua Python UDF đạt ~15,000 bản ghi/giây — sát nút nhưng khả thi với 4 node thực thi (executor nodes).

---

### Quyết định 4 — Chiến lược Phân vùng: date/hour + Z-order(tenant_id) thay vì chỉ phân vùng theo tenant_id

**Tôi chọn phân vùng theo (date, hour) với Z-order clustering trên (tenant_id, model_id).**

Tôi đã loại bỏ **phân vùng theo tenant_id** vì ba lý do: (a) Các quy tắc S3 Lifecycle áp dụng cho toàn bộ tiền tố khóa (key prefixes) — để xóa bỏ dữ liệu Bronze sau 7 ngày, chúng ta cần một tiền tố dựa trên ngày tháng (`delta_bronze/date=2026-04-27/`); bố cục phân vùng theo khách thuê không thể diễn đạt "xóa dữ liệu cũ hơn 7 ngày" mà không cần liệt kê và xóa từng đối tượng cá biệt, điều này với 1 tỷ bản ghi/ngày là thao tác API S3 với độ phức tạp O(1 tỷ) và tiêu tốn khoảng $5,000/ngày chỉ riêng cho các lệnh LIST + DELETE. (b) Các khách thuê nóng (hot tenants - ví dụ công ty lớn) tạo ra dữ liệu gấp 100 lần so với các khách thuê lạnh (startup) — phân vùng theo khách thuê gây ra sự mất cân bằng nghiêm trọng, làm chậm lịch trình tác vụ của Spark và sinh ra các tập tin phân vùng từ 10–50 GB. (c) Với 10,000 khách thuê và việc nén hàng giờ, `date/hour/tenant` sẽ tạo ra 240,000 tiền tố thư mục/ngày, vượt quá mật độ không gian khóa được khuyến nghị của S3.

**Giải pháp Z-order**: sau khi các lệnh OPTIMIZE chạy hàng giờ, các tệp trong mỗi phân vùng `date/hour` được sắp xếp theo (tenant_id, model_id), ghi lại các thống kê cột min/max ở cuối mỗi tệp Parquet. Một truy vấn `WHERE tenant_id = 'bigcorp' AND ts BETWEEN ...` sử dụng các số liệu thống kê này để bỏ qua hơn 95% số tệp — tương đương với chỉ mục phụ nhưng không có chi phí ghi quá mức của chỉ mục thực sự.

**Mục tiêu kích thước tệp**: 500 GB/ngày ÷ 24 giờ ÷ 16 tệp mỗi phân vùng = ~1.3 GB/tệp sau lệnh OPTIMIZE. Lớn hơn mức tối thiểu 512 MB được khuyến nghị; có thể chấp nhận được.

---

### Quyết định 5 — Công cụ Streaming: Spark Structured Streaming (micro-batch 30s) thay vì Apache Flink

**Tôi chọn Spark Structured Streaming với trigger micro-batch 30 giây.**

SLA của dashboard là mức làm mới 5 phút. Lợi thế chính của Flink — xử lý sự kiện dưới 1 giây (sub-second) — không quan trọng ở đây. Một micro-batch 30 giây sẽ đưa dữ liệu vào lớp Bronze trong vòng 60 giây kể từ khi được sinh ra (batch 30s + commit ~15s), và đường ống Bronze→Silver→Gold mất thêm 2–3 phút nữa. Tổng từ đầu đến cuối: dữ liệu hiển thị trên dashboard trong khoảng ~4 phút kể từ khi sự kiện xảy ra — hoàn toàn thoải mái so với SLA.

Tôi đã loại bỏ **Apache Flink** vì: (a) một cụm Flink chuyên biệt (tối thiểu 3 TaskManager cho HA trong sản xuất) làm tăng ~$800–1,200/tháng chi phí tính toán mà không mang lại lợi ích về độ trễ đối với SLA của chúng tôi; (b) Trình kết nối Delta của Flink (delta-flink) vẫn ở bản beta vào cuối năm 2025 và thiếu hỗ trợ Change Data Feed, mà thiết kế này đang phụ thuộc vào để xử lý gia tăng Silver→Gold; (c) Flink đưa ra một miền vận hành riêng rẽ (JobManager HA, quản lý savepoint, xung đột classpath) mà nhóm sẽ cần sở hữu cùng với Spark — nhân đôi bề mặt ứng trực (on-call).

**Nếu SLA bị thắt chặt <30 giây** trong tương lai, hướng chuyển đổi là: giữ Spark SS cho luồng Silver→Gold (dạng batch, không đổi), chỉ thay thế job Kafka→Bronze bằng Flink và dùng chính bảng Delta đó làm sink. Phần còn lại của pipeline không bị ảnh hưởng.

---

### Quyết định 6 — Nén (Compression): Zstd cho Bronze/Silver, LZ4 cho Gold

**Tôi chọn Zstd (mức 3) cho Bronze và Silver, LZ4 cho Gold.**

Văn bản prompt/response của LLM có tính nén rất cao — ngôn ngữ tự nhiên có khoảng ~1 bit/ký tự entropy so với 8 bit/ký tự chưa nén. Zstd mức 3 đạt tỷ lệ nén 10–15:1 cho tiếng Anh: từ 5 TB/ngày thô → **~350–500 GB/ngày lưu trữ ở lớp Bronze**. Đây là cơ chế chính để giữ cho chi phí lưu trữ nằm trong ngân sách.

Tôi đã loại bỏ **Snappy** cho lớp Bronze: Snappy đạt độ nén 2–4:1 cho text (thiết kế cho dữ liệu nhị phân, không phải văn bản NLP), điều này sẽ làm Bronze tốn khoảng 1.25–2.5 TB/ngày — phá vỡ ngân sách lưu trữ $5K.

Tôi đã loại bỏ **Zstd cho lớp Gold**: bảng `tenant_metrics_5m` của lớp Gold được dashboard đọc 12 lần/giờ (mỗi 5 phút). Zstd giải nén với tốc độ ~1.1 GB/s so với ~3 GB/s của LZ4. Với một truy vấn dashboard quét qua dữ liệu 5 phút trên Gold (≈50 MB), thì sự chênh lệch là 45 ms với 15 ms — không đáng kể. Tuy nhiên, với 50 người dùng đồng thời tải lại dashboard mỗi 5 phút, CPU giải nén trên cụm Trino sẽ trở thành nút cổ chai nếu dùng Zstd; LZ4 giúp duy trì độ trễ truy vấn p95 dưới 2 giây.

---

### Quyết định 7 — Xóa PII: Delta Deletion Vectors thay vì Viết lại Toàn bộ Phân vùng

**Tôi chọn Delta Deletion Vectors cho các yêu cầu xóa bỏ (right-to-erasure) theo GDPR.**

Khi một khách thuê gửi yêu cầu "quên tôi đi" (forget me), tất cả các hàng có chứa PII đã token hóa của người dùng đó phải bị xóa khỏi Bronze và Silver. Tại quy mô 500 GB/ngày × 7 ngày = 3.5 TB ở lớp Bronze, việc ghi lại toàn bộ bảng để xóa các hàng của 1 người dùng sẽ tốn ~2 giờ và tiêu thụ ~$15 tiền tính toán — không thể chấp nhận được cho SLA yêu cầu xóa trong vòng 72 giờ (GDPR).

Delta Deletion Vectors đánh dấu vị trí các hàng đã bị xóa vào một tệp phụ `.dvbin` nhỏ mà không cần phải viết lại tệp Parquet gốc. Thao tác xóa này được ACID xác nhận (commit) trong phần nghìn giây (ms). Các lần đọc sau đó sẽ bỏ qua các hàng đã xóa thông qua vector bitmap xóa này. Một lệnh REWRITE (OPTIMIZE) hàng đêm sẽ gộp chung các thao tác xóa vào các file sạch mới.

Tôi đã loại bỏ **partition rewrite** (viết lại phân vùng: xóa và tái tạo phân vùng, loại trừ các hàng đã xóa): thao tác này không theo nguyên tắc nguyên tử (atomic) — sẽ có một khoảng thời gian phân vùng này không tồn tại, khiến cho dashboard truy vấn trả về dữ liệu bằng không cho cửa sổ thời gian đó. Hơn nữa, nó yêu cầu phải load toàn bộ dữ liệu phân vùng vào bộ nhớ của Spark trước khi ghi file mới.

Tôi đã loại bỏ **cột cờ (flag) để soft-delete** (`is_deleted BOOLEAN`): dữ liệu lúc này vẫn tồn tại vật lý và những người nội bộ mang ý đồ xấu hoặc một lỗi trong filter có thể đọc được; vi phạm nguyên tắc "quyền được quên" (right to be forgotten) của GDPR, vì GDPR yêu cầu phải xóa bỏ vật lý.

---

## 4. Các Chế độ Lỗi (Failure Modes)

### FM-1 — Tăng đột biến độ trễ (Lag) Kafka Consumer: Bronze bị trễ >2 giờ

**Căn nguyên**: Có một đợt khởi chạy sản phẩm tạo tiếng vang lớn hoặc bị DDoS khiến lưu lượng LLM API đầu nguồn tăng vọt x5 lần trong 30 phút (thêm 5 tỷ sự kiện). Lag (độ trễ) của Kafka consumer group tăng lên đến 500 triệu tin nhắn (≈50 phút tồn đọng ở thông lượng bình thường).

**Phát hiện**: Số liệu Kafka consumer-group metric `consumer_lag > 50M` sẽ kích hoạt báo động PagerDuty trong vòng 2 phút. Dashboard Grafana hiển thị timestamp của Bronze watermark bị lệch — panel "độ tươi của dữ liệu" (data freshness) sẽ chuyển sang màu vàng ở 10 phút, và màu đỏ nếu trễ 30 phút. Dashboard lớp Gold sẽ hiện dữ liệu cũ với dòng banner: "Cập nhật lần cuối 12 phút trước — đang trong quá trình bắt kịp (catch-up)."

**Khắc phục**: Spark SS tự động khởi động lại từ offset cuối cùng được ghi nhớ trong `_checkpoints/` của Kafka. Nới rộng số node thực thi của cụm Spark từ 8 lên 20 (autoscaling của Databricks Jobs Cluster, tốn khoảng 5 phút để kích hoạt). Tạm thời nhân đôi `maxOffsetsPerTrigger` từ 350K lên 700K mỗi micro-batch để tăng tốc độ catch-up. Nếu trễ > 4 giờ, vô hiệu hóa mã hóa (tokenization) nội tuyến (inline) của Presidio và ghi cờ `needs_tokenization=true` vào lớp Bronze — một job backfill sau đó sẽ token hóa những hàng này song song, trong khi job chính ingest sẽ tiếp tục để bắt kịp tiến độ. Ước tính thời gian bắt kịp: Thời lượng trễ 2× ÷ Thông lượng 2× = bằng với khoảng thời gian bị trễ gốc. Sự tăng vọt lưu lượng trong 4 giờ sẽ mất ~4 giờ để bắt kịp.

---

### FM-2 — Âm tính giả (False-Negative) của Presidio: PII thô bị lọt vào lớp Silver

**Căn nguyên**: Một định dạng PII mới không có trong mô hình của Presidio (ví dụ: định dạng ID quốc gia mới, schema số tài khoản dành riêng cho vendor) không bị phát hiện lúc token hóa và lọt được vào cột `prompt_tokenized` của lớp Silver. Audit log của Unity Catalog cho thấy một analyst đã chạy câu lệnh `SELECT response_tokenized FROM silver.events WHERE tenant_id = 'X'` — họ có khả năng đã nhìn thấy PII thô.

**Phát hiện**: Lệnh scan phân loại dữ liệu (Amazon Macie hoặc một custom Spark job dùng Regex + Presidio để quét 0.1% dữ liệu ở các phân vùng Silver mới) chạy hàng đêm qua Unity Catalog sẽ phát hiện một mẫu trông giống như PII. Cảnh báo sẽ được kích hoạt trong vòng 24 giờ. Thêm vào đó, có cả hệ thống cảnh báo red-team (canary): mỗi tuần 1 lần, một request giả mạo mang theo một Fake SSN có mẫu nhất định sẽ được chèn vào; nếu dữ liệu này xuất hiện ở lớp Silver mà không bị che lấp (unmasked), đường ống giám sát (detection pipeline) sẽ báo động ngay lập tức.

**Hoàn tác (Rollback) sử dụng Delta time travel**: đây là một cơ chế thiết yếu của Ngày 18.
1. Xác định các phân vùng ngày tháng bị ảnh hưởng từ timestamp của offset Kafka tại thời điểm xảy ra sự kiện lỗi (undetected event) đầu tiên.
2. `RESTORE TABLE silver.events TO VERSION AS OF <version_before_contamination>` — Delta tái hiện lại transaction log (lịch sử giao dịch) để đưa bảng trở về trạng thái chưa bị nhiễm lỗi (contamination) trước đó chỉ trong vài giây (không di chuyển dữ liệu, chỉ là cập nhật con trỏ trong transaction log).
3. Cập nhật mô hình Presidio NER với mẫu regex hoặc các pattern mới nhận diện (huấn luyện lại mô hình hoặc thêm rule regex mới).
4. Chạy lại job Bronze→Silver trong phạm vi các ngày bị ảnh hưởng với trình token hóa đã được sửa lỗi. MERGE INTO lớp Silver bằng cách sử dụng `request_id` làm khóa idempotency (chống trùng lặp).
5. Phiên bản Silver cũ bị nhiễm PII sẽ bị thu dọn trong lần chạy `VACUUM` tiếp theo sau khi khoảng thời gian lưu giữ cũ (RETAIN window) đã hết.
6. Viết báo cáo sự cố bảo mật (security incident report); thông báo cho các khách thuê bị ảnh hưởng theo tiêu chuẩn về rò rỉ bảo mật (breach disclosure SLA).

---

### FM-3 — Lỗi Cục bộ lúc MERGE ở Gold: Dashboard hiển thị khoảng trống trong Biểu đồ thời gian (Time-Series)

**Căn nguyên**: OOM (Out of Memory) của node thực thi Spark trong lúc chạy `MERGE INTO gold.tenant_metrics_5m` cho khoảng thời gian 14:05–14:10 UTC. Job bị treo ở giữa tiến trình ghi (mid-write). Dashboard hiển thị khoảng trống trong các sparkline (biểu đồ đường nhánh) của mọi khách thuê từ lúc 14:05 đến 14:15.

**Phát hiện**: Status của Spark job chuyển sang FAILED ở lớp điều phối (Airflow / Databricks Jobs). Transaction log của Delta cho `gold.tenant_metrics_5m` chỉ ra là không có COMMIT (xác nhận) đối với thời điểm dự kiến. Một tác vụ kiểm tra sức khỏe độc lập (chạy mỗi 2 phút, check `MAX(window_end)` trên Gold) kích hoạt báo động ở mức P2 nếu timestamp lâu nhất (max timestamp) bị trễ >10 phút so với thời gian trên đồng hồ (wall clock).

**Tại sao việc này lại an toàn (Ngày 18: Các giao dịch ACID)**: write-ahead log (nhật ký ghi trước) của Delta sẽ đảm bảo rằng job MERGE bị fail hoàn toàn không để lại trạng thái dở dang (partial state) nào — hoặc là giao dịch đã commit nguyên vẹn toàn bộ (atomic) hoặc là không có gì được commit cả. Không bao giờ tồn tại trường hợp bảng Gold bị ghi dở chừng. Khoảng trống (gap) của dashboard là do trễ dữ liệu (staleness issue), không phải là dữ liệu bị tham nhũng (data corruption).

**Khắc phục**: Việc thiết kế theo mô hình lũy đẳng (idempotent) của `MERGE INTO` đảm bảo rằng chạy lại khoảng 5 phút đó trên Silver cũng chỉ trả về cùng một kết quả trên Gold y hệt nhau. Bộ điều phối (orchestrator) sẽ tự động chạy lại (retries) job sau 60 giây. Căn nguyên khiến OOM (thiếu bộ nhớ RAM): việc thực hiện MERGE trên Gold đối với một tenant (khách thuê) có 100M request/ngày đòi hỏi phải shuffle khá nặng; cách sửa lỗi là thêm `REPARTITION BY tenant_id` ngay trước MERGE để không một tác vụ (task) duy nhất nào bị bắt ôm trọn cả một lượng dữ liệu quá khủng của chỉ một khách thuê. Hành động hậu sự cố: Đặt giới hạn đồng thời ghi (write concurrency limit) trong quá trình MERGE lớp Gold với từng tenant.

---

### FM-4 — Dữ liệu thống kê Z-Order lỗi thời: Hiệu năng Truy vấn Khách thuê Tụt dốc 10×

**Căn nguyên**: Một phân vùng ở lớp Bronze (`date=2026-05-01/hour=14`) đã đón một lượng lưu lượng ghi gấp 3× bình thường do tiến trình backfill (của quá trình FM-1 catch-up). Giờ phân vùng đó mang chứa tầm 2,400 tập tin nhỏ cỡ ~130 KB thay vì mục tiêu là 16 file cỡ ~1.3 GB. Dữ liệu số liệu thống kê Z-Order bị cũ (stale). Truy vấn dashboard cho khách thuê (`WHERE tenant_id = 'bigcorp' AND date = '2026-05-01'`) phải scan 2,400 tệp thay vì chỉ 40–50 tệp bình thường, làm tăng thời gian query từ 2 giây sang hơn 20+ giây.

**Phát hiện**: Trino báo log chậm trên query (giới hạn: p95 > 10s đối với các dashboard query) gây cảnh báo mức P2. Đã xác nhận nguyên nhân sự cố thông qua lệnh `DESCRIBE DETAIL delta.delta_silver` — `numFiles` tại phân vùng chịu ảnh hưởng là 2,400 so với mức bình thường ≤16.

**Khắc phục**: Lệnh `OPTIMIZE delta.delta_silver WHERE date = '2026-05-01' AND hour = 14 ZORDER BY (tenant_id, model_id)` sẽ tái thiết lập 2,400 tập tin kể trên về chỉ còn ~16 tệp có mang những thống kê chuẩn Z-order. Thời gian chạy: ~8 phút đối với tầm dữ liệu 1.3 GB. Độ hiệu quả pruning từ Z-order sẽ quay lại mức hiệu năng >95%. Phòng ngừa: OPTIMIZE được xếp lịch chạy hàng đêm để quét mọi phân vùng Bronze/Silver thuộc khoảng thời gian 24 giờ vừa qua; bất cứ một phân vùng nào từng nhận các lệnh ghi backfill đều sẽ gây kích hoạt thao tác OPTIMIZE bổ sung ngay lập tức nhờ vào Delta post-commit hook.

---

## 5. Ước tính Chi phí (Tính nhẩm đại khái)

### Lưu trữ

| Lớp (Tier) | Kích thước nén hàng ngày | Lưu giữ | Khối lượng Trực tiếp | $/GB-tháng | Chi phí Hàng tháng |
|---|---|---|---|---|---|
| Bronze | 500 GB/ngày (Zstd 10:1) | 7 ngày | 3.5 TB | $0.023 (S3 Standard) | **$80** |
| Silver | 600 GB/ngày (Zstd ~8:1) | 7 ngày | 4.2 TB | $0.023 (S3 Standard) | **$97** |
| Gold tenant_metrics_5m | ~10 GB/ngày (dữ liệu tổng hợp) | 13 tháng | 4 TB | $0.023 / $0.0125 (S3-IA sau 30 ngày) | **$32** |
| Gold tenant_daily_agg | ~1 GB/ngày | 13 tháng | 390 GB | $0.0125 (S3-IA) | **$5** |
| Kafka (MSK, lưu 24h, 3× bản sao) | 5 TB/ngày × 3 = 15 TB | 24h | 15 TB EBS gp3 | $0.08/GB-tháng | **$1,200** |
| Delta transaction logs + checkpoints | ~5 GB/ngày | 30 ngày | 150 GB | $0.023 | **$4** |
| **Tổng Lưu trữ** | | | | | **~$1,418/tháng** |

**Toán tính tỉ lệ nén**: 5 TB/ngày file raw × (1/10 theo tỉ lệ Zstd) = 500 GB/ngày ở Bronze. Kafka xài ổ EBS (chứ không phải theo nén của S3 object) — đây là phần ăn tiền nhất của toàn bộ khoản phí.

**Phương án thay thế về giá tiền của Kafka**: Thay EBS của MSK thành MSK Serverless (tính theo tốc độ băng thông throughput, $0.10/GB) = 5 TB/ngày × 30 ngày × $0.10 = **$1,500/tháng** — tương đối tương đương. Hoặc: chọn Kinesis Data Streams có $0.015/shard-hour × 50 shards × 720 hours = $540/tháng nhưng chi phí mạng tải ra (egress) sẽ phải công thêm ~$300 — tổng thể cũng tương đương.

Tổng phí lưu trữ ≈ **$1,418/tháng → 28% trong cái ngân sách $5K**. Có một khoản dư giả hợp lý.

### Tính toán (Compute) (Bối cảnh, không nằm trong ngân sách Lưu trữ)

| Thành phần | Thông số máy (Spec) | Giờ/tháng | Đơn giá | Chi phí hàng tháng |
|---|---|---|---|---|
| Spark Streaming (Kafka→Bronze+Silver) | r5.4xlarge × 6, spot (giảm giá ~70%) | 720h | $0.50/node-hr | **$2,160** |
| Spark Batch (Silver→Gold, mỗi 5 phút) | r5.2xlarge × 4, spot | 720h | $0.25/node-hr | **$720** |
| Trino (Gold dashboard queries) | c5.2xlarge × 4, on-demand | 720h | $0.34/node-hr | **$979** |
| OPTIMIZE jobs (chạy hàng đêm) | r5.2xlarge × 8, spot, 2h/đêm | 60h/tháng | $0.25/node-hr | **$120** |
| **Tổng phí Compute** | | | | **~$3,979/tháng** |

**Tổng chi phí cuối cùng (Lưu trữ + Tính toán): ~$5,397/tháng**. Vượt qua chút đỉnh nếu sử dụng kiểu máy ảo Spark tính theo giờ (on-demand); ở dưới mức $5K nếu tính trên giá instance dạng spot 70% trên cluster chạy stream. Nếu như ngân sách này lại gắt (binding): thu hẹp cụm máy Trino xuống thành 2 node (truy vấn p95 dashboard lúc đó là ≤4s chứ không còn ≤2s) → tiết kiệm được $490/tháng.

---

## 6. Lát Cắt MVP — Tuần 1

**Mục tiêu**: Chứng minh sự rủi ro về mặt công nghệ (khó nhằn nhất) — hạ cánh an toàn PII Bronze ở môi trường có throughput thực tế — trước khi bắt tay làm lớp Silver hoặc lớp Gold.

**Vì sao điều này lại ưu tiên tiên quyết**: Nếu công đoạn gắn thẻ Token của Presidio không thể duy trì nổi ≥11,600 record/giây trong môi trường Spark micro-batch thì cả dự án cấu trúc kỹ thuật kiến trúc sẽ thành số không (không hợp lệ). Khám phá ra việc đó vào trong Tuần 1 thì mất cỡ có 1 tuần; phát hiện ở Tuần 6 thì đi luôn 6 tuần.

**Các sản phẩm bàn giao ở cuối Tuần 1**:

1. **Kafka topic** tên `llm-events-dev` với trình sinh dữ liệu ảo (synthetic load generator) tạo lưu lượng tạo ra 10,000 sự kiện/giây (≈1% so với quy mô hoạt động production) với trọng tải JSON nặng 5 KB được gắn những hạt PII giả mạo (Fake name, Email, Số điện thoại từ Faker).

2. **Một Job Spark Structured Streaming** (ở trên Databricks hay hệ thống Spark local bản 3.5): Đọc thẳng từ `llm-events-dev`, chạy Presidio NER tokenization dạng nối chuỗi (inline) sử dụng tính năng UDF `mapInPandas`, in vào vùng chứa `delta_bronze_dev` đã phân vùng `date/hour` + lệnh gọi Z-order cho cụm `(tenant_id, model_id)`.

3. **Bài kiểm thử độ chuẩn xác của chức năng gắn thẻ tokenization**: Chạy truy vấn `SELECT prompt_tokenized FROM delta_bronze_dev LIMIT 1000` được điều vào quét qua regex dựa trên mẫu PII — Bắt buộc trả về 0 phản hồi. Chích ngầm dòng SSN cảnh báo là `123-45-6789` vào 1% sự kiện — phải xác nhận là mã đầu ra của dòng dữ liệu trên có dạng `<SSN_TOKEN_xxxxx>`.

4. **Bài test Thông lượng (Throughput Benchmark)**: đo test số lượng kết xuất thông qua chạy hàm UDF trên máy với hệ cụm Spark có 4 Executor. Yêu cầu đáp ứng ≥11,600 dữ kiện bản ghi/s (records/s). Nếu chậm hơn dự định: tìm và profiling cái lỗi chỗ nghẽn hệ thống (thường sẽ nằm ở phần giao tiếp JVM với Python), rồi cấu hình sang dạng giao tiếp API của Presidio Analyzer Server (REST) thay vì chạy UDF, chạy trên luồng gọi Spark HTTP để bắn request thẳng sang, hoặc dùng giải pháp chuyển dòng tiến trình dạng Vector như xử lý nhóm (batch) của mô hình spaCy.

5. **Bài smoke test cho các tính năng của Delta**:
   - `DESCRIBE HISTORY delta_bronze_dev` in ra có ≥10 xác nhận (commit) micro-batch cùng biến `operation = WRITE`.
   - Lệnh `SELECT COUNT(*) FROM delta_bronze_dev VERSION AS OF 0` hiện số là 0 (chứng nhận được là time travel sẽ đưa về hệ trắng trống đầu).
   - Lệnh `OPTIMIZE delta_bronze_dev ZORDER BY (tenant_id, model_id)` — kiểm chứng là bộ lượng file `numFiles` giảm từ ngưỡng xấp xỉ ~200 (micro-batch files) bay về ≤16.
   - Thêm vào bảng giá trị bằng một khóa `request_id` cụ thể, apply Deletion Vector để càn quét (delete), xác nhận (verify) kết quả truy xuất biến mất lúc SELECT sau lệnh Delete.

**Sẽ KHÔNG nằm ở trong Tuần 1**: Bước quy chuẩn chuyển đổi cho lớp Silver (Silver transformation), Các cụm dữ liệu phân nhánh bên Gold, Chạy Setup các bộ của Trino, hay cả chạy những danh sách cho Quyền Truy Cập (Unity Catalog ACLs), Cụm Production lớn Kafka, Theo dõi chi phí. Chúng nằm trong giai đoạn Tuần 2–4.

**Tiêu chí hoàn thành (Exit criteria) kết thúc quá trình Tuần 1**: Khả năng của quá trình Bronze là đáp ứng trơn tru 10,000 bản ghi/giây đạt ngưỡng rò rỉ (PII Leakage) của dự án bằng 0 với thời lượng duy trì chạy liền mạch trong 1 giờ không ngừng, đồng thời kết quả vượt được qua bài test Delta smoke test cả 5 bước. Nếu đáp ứng cả 5, việc kiến trúc này sẽ đạt mốc validate (xác thực hợp lệ) dưới góc độ cơ chế hoạt động.

---

## Phụ lục — Áp dụng Khái niệm Ngày 18

| Khái niệm | Vị trí áp dụng |
|---|---|
| Kiến trúc Medallion (Đồng/Bạc/Vàng) | Bố cục cốt lõi; mỗi lớp có lược đồ, lưu giữ và chính sách truy cập khác biệt |
| Các giao dịch ACID | Lệnh MERGE ở Gold có tính nguyên tử — FM-3 chứng minh không có hỏng hóc do ghi một phần |
| Tính năng du hành thời gian của Delta | Rollback FM-2 qua lệnh `RESTORE TABLE ... TO VERSION AS OF` |
| Z-order / Tối ưu hóa phân cụm (OPTIMIZE) | Đường dẫn nóng lớp Bronze/Silver cho các truy vấn lọc khách thuê; FM-4 cho thấy sự suy giảm và khắc phục |
| Vectơ xóa (Deletion Vectors) | Quyết định 7: quyền được xóa theo GDPR mà không cần viết lại phân vùng |
| Dữ liệu thay đổi luồng (Change Data Feed - CDF) | Job dữ liệu tuần tự lớp Bronze→Silver chỉ đọc những dữ liệu mới thêm kể từ checkpoint trước |
| Các Tầng vận hành tài chính (FinOps tiering) | Zstd trên lớp Bronze/Silver, LZ4 tại Gold; S3 Standard→IA→Glacier vòng đời cho bảng aggregate (cộng dồn) Gold ngày |
| Token hóa (Mã hóa) PII ở lớp Bronze | Quyết định 3: Presidio + FPE; sẽ không bao giờ có bất kỳ thông tin nào dưới định dạng thô có ở dưới lớp Silver/hạ tầng |
| Catalog & dòng dõi dữ liệu (lineage) | Unity Catalog: che giấu cột, an ninh trên dòng, biên bản kiểm toán (audit log), cấp dữ liệu thuộc cột |
| Ingestion dữ liệu luồng | Cụm dữ liệu chạy bằng Micro-batch theo công cụ Spark Structured Streaming; mốc xác định Watermark khoảng trễ độ 2h trên Silver |
