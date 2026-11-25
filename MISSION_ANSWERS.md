# Day 14 Lab - Mission Answers

> **Họ tên:** Nguyễn Tiến Đạt  
> **MSSV:** 2A202603452  
> **Lớp:** L3A  

---

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found in `01-localhost-vs-production/develop/app.py`
1. **Hardcoded secrets (dòng 17–18):** `OPENAI_API_KEY = "sk-hardcoded-fake-key-never-do-this"` và `DATABASE_URL = "postgresql://admin:password123@localhost:5432/mydb"` ghi trực tiếp trong source code, dễ bị lộ khi push lên Git.
2. **Không quản lý cấu hình qua biến môi trường (dòng 21–22):** `DEBUG = True`, `MAX_TOKENS = 500` được gán cứng thay vì đọc từ `os.getenv(...)` / `config.py`.
3. **Dùng `print()` và log lộ secret (dòng 33–34, 38):** Dùng `print(f"[DEBUG] Using key: {OPENAI_API_KEY}")` thay vì structured JSON logging, làm lộ API key vào stdout/log aggregator.
4. **Thiếu health check endpoint:** Không có `/health` (liveness) hay `/ready` (readiness) để platform (Docker, Railway, Render, Kubernetes) kiểm tra trạng thái và tự restart khi app treo.
5. **Cấu hình server cứng cho localhost (dòng 49–54):** `host="localhost"`, `port=8000`, `reload=True`. Trên container/cloud cần bind `0.0.0.0`, đọc `PORT` từ biến môi trường và tắt `reload`.

### Exercise 1.2: Kết quả chạy bản develop
Lệnh gọi:
```bash
curl -X POST "http://localhost:8000/ask?question=Hello"
```
Kết quả trả về:
```json
{"answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận."}
```

### Exercise 1.3: Comparison table (`develop/app.py` vs `production/app.py`)
| Feature | Develop (`develop/app.py`) | Production (`production/app.py`) | Why Important? |
|---|---|---|---|
| **Config** | Hardcode `OPENAI_API_KEY`, `DATABASE_URL`, `DEBUG`, `port=8000` | Đọc qua `config.py` (`settings`) từ biến môi trường / `.env` | Tách biệt cấu hình theo môi trường (dev/staging/prod), tránh lộ secret trên Git (12-Factor App). |
| **Health check** | Không có `/health` hay `/ready` | Có `GET /health` (liveness) và `GET /ready` (readiness) | Cho phép orchestrator/load balancer biết container còn sống và đã sẵn sàng nhận traffic hay chưa. |
| **Logging** | `print("[DEBUG] ...")`, log cả `OPENAI_API_KEY` | `logging` với format JSON, không log secrets | Log JSON dễ parse/lọc trên Datadog, Loki, CloudWatch; bảo mật thông tin nhạy cảm. |
| **Shutdown** | Tắt đột ngột, không dọn dẹp | Dùng `lifespan` của FastAPI: set `is_ready = False`, chờ request đang chạy hoàn tất | Tránh cắt ngang request của người dùng khi deploy phiên bản mới (zero-downtime rolling deploy). |
| **Network binding** | `host="localhost"`, `reload=True` | `host="0.0.0.0"`, `port=settings.port`, tắt `reload` | Container chỉ nhận traffic từ bên ngoài nếu bind `0.0.0.0` và dùng `PORT` do cloud cấp. |

---

## Part 2: Docker

### Exercise 2.1: Dockerfile questions (`02-docker/develop/Dockerfile`)
1. **Base image:** `python:3.11` (bản đầy đủ của Python 3.11 trên Debian, dung lượng lớn ~1 GB+).
2. **Working directory:** `/app` (`WORKDIR /app`).
3. **Tại sao `COPY requirements.txt` trước `COPY app.py`:** Tận dụng **Docker layer cache**. `requirements.txt` ít thay đổi hơn source code; nếu copy và `pip install` trước, các lần rebuild khi chỉ sửa `app.py` sẽ dùng lại layer cache mà không phải cài lại toàn bộ thư viện.
4. **`CMD` vs `ENTRYPOINT`:**
   - `CMD` định nghĩa lệnh/tham số mặc định khi container chạy và **dễ dàng bị ghi đè** khi truyền tham số ở cuối `docker run`.
   - `ENTRYPOINT` định nghĩa tiến trình chính cố định của container; tham số truyền thêm ở `docker run` (hoặc `CMD`) sẽ được nối vào làm đối số cho `ENTRYPOINT`.

### Exercise 2.2: Build và test container `my-agent:develop`
Lệnh build từ gốc repo:
```bash
docker build -f 02-docker/develop/Dockerfile -t my-agent:develop .
docker run -d --rm -p 8000:8000 my-agent:develop
curl -X POST "http://localhost:8000/ask?question=What%20is%20Docker?"
```
Kết quả:
```json
{"answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!"}
```

### Exercise 2.3: Multi-stage build & Image size comparison
- **Stage 1 (`builder`):** Dùng `python:3.11-slim`, cài `gcc` và `libpq-dev` để compile các package Python, cài dependencies vào `/root/.local` bằng `pip install --user`.
- **Stage 2 (`runtime`):** Dùng `python:3.11-slim` sạch, tạo non-root user `appuser`, chỉ `COPY --from=builder /root/.local /home/appuser/.local` cùng source code và `HEALTHCHECK`. Không mang theo `gcc`, `libpq-dev` hay apt cache nên image nhỏ gọn và an toàn hơn.

Kích thước đo được bằng `docker images my-agent`:
- **Develop (`my-agent:develop`):** `1.67 GB` (`1670 MB`)
- **Production (`my-agent:advanced`):** `240 MB`
- **Difference:** Giảm khoảng **`85.6%`** dung lượng (`(1670 - 240) / 1670`).

### Exercise 2.4: Docker Compose stack (`02-docker/production/docker-compose.yml`)
Kiến trúc gồm 4 services nằm chung trong mạng `internal`:
```text
Client ──▶ [nginx :80/:443] ──▶ [agent :8000] ──▶ [redis :6379]
                                      │
                                      └──▶ [qdrant :6333]
```
- `nginx`: Reverse proxy & load balancer, mở cổng `80`/`443` ra ngoài và chuyển tiếp request vào `agent` trong mạng `internal`.
- `agent`: FastAPI AI Agent (`target: runtime`), `depends_on` `redis` và `qdrant` (với `condition: service_healthy`).
- `redis`: `redis:7-alpine`, lưu cache session và rate limiting (`redis_data` volume).
- `qdrant`: `qdrant/qdrant:v1.9.0`, vector database cho RAG (`qdrant_data` volume).

---

## Part 3: Cloud Deployment

### Exercise 3.2: Render Deployment
- **Public URL:** https://k5-l3a-day14-nguyentiendat-2a202603452.onrender.com
- **Platform:** Render (Web Service chạy Docker từ `06-lab-complete/` + Render Key Value / Redis)
- **Kiểm tra tự động:** `/health` → `200`, `/ready` → `200`, `POST /ask` (không key) → `401`, `POST /ask` (có `X-API-Key`) → `200`.
- **Screenshots:** [dashboard.png](screenshots/dashboard.png), [running.png](screenshots/running.png), [test.png](screenshots/test.png).

### So sánh `railway.toml` và `render.yaml`
| Tiêu chí | `railway.toml` | `render.yaml` |
|---|---|---|
| **Định dạng & Mục đích** | TOML ngắn gọn, cấu hình cho **1 service** hiện tại trong thư mục (`[build]`, `[deploy]`). | YAML dạng **Blueprint (IaC)**, có thể khai báo **nhiều services** (`web`, `redis`...) cùng lúc trong `services:`. |
| **Cách chỉ định build** | `builder = "DOCKERFILE"` trong `[build]`. | `runtime: docker` trong từng service. |
| **Health check** | `healthcheckPath = "/health"` trong `[deploy]`. | `healthCheckPath: /health` trong service. |
| **Quản lý biến môi trường & secret** | Thường set riêng qua Railway Dashboard hoặc `railway variables set` (ít khai báo trực tiếp trong `railway.toml`). | Hỗ trợ khai báo trực tiếp trong `envVars:` và xử lý secret an toàn bằng `generateValue: true` (tự sinh key) hoặc `sync: false` (nhập tay trên dashboard, không lộ vào Git). |

### Exercise 3.3: GCP Cloud Run CI/CD Pipeline (`03-cloud-deployment/production-cloud-run/`)
Khi push code lên nhánh `main`, Google Cloud Build tự động chạy pipeline 4 bước theo `cloudbuild.yaml`:
1. **`test`:** Dùng container `python:3.11-slim`, cài `requirements.txt` và `pytest`, chạy `pytest tests/ -v` để đảm bảo test pass trước khi build.
2. **`build`:** Build Docker image với tag theo `$COMMIT_SHA` và `latest` (tận dụng `--cache-from` để tăng tốc).
3. **`push`:** Đẩy các image tag lên Google Container Registry (`gcr.io/$PROJECT_ID/ai-agent`).
4. **`deploy`:** Gọi `gcloud run deploy` triển khai image `$COMMIT_SHA` lên Cloud Run (`asia-southeast1`), cấu hình `--min-instances=1` (tránh cold start), `--max-instances=10`, memory `512Mi`, và lấy secret an toàn từ **Secret Manager** (`--set-secrets`). File `service.yaml` định nghĩa cấu hình Knative Service tương ứng (tối đa 80 concurrent requests/instance, `livenessProbe` `/health` và `startupProbe` `/ready`).

---

## Part 4: API Security

### Exercise 4.1: API Key Authentication (`04-api-gateway/develop`)
Chạy server với `AGENT_API_KEY=secret-key-123 python app.py`:

- **Không có key (`401 Unauthorized`):**
```http
HTTP/1.1 401 Unauthorized
content-type: application/json

{"detail":"Missing API key. Include header: X-API-Key: <your-key>"}
```
- **Có key hợp lệ (`200 OK`):**
```http
HTTP/1.1 200 OK
content-type: application/json

{"question":"Hello","answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic."}
```

### Exercise 4.2 & 4.3: JWT Authentication & Rate Limiting (`04-api-gateway/production`)
1. Lấy JWT token qua `POST /auth/token` với tài khoản `student` / `demo123` (giới hạn 10 request/phút).
2. Gọi `POST /ask` với `Authorization: Bearer <TOKEN_DA_CHE>`:
```http
HTTP/1.1 200 OK
content-type: application/json
x-content-type-options: nosniff
x-frame-options: DENY
x-xss-protection: 1; mode=block
referrer-policy: strict-origin-when-cross-origin

{"question":"Explain JWT","answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic.","usage":{"requests_remaining":9,"budget_remaining_usd":2.1e-05}}
```
3. Vòng lặp 12 request liên tiếp (`rate_limiter.py` dùng thuật toán **Sliding Window Counter** lưu timestamps trong `deque`, giới hạn `10 req/60s` cho `user` và `100 req/60s` cho `admin`):
   - Request 1 → 9: `HTTP 200` (cộng với 1 request test trước đó = đủ 10 requests trong window)
   - Request 10 → 12: `HTTP 429 Too Many Requests`:
```http
HTTP/1.1 429 Too Many Requests
x-ratelimit-limit: 10
x-ratelimit-remaining: 0
x-ratelimit-reset: 1795596092
retry-after: 59
content-type: application/json

{"detail":{"error":"Rate limit exceeded","limit":10,"window_seconds":60,"retry_after_seconds":59}}
```

### Exercise 4.4: Cost Guard (`04-api-gateway/production/cost_guard.py`)
- **Cách hoạt động:** Tính chi phí dựa trên số `input_tokens` và `output_tokens` theo đơn giá trên 1K tokens (`PRICE_PER_1K_INPUT_TOKENS`, `PRICE_PER_1K_OUTPUT_TOKENS`).
- Trước khi gọi LLM, `check_budget()` kiểm tra:
  - Tổng chi phí toàn hệ thống (`global_daily_budget_usd`) → vượt trả `503`.
  - Chi phí theo ngày của user (`daily_budget_usd`) → vượt trả **`402 Payment Required`**.
  - Đạt ngưỡng cảnh báo (`warn_at_pct = 0.8`, tức 80%) → ghi log `WARNING`.
- Sau khi gọi LLM xong, `record_usage()` cộng dồn token và chi phí. Trong production (Part 6), state chi phí được lưu vào **Redis** (`INCRBYFLOAT`) thay vì dict trong RAM để đồng bộ giữa nhiều worker/instance.
- **Tại sao cần cả Rate Limit và Cost Guard:** Rate Limit bảo vệ **tốc độ** (chống spam/DDoS theo số request/phút), còn Cost Guard bảo vệ **ngân sách tiền** (vì 1 request dài vẫn có thể tốn nhiều token dù chưa chạm rate limit).

---

## Part 5: Scaling & Reliability

### Exercise 5.1–5.3: Health Checks, Graceful Shutdown & Stateless Design
- **Liveness (`/health`) vs Readiness (`/ready`):**
  - `GET /health`: Kiểm tra process còn sống hay không, luôn trả `200` khi app chạy (kể cả khi Redis tạm mất kết nối, tránh orchestrator restart nhầm vòng lặp).
  - `GET /ready`: Kiểm tra app đã sẵn sàng nhận traffic chưa (ping Redis thành công → `200`, mất kết nối Redis → `503`). Load balancer sẽ tạm ngưng gửi request tới instance chưa ready.
- **Graceful shutdown:** Khi nhận `SIGTERM`, app ngừng nhận request mới, đợi các request đang xử lý hoàn thành rồi mới thoát (`exit code 0`), tránh lỗi 502/cắt ngang response.
- **Stateless design:** Không lưu session/history/rate limit/cost trong RAM của từng instance (vì khi scale nhiều instance, mỗi instance có RAM riêng). Mọi state dùng chung được đưa vào **Redis**.

### Exercise 5.4–5.5: Load Balancing & Test Stateless (`05-scaling-reliability/production/`)
Chạy `docker compose -p k5-day14-scaling up -d --build --scale agent=3` (mở cổng `8080`, endpoint `POST /chat`) và chạy `python test_stateless.py`:

```text
Session ID: d9f5a621-ba01-447c-a345-df1abace8c93

Request 1: [instance-8ab3a3]
Request 2: [instance-200d26]
Request 3: [instance-8ab3a3]
Request 4: [instance-2b93df]
Request 5: [instance-200d26]

Total requests: 5
Instances used: {'instance-8ab3a3', 'instance-2b93df', 'instance-200d26'}
✅ All requests served despite different instances!

--- Conversation History ---
Total messages: 10
✅ Session history preserved across all instances via Redis!
```
**Giải thích:** Nginx phân phối 5 request trong cùng một `session_id` lần lượt tới 3 container khác nhau (`instance-8ab3a3`, `instance-200d26`, `instance-2b93df`). Nhờ toàn bộ lịch sử hội thoại được lưu tập trung trong **Redis**, mọi instance đều đọc và ghi được đầy đủ 10 message mà không bị mất dữ liệu khi request chuyển sang instance khác.

---

## Part 6: Final Project (`06-lab-complete/`)

Kết quả chạy `python check_production_ready.py` (chế độ đầy đủ với Docker & Redis trong Ubuntu WSL2):

```text
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
```

---

## AI Log
- Sử dụng Antigravity AI hỗ trợ chạy thử nghiệm các bài lab Part 1–5 trong môi trường Ubuntu WSL2 + Docker, tổng hợp kết quả đo đạc thực tế và triển khai Part 6 (`06-lab-complete/`). Đã kiểm tra lại toàn bộ câu trả lời khớp với source code và output lệnh thực tế.
