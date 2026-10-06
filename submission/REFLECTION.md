# Reflection — Day 20 Lab (Báo cáo cá nhân)

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

> Từ `make probe`.

- **OS:** Linux 6.8.0-1064-azure (x86_64)
- **CPU:** AMD EPYC 7763 64-Core Processor
- **Cores:** 1 physical / 2 logical (phần được cấp cho Codespace)
- **CPU extensions:** có AVX2; không có AVX-512 và NEON
- **RAM:** 7.8 GB
- **Accelerator:** chỉ CPU, không GPU
- **llama.cpp asset đã tải:** llama-b10488-bin-ubuntu-x64.tar.gz (b10488)
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** GitHub Codespace (Linux, 2 vCPU, ~7 GB RAM) — **không** dùng Colab/Kaggle.
Chọn model nhỏ vì RAM đo được dưới 8 GB.

**Setup story** (≤ 80 chữ):

Tải model qua Hugging Face bị lỗi I/O do thư mục cache chỉ đọc, nên tôi dùng lệnh curl
trong tài liệu để tải hai file GGUF, sau đó `make setup` chạy thành công. Pip tự tắt cache
không ghi được và vẫn cài đủ thư viện. Codespace không có giao diện đồ hoạ nên 5 screenshot
được dựng thành ảnh PNG từ output terminal thật đã ghi lại, không phải chụp màn hình.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Bảng từ `benchmarks/01-quickstart-results.md` (`make bench`).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 4388 | 577 / 683 | 56.9 / 61.9 | 4178 / 4542 / 4542 | 17.6 |
| UD-Q2_K_XL | 0.39 | 4173 | 953 / 1141 | 60.6 / 62.3 | 4787 / 4954 / 4954 | 16.5 |

**Quan sát** (≤ 60 chữ):

2-bit không nhanh hơn mà **chậm hơn 1.07×** (16.5 so với 17.6 tok/s, TTFT 953 so với
577 ms) dù nhỏ hơn 0.11 GB — không đáng trên máy này. Hỏi cùng một câu: Q4 giải thích sai
(đổ cho "mạng bên ngoài"), Q2 lặp ý và bị cắt giữa câu. Một câu hỏi chưa đủ kết luận chất lượng.

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
chạy): 4.00 / 4 slots (giá trị trung bình cao nhất lấy mẫu được trên mỗi bước decode)

**Saturation reading** (≤ 80 chữ):

Server đã chịu áp lực từ 10 users và bão hoà rõ ở 50: tải tăng 5× nhưng RPS chỉ tăng
1.04×, P95 tăng 1.12×. Theo Little's Law, concurrency 7.4 vượt 4 slot, nên phần dư là request đang chờ.
4/4 slot luôn bận và có 46 request bị hoãn, cho thấy latency thêm là queue time chứ không
phải compute. Để nâng goodput@SLO, tôi giới hạn hàng đợi (admission) trước, vì thêm slot
trên 2 vCPU chỉ gây tranh chấp CPU. Mỗi lượt chỉ 14 request hoàn thành nên số liệu mang tính tham khảo.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Cái nào real, cái nào stub.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Không provisioning/IaC; inference chạy local | stub |
| N17 Data pipeline | Tài liệu hard-code trong TOY_DOCS; không có ingestion | stub |
| N18 Lakehouse | List Python; không có lakehouse lưu trữ | stub |
| N19 Vector + features | So khớp từ khoá; không có embedding hay vector index | stub (dựng prompt RAG và gọi HTTP là real) |
| N20 Serving | `llama-server` | real |

**Latency split** (trung bình 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 8881.7 ms
- **stage chiếm nhiều nhất:** llm (~100% tổng thời gian; tổng trung bình 8881.8 ms)

**Reflection** (≤ 60 chữ):

Bottleneck là LLM (8881.7 ms, ~100%), đúng kỳ vọng vì bỏ qua embedding và retrieve chỉ
so khớp từ khoá trên vài tài liệu (0.0 ms là do làm tròn, một query đo 0.1 ms). Muốn giảm
2×, tôi giảm số token sinh ra và cắt context không liên quan; tối ưu retrieve không giúp gì.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

**Change:** Tăng số thread từ mặc định theo số core vật lý `-t 1` lên `-t 2` (mức tốt nhất
trong sweep). Đây là cải thiện đo được; còn đổi Q4_K_M sang UD-Q2_K_XL thì lại làm decode chậm đi.

```
before:  19.0 tok/s (-t 1, tg128)
after:   19.8 tok/s (-t 2, tg128)
speedup: 1.04×
```

**Tại sao nó work:**

Decode sinh từng token và mỗi token phải đọc gần như toàn bộ trọng số model từ bộ nhớ, nên
trần tốc độ xấp xỉ băng thông bộ nhớ chia cho kích thước model (0.50 GB với Q4). Giảm số
byte có thể giúp khi decode bị chặn bởi băng thông, nhưng kernel giải nén (dequantization)
và phần tính toán cũng tốn thời gian. Q2 nhỏ hơn (0.39 GB) nhưng chậm hơn (16.5 so với
17.6 tok/s), phù hợp với giả thuyết chi phí giải nén 2-bit lớn hơn phần băng thông tiết kiệm
được trên một máy chỉ có một core vật lý — dù phép đo này không tách riêng được nguyên nhân.

Máy báo 1 core vật lý và 2 logical CPU, tức hai luồng hyper-thread dùng chung đơn vị thực thi,
cache và băng thông bộ nhớ. Luồng thứ hai không thêm sức tính toán thật, nhưng có thể chạy
trong lúc luồng kia đang chờ dữ liệu từ bộ nhớ, nhờ đó tận dụng core tốt hơn — đổi lại có thêm
chi phí đồng bộ và tranh chấp, nên lợi ích chỉ nhỏ (1.04×). Kết quả này **khác kỳ vọng của
deck** (tối ưu ở đúng số core vật lý), nhưng chỉ khác ít. Host ảo dùng chung có nhiễu, nên mức
lợi nhỏ này cần hiểu thận trọng; sweep chỉ có hai điểm nên không xác định được điểm bão hoà
rộng hơn. Số tg128 không nên so trực tiếp với benchmark request ở §2 vì đó là hai phép đo khác nhau.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

**Đã làm:** B2 — `make sweep-batch` (sweep `-b` / `-ub` cho prefill, prompt 512 token,
`-t 2`), kết quả ở `benchmarks/bonus-batch-size-sweep.md`; B3 — before/after dưới đây.

**Numbers:**

```
before:  42.0 tok/s (pp512, -b 128 -ub 128)
after:   50.9 tok/s (pp512, -b 1024 -ub 512)
speedup: 1.21×
```

**Điều này nói lên gì mà deck chưa nói:**

Prefill khác decode: nó xử lý cả prompt một lúc nên bị chặn bởi tính toán (nhân ma trận),
và hiệu quả phụ thuộc mỗi lần đọc trọng số được dùng cho bao nhiêu token. Khi `-b` nhỏ hơn
prompt, prefill bị chia thành 2–4 chunk, mỗi chunk đọc lại ~0.5 GB trọng số và chịu thêm
overhead đồng bộ thread. Bằng chứng: `-b 512 -ub 256` (50.6) ≈ `-b 512 -ub 512` (50.5),
còn `-b 256 -ub 256` chỉ 41.3 — trên máy này yếu tố quyết định là logical batch có chứa
trọn prompt hay không, chứ micro-batch gần như không ảnh hưởng. Từ 512 trở lên đường cong
phẳng. Sweep chỉ đo throughput của một request; batch lớn có thể làm request khác chờ lâu
hơn, nên cần đo lại P95 dưới `make load-50` trước khi dùng cho production.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Tôi ngạc nhiên nhất khi bản 2-bit lại chậm hơn bản 4-bit: tôi vốn nghĩ ít bit hơn thì luôn
nhanh hơn, nhưng trên máy chỉ có một core vật lý, chi phí giải nén lớn hơn lợi ích băng thông.
Điều thứ hai là server đã gần bão hoà ngay từ 10 users — với 2 vCPU, thêm người dùng chỉ làm hàng đợi dài ra.

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

Các tác nhân Claude được sử dụng để hỗ trợ lập kế hoạch và rà soát, trong khi các tác nhân Codex hỗ trợ thực thi lệnh và soạn thảo. Các ảnh PNG được tạo từ dữ liệu đầu ra của terminal do môi trường Codespace không có giao diện đồ họa (GUI), nhằm thể hiện trực tiếp quá trình tôi thao tác, chạy lệnh và kiểm tra kết quả trong quá trình phát triển. AI chỉ đóng vai trò hỗ trợ trong một số bước, không thay thế việc thực hiện, kiểm thử hay đánh giá của tôi. Các số liệu đo lường được lấy trực tiếp từ báo cáo cục bộ và dữ liệu ghi lại từ hệ thống.
