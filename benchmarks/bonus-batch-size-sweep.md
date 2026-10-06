# Bonus - Batch-size sweep (chunked prefill)

Host `Linux-x86_64` · llama.cpp `b10488` ·
`threads=2` `ngl=0` · metric `pp512`

| -b (logical) | -ub (micro) | pp512 (tok/s) | vs best |
|:--|--:|--:|--:|
| 128 | 128 | 42.0 | 83% |
| 256 | 256 | 41.3 | 81% |
| 512 | 256 | 50.6 | 100% |
| 512 | 512 | 50.5 | 99% |
| 1024 | 512 | 50.9 | 100% |
| 2048 | 512 | 50.4 | 99% |

Best: `-b 1024 -ub 512` at 50.9 tok/s
(1.23x the slowest point tested).

This sweep only measures the throughput half of the trade. The cost it hides is
TTFT for queued requests: a larger micro-batch holds the device longer per step,
so anything waiting behind it waits longer. To see both halves, re-run
`make load-50` with your best and worst settings via
`.venv/bin/python labs/02-serve/serve.py -- -b N -ub M` and compare P95.

## Your finding

Với prompt 512 token, điểm chậm nhất là `-b 256 -ub 256` (41.3 tok/s), `-b 128 -ub 128`
đạt 42.0 tok/s, còn mọi cấu hình có `-b ≥ 512` đều nằm trong khoảng 50.4–50.9 tok/s
(tốt nhất `-b 1024 -ub 512`, 1.23× so với điểm chậm nhất, 1.21× so với `-b 128`). Điểm
đáng chú ý: `-b 512 -ub 256` (50.6) gần bằng `-b 512 -ub 512` (50.5), trong khi
`-b 256 -ub 256` chỉ 41.3 — tức là trên máy này, thứ quyết định là **logical batch `-b`
có chứa trọn prompt trong một lượt hay không**, còn micro-batch 256 hay 512 gần như
không khác. Khi `-b < 512`, prefill bị chia thành 2–4 chunk; mỗi chunk phải đọc lại toàn
bộ ~0.5 GB trọng số và chịu thêm overhead dựng graph + đồng bộ thread, nên mỗi byte
trọng số được tái sử dụng cho ít token hơn. Từ 512 trở lên, prompt đã vừa một lượt nên
tăng thêm không còn lợi (đường cong phẳng). Chênh lệch 41.3 vs 42.0 giữa 256 và 128 nhỏ
hơn nhiễu của host ảo dùng chung (mỗi điểm chỉ 2 lần lặp), nên tôi không diễn giải nó.

Production: tôi chọn `-b 512 -ub 256` — đạt ~100% throughput prefill nhưng giữ
micro-batch nhỏ hơn để mỗi bước chiếm CPU ngắn hơn. Để chắc nó không làm hại P95 của
server đang tranh chấp, cần chạy lại `make load-50` với cấu hình tốt nhất và tệ nhất
(`serve.py -- -b N -ub M`) rồi so P95 và TTFT dưới tải, vì sweep này chỉ đo một request
đơn lẻ, không đo thời gian request khác phải chờ phía sau một bước prefill dài.
