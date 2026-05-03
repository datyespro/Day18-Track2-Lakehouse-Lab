# Day 18 — Lakehouse Lab (Track 2)

Lab cho **AICB-P2T2 · Ngày 18 · Data Lakehouse Architecture**.
Build Bronze → Silver → Gold pipeline với Delta Lake.

**Hai paths để chọn:**

| Path | Stack | Setup | RAM | Khi nào dùng |
|---|---|---|---|---|
| **Lightweight (default)** | `deltalake` + DuckDB + Polars | `make setup` (~10 s) | ~500 MB | Hầu hết học viên — laptop yếu, mạng chậm, muốn focus vào concept |
| **Spark (Docker)** | PySpark + delta-spark + MinIO | `make spark-up` (~3 min) | ~4 GB | Học viên muốn trải nghiệm Spark API y hệt production Databricks |

> Cả hai paths viết ra **cùng một Delta Lake on-disk format** — bạn có thể đổi
> giữa hai paths bất cứ lúc nào, các tables vẫn đọc được.

---

## Quick Start — Lightweight (recommended)

```bash
git clone https://github.com/VinUni-AI20k/Day18-Track2-Lakehouse-Lab.git
cd Day18-Track2-Lakehouse-Lab
make setup    # ~10 s with pip, ~2 s with uv
make smoke    # ~5 s — verifies the stack works
make lab      # opens http://localhost:8888
```

Yêu cầu: **Python ≥ 3.10**. Không cần Docker, không cần Java, không cần MinIO.

Khi `make smoke` báo `All checks passed`, mở
**http://localhost:8888/lab/tree/01_delta_basics.ipynb** và bắt đầu.

Generate sample data cho NB4:
```bash
make data    # 200K rows → _lakehouse/bronze/llm_calls_raw/
```

### Tất cả lệnh `make`

```
make setup     Lightweight: tạo venv + install (80 MB)
make smoke     Lightweight: 5-second smoke test
make lab       Lightweight: open Jupyter Lab
make data      Lightweight: generate Bronze sample
make clean     Lightweight: wipe venv + _lakehouse/

make spark-up      Spark/Docker: start full stack
make spark-smoke   Spark/Docker: smoke test
make spark-data    Spark/Docker: generate 1M-row sample
make spark-down    Spark/Docker: stop (data persists)
make spark-clean   Spark/Docker: full reset
```

---

## Quick Start — Spark/Docker (optional)

```bash
make spark-up && make spark-smoke
```

Yêu cầu: Docker Desktop ≥ 4.x, RAM ≥ 8 GB free.
Endpoints + troubleshooting cho path này: xem [`notebooks-spark/README.md`](notebooks-spark/) (notebooks dùng PySpark API).

---

## Cấu trúc & tiến trình (cả hai paths)

| Notebook | Skill | Slide-5 deliverable bullet | Pass when… |
|---|---|---|---|
| `01_delta_basics` | Write/read Delta, schema enforcement, transaction log | NB1 — `_delta_log/` JSON visible + `schema_mode="merge"` evolution | bad-write blocked + `tier` column added on opt-in evolve |
| `02_optimize_zorder` | Small-file problem; OPTIMIZE + Z-order benchmark | NB2 — speedup ≥ 3× **or** files-pruned ≥ 10× (min/max stats) | notebook prints both metrics; either ≥ target |
| `03_time_travel` | versionAsOf, RESTORE, MERGE, `history()` | NB3 — MERGE 100K + RESTORE; `history()` ≥ 5 versions (kể cả RESTORE) | final history dump shows v0…v4 |
| `04_medallion` | LLM-observability Bronze→Silver→Gold pipeline | NB4 — dedup observable + Gold p50/p95/cost qua ≥ 7 ngày | Silver < Bronze rows; Gold has ≥ 7 distinct dates × 3 models |

**Source format:** Notebooks live as Jupytext `.py` files (small, easy to review).
`make setup` and `make lab` auto-convert to `.ipynb`. Edit `.ipynb` in Jupyter
and Jupytext keeps both in sync.

**Spark API equivalent:** Each lightweight notebook has a comment showing the
PySpark equivalent at the top, so you can mentally map between the two paths.

---

## Deliverable (4 notebook đã chạy + ảnh chụp)

Mapping 1-to-1 với slide-5 deliverable bullets:

1. **NB1** — Delta table created; `_delta_log/00..0.json` visible; bad-schema
   write blocked; `schema_mode="merge"` adds the `tier` column.
2. **NB2** — `OPTIMIZE+Z-ORDER` gives **speedup ≥ 3× OR files-pruned ratio ≥ 10×**
   (notebook prints both — screenshot whichever passes).
3. **NB3** — `history()` ≥ 5 versions **including the RESTORE row** (the
   notebook prints history *after* `restore()` — that's the screenshot to take);
   MERGE 100K succeeds; RESTORE < 30 s and removes `score < 0` rows.
4. **NB4** — Bronze + Silver + Gold all present on disk; **Silver < Bronze**
   (dedup observable); Gold spans **≥ 7 dates × 3 models** with populated
   p50/p95 latency, cost_usd, and error_rate.

Chấm điểm: xem [`rubric.md`](rubric.md). Tổng 100 pts → Track-2 Daily Lab (30%).

---

## Bonus Challenge — Design Your Own Lakehouse (optional, ungraded)

A separate, open-ended **architecture brief**: pick a hard real-world data
problem (LLM observability at 1B req/day, Decree-13-compliant CDC pipeline,
trillion-token training corpus, multimodal RAG, FinOps-capped tiering, catalog
migration, feature-store lineage — or your own), and design the storage
strategy you'd defend in a design review.

**Document is the deliverable**; code is optional. Submissions get a written
instructor review focused on *judgment*: do your decisions show explicit
rejected alternatives with reasons? Are your numbers realistic? Did you apply
Day 18 concepts (medallion, ACID, time travel, catalogs, lineage, security,
FinOps)?

It does not affect the core grade — it's there for students who want to push
past the rote deliverable and build a portfolio piece. Full brief,
recommended topics, and self-checklist:
[`BONUS-CHALLENGE.md`](BONUS-CHALLENGE.md) (tiếng Việt) ·
[`BONUS-CHALLENGE-EN.md`](BONUS-CHALLENGE-EN.md) (English).

---

## Cấu trúc repo

```
.
├── Makefile              # both paths
├── README.md             # bạn đang đọc
├── BONUS-CHALLENGE.md    # optional architecture brief (tiếng Việt)
├── BONUS-CHALLENGE-EN.md # optional architecture brief (English)
├── requirements.txt      # lightweight (deltalake + duckdb + polars)
├── requirements-spark.txt# Spark path
├── rubric.md             # grading
├── notebooks/            # ← lightweight path (default)
│   ├── 01_delta_basics.py
│   ├── 02_optimize_zorder.py
│   ├── 03_time_travel.py
│   └── 04_medallion.py
├── notebooks-spark/      # Spark/Docker path (same lessons, PySpark API)
├── scripts/
│   ├── lakehouse.py            # path helper (lightweight)
│   ├── generate_data_lite.py   # lightweight Bronze generator
│   ├── verify_lite.py          # lightweight smoke test
│   ├── spark_session.py        # Spark factory
│   ├── generate_data.py        # Spark Bronze generator
│   └── verify.py               # Spark smoke test
└── docker/
    └── docker-compose.yml      # Spark/MinIO/Jupyter stack
```

---

## Troubleshooting (lightweight)

| Triệu chứng | Fix |
|---|---|
| `make setup` báo `python3: command not found` | Install Python 3.10+ (https://www.python.org/downloads/) |
| `make lab` báo "port 8888 in use" | Đổi: `$(JUPYTER) lab --port 8889` trong Makefile |
| NB2 speedup < 3× | Bình thường nếu RAM < 4 GB — DuckDB cache làm before/after gần nhau. Reset bằng `make clean && make setup`. |
| NB4 lỗi "Path does not exist" | Quên `make data` |

---

## Submission

Fork repo → push 4 notebook đã chạy + `submission/REFLECTION.md` (≤ 200 words: anti-pattern nào trong slide §5 team bạn dễ vướng nhất, vì sao?). PR back vào upstream với title `[NXX] Lab18 — <Họ Tên>`.

---

© VinUniversity AICB program. Phỏng theo Track 2 Day 18 slide.
