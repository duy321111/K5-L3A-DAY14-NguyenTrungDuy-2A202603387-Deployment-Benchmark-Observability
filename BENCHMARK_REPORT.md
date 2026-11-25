# Benchmark & Observability Report — Day 14

**Họ tên:** Nguyễn Trung Duy — MSSV: 2A202603387  
**Ngày chạy:** 2026-11-25  
**Môi trường:** Windows 11 + WSL2 (Ubuntu, Python 3.11) + Docker Desktop 29.9.3

---

## 1. SLO của agent

| SLI | SLO | Lý do chọn ngưỡng này |
| :--- | :--- | :--- |
| p95 latency `/ask` | ≤ 500 ms (local) / ≤ 1000 ms (production stack) | 95% người dùng nhận phản hồi mượt (< 1 giây); đặt theo p95 thay vì average để không che giấu tail latency. |
| Error rate `/ask` | ≤ 2.0% (local) / ≤ 5.0% (alert production) | Đủ biên độ chịu lỗi upstream LLM ngẫu nhiên (mặc định ~1%) nhưng vẫn phát hiện nhanh khi sự cố tăng đột biến. |
| Accuracy trên golden dataset | ≥ 85% (CI quality gate), mục tiêu 100% | Đảm bảo chất lượng câu trả lời không bị suy giảm (0 regression) trước mỗi lần merge/deploy. |

---

## 2. Benchmark hiệu năng (develop, `load_test.py`)

### 2.1 Concurrency vs latency (agent v2, 300 requests/mức)

| Concurrency | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Error rate | SLO (p95 ≤ 500ms, err ≤ 2%) |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| 1 | 3.48 | 214.7 | 490.8 | 1729.6 | 0.33% (1/300) | ✅ PASS |
| 10 | 34.21 | 223.4 | 302.8 | 1727.6 | 2.00% (6/300) | ✅ PASS |
| 50 | 9.09 | 3889.9 | 13240.3 | 19524.3 | 0.33% (1/300) | ❌ FAIL (p95) |
| 100 | 8.56 | 7269.1 | 26005.1 | 31588.2 | 1.00% (3/300) | ❌ FAIL (p95) |

**Nhận xét:** RPS **không** tăng tuyến tính theo concurrency. Từ `c=1` lên `c=10`, RPS tăng gần 10 lần (`3.48 → 34.21 req/s`) trong khi p50 gần như không đổi (`214.7 → 223.4 ms`) vì FastAPI async tận dụng tốt thời gian chờ I/O. Tuy nhiên khi tăng lên `c=50` và `c=100`, 1 process uvicorn duy nhất bị bão hòa hàng đợi/event loop và CPU tranh chấp trên cùng máy chạy `load_test.py`, khiến latency xếp hàng tăng vọt (p95 lên 13.2s và 26.0s) còn RPS tụt xuống ~8.5–9 req/s.

### 2.2 Tail latency (`concurrency=20`, 300 requests)

| `MOCK_SLOW_RATE` | avg (ms) | p95 (ms) | p99 (ms) |
| :--- | :---: | :---: | :---: |
| 0 | 334.6 | 721.5 | 818.6 |
| 0.03 (mặc định) | 353.0 | 692.9 | 1762.8 |
| 0.1 | 414.3 | 1741.6 | 1897.5 |

**Nhận xét:** **p99 và p95** là hai chỉ số nhạy nhất với request chậm bất thường: khi tăng `MOCK_SLOW_RATE` từ `0` → `0.03` → `0.1`, `avg` chỉ nhích nhẹ (`334.6 → 353.0 → 414.3 ms`), trong khi `p99` nhảy vọt từ `818.6 ms` lên `1762.8 ms` (ngay ở mức 3%) và `p95` nhảy vọt lên `1741.6 ms` (ở mức 10%). Vì vậy SLO **không nên đặt theo average** — average làm mờ các ngoại lệ chậm và che giấu trải nghiệm tệ của 1–5% người dùng.

---

## 3. Benchmark chất lượng (develop, `eval/run_eval.py`)

| Chỉ số | Agent v1 | Agent v2 | Agent v2 (sau khi sửa KB) |
| :--- | :---: | :---: | :---: |
| Accuracy | 4.3% (1/23) | 91.3% (21/23) | **100.0% (23/23)** |
| Latency p50 / p95 (ms) | 187.3 / 257.7 | 228.8 / 1636.3 | 197.6 / 232.3 |
| Avg tokens / câu | 57.0 | 109.1 | 113.7 |
| Tổng cost (USD) | \$0.000403 | \$0.000762 | \$0.000790 |
| Cost / câu đúng (USD) | \$0.00040275 | \$0.00003627 | **\$0.00003436** |
| Regression | — | 0 | 0 |
| Quality gate | ❌ FAIL | ✅ PASS (≥80%) | **✅ PASS (≥100%)** |

**Những câu v2 trả lời sai ban đầu và cách sửa** (chỉ sửa `KNOWLEDGE_BASE` trong `develop/utils/mock_llm.py` và đồng bộ sang `production/utils/mock_llm.py`, không sửa `golden_dataset.jsonl`):

| ID | Keyword bị thiếu | Đã sửa gì |
| :--- | :--- | :--- |
| `security-02` | `token`, `hết hạn`, `chữ ký` | Thêm mục tài liệu về **JWT vs API key** vào `KNOWLEDGE_BASE` (từ khóa `["jwt"]`), giải thích JWT là token có chữ ký số, chứa thông tin user và thời điểm hết hạn. |
| `scaling-03` | `cpu`, `memory`, `custom metrics` | Thêm mục tài liệu về **Kubernetes HPA** vào `KNOWLEDGE_BASE` (từ khóa `["hpa", "kubernetes", "autoscal"]`), nêu rõ HPA scale số pod theo CPU, memory hoặc custom metrics. |

**CI Quality Gate (Exercise 7.3):** Đã tạo workflow [.github/workflows/eval.yml](.github/workflows/eval.yml) chạy trên mỗi `pull_request` vào `main`. Script `eval/run_eval.py --min-accuracy 0.85` trả về **exit code `1`** nếu accuracy < 85% hoặc có regression, khiến GitHub Actions job fail và tự động chặn merge PR.

**Nhận xét trade-off giữa v1 và v2:** Agent v2 thêm bước retrieval (RAG) nên prompt dài hơn gần gấp đôi (`57 → 113.7` tokens/câu) và tổng chi phí tăng gần 2 lần (`$0.000403 → $0.000790`), nhưng accuracy tăng vượt bậc từ `4.3%` lên `100%`. Nhờ đó **"cost / câu đúng"** của v2 rẻ hơn tới **~11.7 lần** (`$0.00003436` so với `$0.00040275`), chứng minh "cost / câu đúng" phản ánh giá trị thực tốt hơn nhiều so với "cost / câu".

---

## 4. Observability stack (production)

### 4.1 Screenshot & Trả lời Exercise 7.4
- [x] Dashboard Grafana khi đang có tải: [screenshots/grafana-dashboard.png](screenshots/grafana-dashboard.png) *(thấy rõ Golden Signals: Traffic lên ~50–60 req/s, Latency p95 = 1.58s, LLM cost 1h = \$0.4059, biểu đồ p50/p95/p99 và Requests/s theo status code).*
- [x] Trace chậm trong Jaeger: [screenshots/jaeger-slow-trace.png](screenshots/jaeger-slow-trace.png) *(trace chậm nhất `observable-agent: POST /ask` đạt **1.7s** với 4 spans).*
- [x] Trace lỗi trong Jaeger: [screenshots/jaeger-error-trace.png](screenshots/jaeger-error-trace.png) *(trace `9207c8df9ced39380d7a4c1da8c57e10`, `POST /ask 503`, tổng 228ms, span lỗi `chat mock-llm-small` = 113ms).*
- [x] Trang Alerts của Prometheus với **2 alert cùng `FIRING`**: [screenshots/prometheus-alert-firing.png](screenshots/prometheus-alert-firing.png) (`AgentHighErrorRate` và `AgentInstanceDown` đều `FIRING`).

**Trả lời nhanh câu hỏi Exercise 7.4:**
1. Khoảng cách lớn giữa `p95` (~1.58s) và `avg` (~400–600ms) cho thấy có đuôi latency dài (tail latency do `MOCK_SLOW_RATE=0.03` cộng thêm ~1500ms).
2. Nginx chia đều traffic giữa các instance theo round-robin nhờ `server agent:8000 resolve;`.
3. `input tokens/s` lớn hơn `output tokens/s` vì mỗi prompt của v2 bao gồm `SYSTEM_PROMPT` + tài liệu RAG truy xuất được + câu hỏi, dài hơn nhiều so với câu trả lời ngắn gọn.
4. `curl http://localhost:8080/metrics` trả về **`404 Not Found`** vì trong `nginx.conf` có `location = /metrics { return 404; }` để chặn public truy cập endpoint nội bộ (chỉ cho Prometheus scrape trực tiếp trong mạng `obs_net`), tránh lộ thông tin hệ thống.

### 4.2 Thí nghiệm "phá hệ thống" (Exercise 7.6)

| # | Thay đổi | Alert nào firing | Thời gian từ lúc đổi → firing | Panel nào phản ánh rõ nhất |
| :---: | :--- | :--- | :---: | :--- |
| A | `MOCK_ERROR_RATE=0.2` (chạy tải Locust 30 users) | **`AgentHighErrorRate`** (error rate > 5%, `for: 1m`) | ~60–75 giây | Error rate (%) & Requests/s theo status code (503 tăng vọt) |
| D | `docker pause k5-day14-observability-agent-2` | **`AgentInstanceDown`** (`up{job="agent"} == 0`, `for: 30s`) | ~30–40 giây | Instances up (giảm từ 2 → 1) |
| E | `AGENT_VERSION=v1` (phân tích lý thuyết) | Không alert nào firing | Không firing | Không có panel hạ tầng nào báo đỏ (thậm chí latency & token còn giảm!) |

**Thí nghiệm E không làm alert nào firing — cách phát hiện trong production:**
Các alert hạ tầng (error rate, latency, instance down, cost) đều bình thường khi đổi nhầm sang `v1` vì HTTP vẫn trả `200 OK` và còn nhanh/rẻ hơn. Để phát hiện sự cố suy giảm chất lượng im lặng (silent quality degradation), cần kết hợp: (1) chạy **eval quality gate trên golden dataset trong CI/CD** trước khi deploy để chặn regression; (2) **online evaluation** (lấy mẫu một tỉ lệ traffic thật chấm bằng LLM-as-a-judge); (3) theo dõi tín hiệu người dùng (thumbs up/down, tỉ lệ hỏi lại) và metric nghiệp vụ (`retrieval_hit_rate` — khi `v1` chạy, `retrieved=false` sẽ rơi về 0% ngay lập tức).

### 4.3 Debug bằng trace (Exercise 7.5)
- **Trace lỗi trong ảnh [jaeger-error-trace.png](screenshots/jaeger-error-trace.png):**
  - `trace_id`: `9207c8df9ced39380d7a4c1da8c57e10`
  - Tổng thời gian: **228 ms** (`POST /ask` → HTTP `503`)
  - Span breakdown: `agent.run` (181 ms) gồm `agent.retrieve_context` (66.8 ms) và **`chat mock-llm-small` (113 ms — chiếm 49.6% tổng thời gian, phát sinh lỗi `503 Service Unavailable`)**.
- **Trace chậm nhất trong ảnh [jaeger-slow-trace.png](screenshots/jaeger-slow-trace.png):**
  - Tổng thời gian: **1.7 s** (4 spans; phần lớn thời gian nằm ở span gọi LLM `chat mock-llm-small` do nhánh tail latency `+1500ms`).
- **Dòng log JSON thực tế (lấy từ `docker compose -p k5-day14-observability logs agent | grep llm_error`) có gắn `trace_id` và `span_id`:**

```json
{"ts": "2026-11-25T13:52:46.967781+00:00", "level": "INFO", "service": "observable-agent", "instance": "d614fc69b5a9", "event": "llm_error", "trace_id": "a3be2a8a8e8b1e62a4d5a8c9d6175c06", "span_id": "f64685d82a9ba474", "request_id": "a6b9bc9846e6", "error": "Upstream LLM error: 503 Service Unavailable (mock)"}
```

---

## 5. Benchmark scaling (production, Locust headless, 100 users, 2 phút)

Số liệu khớp trực tiếp với `benchmark-results/scale_1_stats.csv` và `benchmark-results/scale_3_stats.csv`:

| Số instance | Scope | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Failures |
| :---: | :--- | :---: | :---: | :---: | :---: | :---: |
| **1** | `Aggregated` (`POST /ask`) | **56.22** (`51.53`) | **200** (`200`) | **1700** (`1700`) | **5000** (`5000`) | **62 / 6739** (0.92%) |
| **3** | `Aggregated` (`POST /ask`) | **49.13** (`44.55`) | **210** (`210`) | **2700** (`2800`) | **6700** (`6700`) | **47 / 5789** (0.81%) |

**Giải thích kết quả:** Khi scale từ 1 lên 3 instance trên cùng một máy laptop (WSL2), **p95 không giảm mà tăng từ 1700ms lên 2700ms** (RPS giảm nhẹ từ 56.22 → 49.13). Nguyên nhân:
1. Tác vụ của agent chủ yếu là **I/O-bound** (`asyncio.sleep` giả lập LLM + retrieval), 1 instance async vốn đã xử lý được ~56 RPS mà chưa nghẽn event loop.
2. Cả 3 container `agent` (mỗi container giới hạn `0.5 CPU`), Nginx, Prometheus, Grafana, Jaeger và Locust cùng chạy trên **1 máy host duy nhất** → tăng lên 3 container làm tăng context switching, tranh chấp CPU/I/O và thêm overhead gửi OpenTelemetry spans + scrape metrics, khiến nút thắt chuyển sang **tài nguyên máy host**. Trên hạ tầng cloud phân tán (các node riêng biệt), horizontal scaling mới thực sự nhân thông lượng và giảm p95 khi CPU bão hòa.

---

## 6. Kết luận

- **3 điều học được về benchmark & observability:**
  1. **Đặt SLO theo percentile (p95/p99), không theo average:** average che giấu tail latency; chỉ cần 3% request chậm là p99 đã tăng gấp đôi.
  2. **Đo chất lượng song song hiệu năng:** một agent nhanh và rẻ (v1) nhưng trả lời sai 95.7% vẫn vô dụng; cần golden dataset + CI quality gate để chặn regression tự động.
  3. **3 trụ cột Observability bổ trợ nhau:** Metrics + Alerts báo *khi nào* hệ thống có vấn đề, Traces (Jaeger) chỉ rõ *chậm/lỗi ở span nào*, và Structured Logs gắn `trace_id` cho biết *ngữ cảnh chi tiết* để sửa lỗi nhanh.
- **Nếu đưa agent này lên production thật, sẽ thêm:**
  - Metric `retrieval_hit_rate` và cảnh báo khi tỉ lệ tìm thấy tài liệu giảm đột ngột (bắt lỗi im lặng như thí nghiệm E).
  - Metric `llm_tokens_per_request` (Histogram) + cảnh báo khi độ dài prompt tăng bất thường (phòng prompt injection / tốn chi phí).
  - Alert cho tỉ lệ `429 Rate Limit` và `402 Budget Exceeded` tăng đột biến để phát hiện sớm hành vi lạm dụng hoặc cần nâng quota.
