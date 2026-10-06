# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Dinh Cong Tu
**MSSV:** 2A202602479
**Cohort:** K4 (Level 3)
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Linux 6.8.0-1064-azure (x86_64)
- **CPU:** AMD EPYC 7763 64-Core Processor
- **Cores:** 1 physical / 2 logical (reported allocation)
- **CPU extensions:** AVX2; AVX-512 and NEON unavailable
- **RAM:** 7.8 GB
- **Accelerator:** CPU only
- **llama.cpp asset đã tải:** llama-b10488-bin-ubuntu-x64.tar.gz (b10488)
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** GitHub Codespace (Linux, 2 vCPU, ~7 GB RAM) — not Colab/Kaggle
The smaller model was selected because measured RAM is below 8 GB.

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

Hugging Face download failed with a read-only cache I/O error. The documented curl commands downloaded both GGUF files, and setup then succeeded. Pip disabled its unwritable cache and installed dependencies successfully. All 5 screenshots were rendered to PNG from captured terminal output because the Codespace has no GUI; they are not screen captures.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 4388 | 577 / 683 | 56.9 / 61.9 | 4178 / 4542 / 4542 | 17.6 |
| UD-Q2_K_XL | 0.39 | 4173 | 953 / 1141 | 60.6 / 62.3 | 4787 / 4954 / 4954 | 16.5 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

Q2 was 1.07× slower: 16.5 versus 17.6 tok/s, despite 0.39 versus 0.50 GB. It was not worth the latency trade-off here. Both answered the same question: Q4 wrongly blamed an external network; Q2 repeated itself and ended mid-sentence. Neither explained weight streaming correctly. One prompt cannot establish general quality.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.24 | 32000 | 51000 | 51000 | 6.7 | 0.0% |
| 50 | 0.25 | 39000 | 57000 | 57000 | 7.4 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.04×
- **P95 tăng:** 1.12×
- **Effective concurrency ở 50 users:** 7.4 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 4.00 / 4 slots (highest sampled average per decode step)

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Throughput barely increased (1.04×), while P95 increased 1.12×. Little’s Law, RPS × mean latency, gives occupancy 7.4, exceeding 4 active slots because it includes waiting requests. Continuous batching filled 4.00 slots; 46 deferred requests directly support queueing. Queue time is an inference, not a timing decomposition. First limit admission/queued work to protect goodput@SLO. Increasing slots risks CPU contention. Only 14 requests completed per run; unfinished requests and shared-host noise limit precision.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | No provisioning or IaC; local inference | stub |
| N17 Data pipeline | Hard-coded TOY_DOCS; no ingestion | stub |
| N18 Lakehouse | Python list; no persistent lakehouse | stub |
| N19 Vector + features | Keyword overlap; no embeddings or vector index | stub (RAG prompt and HTTP call real) |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 8881.7 ms
- **stage chiếm nhiều nhất:** llm (100% của total, rounded; mean total 8881.8 ms)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

LLM inference dominates at 8881.7 ms, or 100% rounded. This matches expectations: embeddings are skipped and keyword overlap only searches toy documents. Rounded 0.0 ms retrieval is not free; one query measured 0.1 ms. To target halving latency, reduce generated tokens and irrelevant context first, then measure again; optimising retrieval cannot materially help here.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Increase threads from the physical-core default `-t 1` to the best tested `-t 2`. This is the measured improvement; switching from Q4_K_M to UD-Q2_K_XL instead slowed decode.

```
before:  19.0 tok/s (-t 1, tg128)
after:   19.8 tok/s (-t 2, tg128)
speedup: 1.04×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Decode streams approximately the model’s weight bytes per token, so its bandwidth ceiling is roughly memory bandwidth divided by model size. The Q4 model is 0.50 GB; reducing transferred bytes can help bandwidth-bound decode, but dequantisation kernels and compute also matter. Q2’s 0.39 GB size did not help: it measured 16.5 versus 17.6 tok/s in the separate latency benchmark. That result is consistent with extra dequantisation cost outweighing bandwidth savings, without proving the cause.

The reported 1 physical core and 2 logical CPUs likely share execution units, caches and memory bandwidth. A second thread can hide stalls or improve utilisation, but also adds synchronisation and contention. The tg128 sweep’s 19.0 to 19.8 tok/s improvement contradicts the deck’s expected optimum at the physical-core count, modestly. Shared, virtualised host noise can affect this small gain. Only the tested thread counts establish the best observed setting; the sweep does not locate a wider saturation point. These tg128 values should not be compared directly with the separate request benchmark as a controlled change.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** _<B1 build-compare / B2 sweep nào / B4 challenge nào / B5 lựa chọn nào>_

**Numbers:**

```
before:  <số>
after:   <số>
speedup: <X.Y>×
```

**Điều này nói lên gì mà deck chưa nói:**

_(để trống nếu bạn không làm phần này)_

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

_(1–2 câu. Không bắt buộc, nhưng grader đọc hết.)_

_(để trống nếu bạn không làm phần này)_

---

## 8. Self-check trước khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [x] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [x] `benchmarks/02-server-results.md` committed (`make load-report`)
- [x] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [x] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [x] 5 screenshots trong `submission/screenshots/`
- [x] `make verify` → **exit 0**
- [x] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [x] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Claude agents were used for planning/review, and Codex agents for running commands and drafting text. All 5 screenshots were rendered to PNG from captured terminal output (no GUI in the Codespace), not taken as screen captures. Measurements come from the generated local reports and captures; AI assistance does not replace those measurements.
