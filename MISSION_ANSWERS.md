# Day 14 Lab — Mission Answers

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns phát hiện trong `01-localhost-vs-production/develop/app.py`
1. **Hardcode secrets trong source code** (dòng 17–18): `OPENAI_API_KEY = "sk-hardcoded-fake-key-never-do-this"` và `DATABASE_URL = "postgresql://admin:password123@localhost:5432/mydb"`. Khi push lên Git sẽ lộ ngay lập tức.
2. **Không có config management / hardcode cấu hình** (dòng 21–22): `DEBUG = True`, `MAX_TOKENS = 500` viết cứng trong code, không đổi được theo môi trường (dev/staging/prod).
3. **Dùng `print()` thay cho structured logging và log cả secret** (dòng 33–38): `print(f"[DEBUG] Using key: {OPENAI_API_KEY}")` vừa khó thu thập/tìm kiếm trên log aggregator, vừa làm rò rỉ API key ra log.
4. **Không có health check / readiness endpoint** (thiếu `/health` và `/ready`): nền tảng cloud hoặc load balancer không biết container còn sống hay đã sẵn sàng nhận traffic để tự restart / route traffic.
5. **Bind `host="localhost"`, hardcode `port=8000`, bật `reload=True`** (dòng 49–53): `localhost` khiến bên ngoài container không truy cập được; `port=8000` cố định không nhận biến `PORT` do Railway/Render cấp; `reload=True` tốn tài nguyên và không an toàn trên production.
6. **Không có graceful shutdown**: khi nhận `SIGTERM`, process bị ngắt đột ngột, dễ cắt ngang request đang xử lý và không đóng kết nối tài nguyên sạch sẽ.

### Exercise 1.2: Kết quả chạy bản develop
Chạy `01-localhost-vs-production/develop/app.py` và gọi `curl -sS -X POST "http://localhost:8000/ask?question=Hello"`:
```json
{"answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận."}
```

### Exercise 1.3: So sánh Develop vs Production
*(Ví dụ cụ thể sửa anti-pattern: `develop/app.py` dòng 17 hardcode `OPENAI_API_KEY` và dòng 51–53 hardcode `host="localhost", port=8000, reload=True`; còn `production/app.py` dòng 24 đọc từ `config.settings` (biến môi trường) và dòng 194–198 bind `host=settings.host` (`0.0.0.0`), `port=settings.port`, `reload=settings.debug`.)*

| Feature | Develop (`develop/app.py`) | Production (`production/app.py`) | Tại sao quan trọng? |
| :--- | :--- | :--- | :--- |
| **Config & Secrets** | Hardcode `OPENAI_API_KEY`, `DATABASE_URL`, `DEBUG=True` trong code | Đọc từ biến môi trường qua `config.py` (`settings`) và `.env` (không commit) | Tránh lộ secret trên Git; triển khai cùng 1 build cho nhiều môi trường (12-Factor). |
| **Network & Port** | `host="localhost"`, `port=8000`, `reload=True` | `host="0.0.0.0"`, `port=settings.port` (từ `PORT`), `reload=settings.debug` | Container/cloud cần lắng nghe trên `0.0.0.0` và cổng động `PORT` do platform cấp. |
| **Health Check** | Không có | Có `GET /health` (liveness), `GET /ready` (readiness), `GET /metrics` | Platform tự động restart container bị treo và chỉ route traffic vào instance đã sẵn sàng. |
| **Logging** | `print()` plain text, còn in cả API key | `logging` định dạng JSON có cấu trúc (`time`, `level`, `msg`), không log secret | Dễ parse/tìm kiếm/tạo alert trên hệ thống tập trung (Loki, Datadog...) và bảo mật. |
| **Shutdown** | Tắt đột ngột | `lifespan` + `signal.SIGTERM` handler, đặt `is_ready = False` trước khi tắt | Tránh cắt ngang request đang xử lý (zero-downtime deploy) và đóng kết nối sạch sẽ. |
| **CORS & Input** | Không cấu hình CORS, nhận `question` qua query | Cấu hình `CORSMiddleware`, nhận JSON body và validate trả `422` khi thiếu `question` | Kiểm soát nguồn gọi API và xử lý đầu vào chặt chẽ, nhất quán. |

---

## Part 2: Docker

### Exercise 2.1: Trả lời 4 câu hỏi về `02-docker/develop/Dockerfile`
1. **Base image:** `python:3.11` (bản Debian đầy đủ, dung lượng gốc lớn ~1 GB).
2. **Working directory:** `/app` (`WORKDIR /app`).
3. **Tại sao `COPY requirements.txt` đứng trước `COPY app.py` (hay `COPY . .`)?** Để tận dụng **Docker layer cache**: mỗi lệnh trong Dockerfile tạo một layer. File `requirements.txt` ít thay đổi hơn source code, nên khi sửa code chỉ cần rebuild từ bước `COPY` code trở đi mà không phải chạy lại `pip install`, tiết kiệm rất nhiều thời gian build.
4. **`CMD` vs `ENTRYPOINT` khác nhau thế nào?** `ENTRYPOINT` định nghĩa tiến trình chính cố định của container (khó bị ghi đè trừ khi dùng `--entrypoint`), còn `CMD` cung cấp lệnh mặc định hoặc tham số mặc định cho `ENTRYPOINT` và dễ dàng bị ghi đè khi truyền thêm tham số vào lệnh `docker run`.

### Exercise 2.2: Test container develop
Build từ gốc repo (`docker build -f 02-docker/develop/Dockerfile -t my-agent:develop .`), chạy container cổng 8000 và gọi `curl -sS -X POST "http://localhost:8000/ask?question=What+is+Docker"`:
```json
{"answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!"}
```

### Exercise 2.3: So sánh kích thước image (đo thật bằng `docker images`)
```text
my-agent:advanced   93b235d75d35   240MB    (57.4MB compressed)
my-agent:develop    aebcac9e17ff   1.67GB   (426MB compressed)
```
- **Develop (`my-agent:develop`):** `1.67 GB` (~1670 MB)
- **Production (`my-agent:advanced`):** `240 MB`
- **Chênh lệch:** Giảm **1430 MB**, tức nhỏ hơn **~85.6%** (`(1670 - 240) / 1670 = 85.6%`).
- **Giải thích multi-stage:** Stage 1 (`builder` từ `python:3.11-slim`) cài thêm công cụ biên dịch (`gcc`, `libpq-dev`) và chạy `pip install --user`. Stage 2 (`runtime` từ `python:3.11-slim`) chỉ `COPY --from=builder` các package đã cài sang `/home/appuser/.local` và chạy bằng user non-root `appuser` — bỏ toàn bộ trình biên dịch và build cache nên image rất gọn và bảo mật hơn.

### Exercise 2.4: Docker Compose stack (`02-docker/production/docker-compose.yml`)
**Kiến trúc các service** (giao tiếp qua bridge network `internal`, chỉ mở cổng 80/443 ở Nginx):
```text
Client ──(HTTP :80)──▶ Nginx (reverse proxy / LB)
                           │
                           ▼ (internal network :8000)
                         Agent (FastAPI)
                        /               \
                       ▼                 ▼
              Redis (:6379)        Qdrant (:6333)
           (cache/rate limit)      (vector DB)
            [vol: redis_data]    [vol: qdrant_data]
```
**Kết quả test qua Nginx (`http://localhost`):**
- `GET http://localhost/health`:
  ```json
  {"status":"ok","uptime_seconds":3.5,"version":"2.0.0","timestamp":"2026-11-25T10:25:00.671343"}
  ```
- `POST http://localhost/ask` với `{"question": "Explain microservices"}`:
  ```json
  {"answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận."}
  ```

---

## Part 3: Cloud Deployment

### Exercise 3.2: Deploy Agent Part 6 lên Render
- **Public URL:** https://day14-ai-agent-ug77.onrender.com
- **Platform:** Render (Web Service runtime Docker, Root Directory `06-lab-complete`, gắn kèm Render Key Value / Redis Free tier).
- **Tự kiểm 4 lệnh CP6:** `GET /health` → `200`, `GET /ready` → `200`, `POST /ask` (không key) → `401`, `POST /ask` (có `X-API-Key`) → `200`.
- **Screenshots:** [dashboard.png](screenshots/dashboard.png), [running.png](screenshots/running.png), [test.png](screenshots/test.png). Chi tiết xem [DEPLOYMENT.md](DEPLOYMENT.md).

### So sánh `railway.toml` với `render.yaml`
| Tiêu chí | `railway.toml` (Railway) | `render.yaml` (Render Blueprint) |
| :--- | :--- | :--- |
| **Định dạng & phạm vi** | File **TOML** cấu hình cho **1 service** (`[build]`, `[deploy]`). | File **YAML** khai báo **Infrastructure-as-Code** cho nhiều service/database cùng lúc dưới `services:`. |
| **Cách build & healthcheck** | `builder = "DOCKERFILE"`, `dockerfilePath = "Dockerfile"`, `healthcheckPath = "/health"`. | `type: web`, `runtime: docker`, `dockerfilePath: ./Dockerfile`, `healthCheckPath: /health`. |
| **Quản lý biến môi trường & secret** | Biến môi trường và secret thường set trực tiếp trên Dashboard/CLI (`railway variables set`), không khai báo danh sách biến trong `railway.toml`. | Khai báo trực tiếp danh sách `envVars` trong `render.yaml`; secret dùng `sync: false` (nhập trên Dashboard) hoặc `generateValue: true` (Render tự sinh) để không lộ trong Git. |
| **Lý do khác nhau** | Railway hướng tới trải nghiệm tối giản theo từng service, kết nối nhanh qua biến tham chiếu `${{Redis.REDIS_URL}}`. | Render Blueprint hướng tới tái tạo toàn bộ kiến trúc nhiều service từ 1 file khai báo duy nhất trên Git. |

### Exercise 3.3: Mô tả CI/CD pipeline GCP Cloud Run (`cloudbuild.yaml` & `service.yaml`)
Pipeline trong `cloudbuild.yaml` tự động kích hoạt khi push code lên nhánh `main` và đi qua **4 bước tuần tự**: (1) `test` — dùng `python:3.11-slim` cài dependencies và chạy `pytest`; (2) `build` — build Docker image với layer cache (`--cache-from`) và gắn 2 tag (`$COMMIT_SHA` và `latest`); (3) `push` — đẩy toàn bộ tag lên Google Container Registry (`gcr.io/$PROJECT_ID/ai-agent`); (4) `deploy` — dùng `gcloud run deploy` triển khai image mới lên Cloud Run (`asia-southeast1`) với secrets lấy từ Secret Manager (`--set-secrets`). File `service.yaml` định nghĩa cấu hình chạy dạng IaC (Knative): autoscaling từ `minScale: 1` (tránh cold start) đến `maxScale: 10`, `containerConcurrency: 80`, giới hạn `1 CPU / 512Mi RAM`, nạp secret qua `secretKeyRef`, và khai báo `livenessProbe` (`/health`) cùng `startupProbe` (`/ready`).

---

## Part 4: API Security

### Exercise 4.1: API Key Authentication (`04-api-gateway/develop`)
Chạy `AGENT_API_KEY=secret-key-123 python app.py`:

- **Không có key → `401 Unauthorized`:**
```http
HTTP/1.1 401 Unauthorized
date: Wed, 25 Nov 2026 09:58:11 GMT
server: uvicorn
content-length: 67
content-type: application/json

{"detail":"Missing API key. Include header: X-API-Key: <your-key>"}
```

- **Có key hợp lệ (`X-API-Key: secret-key-123`) → `200 OK`:**
```http
HTTP/1.1 200 OK
date: Wed, 25 Nov 2026 09:58:11 GMT
server: uvicorn
content-length: 122
content-type: application/json

{"question":"Hello","answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận."}
```

### Exercise 4.2 & 4.3: JWT Authentication + Rate Limiting (`04-api-gateway/production`)
Lấy JWT token tại `POST /auth/token` với tài khoản `student` / `demo123` (token đã được che: `Bearer <REDACTED_JWT>`), gọi lặp 12 lần vào `POST /ask`:
```text
Req 1:  HTTP/1.1 200 OK
Req 2:  HTTP/1.1 200 OK
Req 3:  HTTP/1.1 200 OK
Req 4:  HTTP/1.1 200 OK
Req 5:  HTTP/1.1 200 OK
Req 6:  HTTP/1.1 200 OK
Req 7:  HTTP/1.1 200 OK
Req 8:  HTTP/1.1 200 OK
Req 9:  HTTP/1.1 200 OK
Req 10: HTTP/1.1 200 OK
Req 11: HTTP/1.1 429 Too Many Requests
Req 12: HTTP/1.1 429 Too Many Requests
```
**Chi tiết response `429 Too Many Requests`:**
```http
HTTP/1.1 429 Too Many Requests
date: Wed, 25 Nov 2026 09:58:18 GMT
server: uvicorn
x-ratelimit-limit: 10
x-ratelimit-remaining: 0
x-ratelimit-reset: 1795600759
retry-after: 59
content-length: 98
content-type: application/json
x-content-type-options: nosniff
x-frame-options: DENY
x-xss-protection: 1; mode=block
referrer-policy: strict-origin-when-cross-origin

{"detail":{"error":"Rate limit exceeded","limit":10,"window_seconds":60,"retry_after_seconds":59}}
```
- **Phân tích `rate_limiter.py`:** Dùng thuật toán **Sliding Window Counter** lưu timestamp các request trong `deque` theo từng `user_id` trong cửa sổ 60 giây. Limit mặc định là **10 req/phút** cho user thường (`rate_limiter_user`) và **100 req/phút** cho admin (`rate_limiter_admin`). Để admin có giới hạn cao hơn hoặc bypass, app kiểm tra role từ payload JWT rồi chọn limiter tương ứng (hoặc bỏ qua check nếu `role == "admin"`).

### Exercise 4.4: Cost Guard
- **Cách thiết kế Cost Guard:**
  - Tính chi phí mỗi request dựa trên số `input_tokens` và `output_tokens` nhân với đơn giá từng loại token (ví dụ GPT-4o-mini: \$0.00015 / 1K input, \$0.0006 / 1K output).
  - Lưu tổng chi phí đã dùng theo ngày/tháng trong **Redis** (key dạng `cost:daily:<YYYY-MM-DD>` và `cost:monthly:<YYYY-MM>`, đặt TTL tự hết hạn) bằng `INCRBYFLOAT` để nhiều worker/instance dùng chung và không mất khi restart.
  - Trước khi gọi LLM gọi `check_budget()`: nếu chi phí hiện tại ≥ ngưỡng ngân sách thì chặn ngay bằng **HTTP `402 Payment Required`** (và có thể ghi log cảnh báo khi đạt 80% budget). Sau khi gọi LLM xong mới gọi `record_cost()` để cộng dồn chi phí thật.
- **Tại sao cần cả Rate Limit lẫn Cost Guard?** Rate limit bảo vệ **tốc độ gọi** (chống spam/DDoS, tránh quá tải server), nhưng một user vẫn có thể gửi rất ít request với prompt siêu dài làm tốn tiền LLM lớn. Cost guard bảo vệ trực tiếp **ngân sách (tiền)** theo tổng chi phí token, đảm bảo không bao giờ bị "cháy bill" ngoài kiểm soát.

---

## Part 5: Scaling & Reliability

### Exercise 5.1–5.3: Health Checks, Graceful Shutdown & Stateless Design
- **Liveness (`/health`) vs Readiness (`/ready`):**
  - `/health` (liveness) trả lời *"process còn sống hay đã treo?"* — luôn trả `200` miễn tiến trình còn hoạt động, kể cả khi Redis/DB tạm thời mất kết nối (tránh việc orchestrator restart vòng lặp vô ích khi lỗi nằm ở Redis).
  - `/ready` (readiness) trả lời *"instance đã sẵn sàng nhận traffic chưa?"* — kiểm tra kết nối Redis (`ping()`); nếu lỗi thì trả `503` để load balancer tạm ngừng gửi request vào instance đó cho đến khi sẵn sàng.
- **Graceful shutdown:** Khi nhận `SIGTERM` (lúc deploy phiên bản mới hoặc scale down), server ngừng nhận kết nối mới, đợi các request đang xử lý (in-flight) hoàn thành trả về `200`, đóng kết nối Redis/DB trong `lifespan`, rồi thoát êm với exit code `0`.
- **Vì sao state trong RAM bị hỏng khi scale nhiều instance?** Mỗi container/worker có vùng nhớ RAM riêng biệt. Khi chạy 3 instance sau Nginx load balancer, request 1 có thể vào `instance A` (lưu history vào RAM của A), nhưng request 2 lại bị Nginx đẩy sang `instance B` — lúc này B không thấy lịch sử hội thoại của user. Giải pháp là thiết kế **stateless**, chuyển toàn bộ session/history/rate-limit ra **Redis** dùng chung.

### Exercise 5.4 & 5.5: Load Balancing (3 instances) & Test Stateless
Chạy `docker compose -p k5-day14-scaling up -d --build --scale agent=3` trong `05-scaling-reliability/production/` (cổng 8080, endpoint `POST /chat`) và chạy `python3 test_stateless.py`:
```text
============================================================
Stateless Scaling Demo
============================================================

Session ID: d3fd226e-6990-4c17-a081-9c50060b718e

Request 1: [instance-1d9f12]
  Q: What is Docker?
  A: Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!...

Request 2: [instance-fd6ddf]
  Q: Why do we need containers?
  A: Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ O...

Request 3: [instance-62ba99]
  Q: What is Kubernetes?
  A: Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ O...

Request 4: [instance-1d9f12]
  Q: How does load balancing work?
  A: Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ O...

Request 5: [instance-fd6ddf]
  Q: What is Redis used for?
  A: Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé....

------------------------------------------------------------
Total requests: 5
Instances used: {'instance-fd6ddf', 'instance-1d9f12', 'instance-62ba99'}
✅ All requests served despite different instances!

--- Conversation History ---
Total messages: 10
✅ Session history preserved across all instances via Redis!
```
**Giải thích kết quả:** 5 request trong cùng một `session_id` (`d3fd226e-...`) được Nginx phân tán lần lượt qua cả **3 instance khác nhau** (`instance-1d9f12`, `instance-fd6ddf`, `instance-62ba99`), nhưng khi đọc lại `/chat/{session_id}/history` vẫn còn nguyên đủ **10 tin nhắn** (5 câu hỏi + 5 câu trả lời) vì toàn bộ lịch sử được lưu tập trung trong Redis.

---

## Part 6: Final Project — Kết quả `check_production_ready.py`

Chạy đầy đủ (có Docker, trên WSL2 Linux) trong `06-lab-complete/`:

```text
================================================================
  📋 Checklist Deliverable
     ✅ Dockerfile (multi-stage, < 500 MB)
     ✅ docker-compose.yml (agent + redis)
     ✅ .dockerignore
     ✅ Health check endpoint (GET /health)
     ✅ Readiness endpoint (GET /ready)
     ✅ API Key authentication
     ✅ Rate limiting
     ✅ Cost guard
     ✅ Config từ environment variables
     ✅ Structured logging
     ✅ Graceful shutdown
     ✅ Public URL ready (Railway / Render config)
----------------------------------------------------------------
  Result: 12/12 tiêu chí đạt · 43/43 checks passed
  🎉 PASS 12/12 — PRODUCTION READY! Deploy nào!
================================================================
```

---

## AI Log

1. **Thiết lập môi trường WSL2 + Python 3.11 + Docker Desktop:**
   - **Hỏi AI:** Cách cài WSL2 trên Windows, khắc phục lỗi `pydantic-core` trên Python 3.14, lỗi quyền `docker.sock` và lỗi Docker Desktop WSL2.
   - **Nhận được:** Hướng dẫn dùng `uv` tạo virtualenv Python 3.11 (`~/.venv-day14`), thêm user vào group `docker`, và reset WSL2 khi Docker Desktop treo.
   - **Tự thực hiện & kiểm tra:** Tự cài Ubuntu trên WSL2, bật WSL Integration trên Docker Desktop, kiểm tra `os.name == posix` và `docker run --rm hello-world` thành công.
2. **Hoàn thiện Part 6 (`06-lab-complete/`):**
   - **Hỏi AI:** Hỗ trợ cấu trúc và triển khai các chỗ `TODO` trong `app/` (`config.py`, `redis_client.py`, `auth.py`, `rate_limiter.py`, `cost_guard.py`, `main.py`), `Dockerfile` multi-stage, `docker-compose.yml`, `.dockerignore`, `.env.example`, `railway.toml`, `render.yaml`.
   - **Nhận được:** Code hoàn chỉnh tuân theo hợp đồng của `06-lab-complete/README.md`.
   - **Tự thực hiện & kiểm tra:** Tự cài `requirements.txt` trong WSL2, chạy đầy đủ `python check_production_ready.py` (không sửa script chấm) và xác nhận đạt `PASS 12/12` (`43/43 checks passed`, image 238 MB, graceful shutdown 2.4s).
3. **Deploy Cloud (CP6) và tổng hợp Part 1–5 (`MISSION_ANSWERS.md`, `DEPLOYMENT.md`):**
   - **Hỏi AI:** Hướng dẫn deploy `06-lab-complete/` lên Render + Redis (do Railway hết hạn Trial) và cách chạy các lệnh đo đạc thật cho Part 1, 2, 4, 5.
   - **Nhận được:** Các bước cấu hình Render, mẫu `DEPLOYMENT.md` và bộ lệnh test.
   - **Tự thực hiện & kiểm tra:** Tự tạo Redis + Web Service trên Render, đặt biến môi trường trên dashboard (không commit secret), tự chạy 4 lệnh `curl` xác nhận `200 / 200 / 401 / 200`, tự chụp 3 ảnh bằng chứng, và tự chạy toàn bộ lệnh đo kích thước image (`1.67GB` vs `240MB`), test `401/200/429` và `test_stateless.py` trên 3 instance để lấy số liệu thật ghi vào báo cáo.
4. **Benchmark & Observability (Part 7 — CP7 & CP8):**
   - **Hỏi AI:** Hướng dẫn thêm metric `agent_question_length_chars`, bổ sung 2 mục thiếu vào `KNOWLEDGE_BASE`, sửa lỗi `PYTHONUSERBASE` trong `production/Dockerfile` gây 502 khi chạy Locust, và cách kích hoạt 2 alert `FIRING` trong Prometheus.
   - **Nhận được:** Đoạn code bổ sung metric/KB, bản vá 2 dòng `production/Dockerfile`, workflow `.github/workflows/eval.yml`, và khung `BENCHMARK_REPORT.md`.
   - **Tự thực hiện & kiểm tra:** Tự chạy toàn bộ 7 bài `load_test.py`, 3 bài `run_eval.py` (đạt `23/23` — `100%`, 0 regression), 2 bài Locust headless (`scale_1` & `scale_3`), tự chụp 4 ảnh Grafana/Jaeger/Prometheus, và đối chiếu toàn bộ số liệu trong `BENCHMARK_REPORT.md` khớp với các file JSON/CSV đã nộp.
