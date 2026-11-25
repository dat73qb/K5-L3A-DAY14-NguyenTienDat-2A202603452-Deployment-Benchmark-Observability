# Benchmark & Observability Report — Day 14

> Copy file này thành `BENCHMARK_REPORT.md` trong repo của bạn và điền đầy đủ.
> Mọi con số phải lấy từ report JSON / CSV / screenshot thật, không ước lượng.

**Họ tên:** Nguyễn Tiến Đạt (`2A202603452`)
**Ngày chạy:** 2026-11-25
**Môi trường:** Windows + Ubuntu 24.04 (WSL2), Docker Desktop 28.5.1, Python 3.12.3

---

## 1. SLO của agent

| SLI | SLO | Lý do chọn ngưỡng này |
|---|---|---|
| p95 latency `/ask` | `< 500 ms` | Latency mock LLM cơ bản ~150–200ms; p95 < 500ms đảm bảo trải nghiệm mượt cho 95% người dùng trước khi tính tới tail latency (slow requests). |
| Error rate `/ask` | `< 2.0 %` | Mock LLM có tỉ lệ lỗi upstream mặc định ~1% (`MOCK_ERROR_RATE=0.01`), đặt ngưỡng 2.0% để chấp nhận lỗi lẻ tẻ nhưng bắt được sự cố hệ thống. |
| Accuracy trên golden dataset | `≥ 80 %` (mục tiêu sau sửa KB: `100 %`) | Đảm bảo chất lượng trả lời đúng trọng tâm kỹ thuật khi thay đổi prompt/KB và không bị regression so với baseline. |

---

## 2. Benchmark hiệu năng (develop, `load_test.py`)

### 2.1 Concurrency vs latency (agent v2)

| Concurrency | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Error rate | SLO |
|---|---|---|---|---|---|---|
| 1 | 4.51 | 193.2 | 221.7 | 1694.4 | 0.33% | PASS |
| 10 | 37.80 | 191.7 | 238.7 | 1703.9 | 0.67% | PASS |
| 50 | 83.81 | 228.5 | 1126.0 | 1788.9 | 0.33% | FAIL (p95 > 500ms) |
| 100 | 47.35 | 746.1 | 3876.8 | 4402.7 | 0.33% | FAIL (p95 > 500ms) |

**Nhận xét:** RPS **không tăng tuyến tính** theo concurrency. Từ concurrency 1 → 10 → 50, RPS tăng (4.51 → 37.80 → 83.81 req/s), nhưng khi lên 100, RPS lại **giảm xuống 47.35 req/s** và p95 nhảy vọt lên **3876.8 ms**. Nguyên nhân là một process uvicorn/event-loop bị bão hòa khi có quá nhiều kết nối đồng thời (contention + queue chờ), làm tăng mạnh thời gian chờ của từng request.

### 2.2 Tail latency

| `MOCK_SLOW_RATE` | avg (ms) | p95 (ms) | p99 (ms) |
|---|---|---|---|
| 0 | 203.3 | 252.6 | 288.2 |
| 0.03 (mặc định) | 236.7 | 308.9 | 1696.7 |
| 0.1 | 363.2 | 1696.9 | 1716.6 |

**Nhận xét:** **p99 và p95** nhạy nhất với các request chậm bất thường: khi `MOCK_SLOW_RATE` tăng từ `0` lên `0.03`, `avg` chỉ tăng nhẹ từ `203.3ms` lên `236.7ms`, nhưng `p99` nhảy vọt từ `288.2ms` lên `1696.7ms` (gấp ~6 lần); khi tăng lên `0.1`, `p95` cũng vọt lên `1696.9ms` và vi phạm SLO. Vì vậy SLO **không nên đặt theo average**, vì giá trị trung bình che giấu trải nghiệm rất tệ của nhóm 1%–5% người dùng bị chậm (tail latency).

---

## 3. Benchmark chất lượng (develop, `eval/run_eval.py`)

| Chỉ số | Agent v1 | Agent v2 | Agent v2 (sau khi sửa KB) |
|---|---|---|---|
| Accuracy | 4.3% (1/23) | 91.3% (21/23) | 100.0% (23/23) |
| Avg tokens / câu | 56.3 | 108.8 | 113.3 |
| Tổng cost (USD) | $0.000394 | $0.000758 | $0.000788 |
| Cost / câu đúng (USD) | $0.00039375 | $0.00003607 | $0.00003426 |
| Quality gate | FAIL | PASS (vs v1) | PASS (100%, 0 regression) |

**Những câu v2 trả lời sai ban đầu và cách bạn sửa:**

| ID | Keyword bị thiếu | Bạn đã sửa gì |
|---|---|---|
| `security-02` | `token`, `hết hạn`, `chữ ký` | Thêm entry `(["jwt", "api key"], ...)` vào `KNOWLEDGE_BASE` trong `utils/mock_llm.py` giải thích JWT là token có thời gian hết hạn và chữ ký số. |
| `scaling-03` | `cpu`, `memory`, `custom metrics` | Thêm entry `(["hpa", "kubernetes"], ...)` vào `KNOWLEDGE_BASE` trong `utils/mock_llm.py` mô tả Kubernetes HPA scale theo CPU, Memory và custom metrics. |

**Nhận xét về trade-off chất lượng / chi phí / latency giữa v1 và v2:**
- `v2` (có bước retrieval RAG) tốn gần gấp đôi số token/câu (`108.8–113.3` so với `56.3` của `v1`) và latency cao hơn một chút (thêm ~20ms retrieval), nhưng độ chính xác tăng từ **4.3%** lên **100%**.
- Nếu xét theo chỉ số **`cost / câu đúng`**, `v2` (`$0.00003426`) **rẻ hơn hơn 11 lần** so với `v1` (`$0.00039375`).
- **Quality gate trong CI:** Khi chạy `python eval/run_eval.py --url http://localhost:8001 --baseline reports/eval_v2.json`, lệnh trả về **exit code `1`** vì accuracy giảm `-87.0 điểm %` (20 câu bị regression). Trong GitHub Actions, chỉ cần thêm step `run: python eval/run_eval.py --min-accuracy 0.8 --baseline reports/eval_v2.json` — exit code khác 0 sẽ tự động làm job CI fail và chặn merge PR.

---

## 4. Observability stack (production)

### 4.1 Screenshot

- [ ] Dashboard Grafana khi đang có tải (thấy rõ Golden Signals) (`screenshots/grafana-dashboard.png`)
- [ ] Một trace chậm trong Jaeger (thấy span nào chiếm thời gian) (`screenshots/jaeger-slow-trace.png`)
- [ ] Một trace lỗi trong Jaeger (có event `exception`) (`screenshots/jaeger-error-trace.png`)
- [ ] Trang Alerts của Prometheus với ít nhất 2 alert đang **firing** (`screenshots/prometheus-alert-firing.png`)

### 4.2 Thí nghiệm "phá hệ thống"

| # | Thay đổi | Alert nào firing | Thời gian từ lúc đổi → firing | Panel nào phản ánh rõ nhất |
|---|---|---|---|---|
| A | `MOCK_ERROR_RATE=0.25` | `AgentHighErrorRate` (`for: 1m`) | ~1 phút 35 giây (kể từ lúc chạy tải lỗi) | Error Rate (%) & HTTP Status 5xx |
| B | `MOCK_SLOW_RATE=0.3` | `AgentHighLatencyP95` (`for: 2m`, đang `pending` ở 80s) | ~2 phút | Latency p50 / p95 / p99 |
| C | `LLM_MODEL=mock-llm-large` | `LLMCostBurnRateHigh` (`for: 2m`) | ~2 phút | LLM Cost Burn Rate ($/giờ) |
| D | `docker pause` 1 instance (`agent-3`) | `AgentInstanceDown` (`for: 30s`) | ~35 giây | Up / Active Instances |
| E | `AGENT_VERSION=v1` | Không có alert nào firing | — | Không thấy trên infra/latency dashboard (chỉ thấy token/cost giảm nhẹ) |

**Thí nghiệm E không làm alert nào firing. Bạn sẽ phát hiện loại sự cố này trong production bằng cách nào?**

`AGENT_VERSION=v1` vẫn trả HTTP 200 nhanh và không tốn nhiều tiền hơn nên các alert hạ tầng (error rate, latency, instance down, cost) đều im lặng, nhưng **chất lượng câu trả lời giảm mạnh** (từ 100% xuống 4.3%). Để phát hiện trong production cần:
1. Chạy **Eval Quality Gate** (`run_eval.py` trên golden dataset) trong CI/CD trước khi deploy.
2. Theo dõi **Online Quality Metrics** (user feedback thumbs-up/down, LLM-as-a-judge sampling trên production traffic) và đặt alert khi điểm chất lượng giảm.

### 4.3 Debug bằng trace

Chọn 1 trace chậm nhất tìm được trong logs & Jaeger (`observable-agent`):

- `trace_id`: `77077348fbc30628f1411fae40718388` (ngoài ra trace lỗi 503 để xem exception event: `14df8e6fd5f4f8f6b7e95b8cfde90c54`)
- Tổng thời gian: `1751.5 ms`
- Span chiếm nhiều thời gian nhất: `chat mock-llm-small` (~`1600 ms` / `1.6s`, chiếm ~91% tổng thời gian do dính tail latency `MOCK_SLOW_MS=1500ms`)
- Dòng log tương ứng (grep theo `trace_id`):

```json
{"ts": "2026-11-25T16:21:12.339927+00:00", "level": "INFO", "service": "observable-agent", "instance": "6ef7b0703a25", "event": "http_request", "trace_id": "77077348fbc30628f1411fae40718388", "span_id": "55ebf3368170ead8", "request_id": "e6c00367b1dd", "method": "POST", "route": "/ask", "status": 200, "ms": 1751.5}
```

---

## 5. Benchmark scaling (production, Locust headless, 100 users, 2 phút)

| Số instance | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Failures |
|---|---|---|---|---|---|
| 1 (`scale_1_stats.csv`, Aggregated) | 48.78 | 220 | 3200 | 7500 | 52 (0.90%) |
| 3 (`scale_3_stats.csv`, Aggregated) | 49.80 | 210 | 3200 | 5600 | 55 (0.93%) |

**Giải thích kết quả:**
- Khi scale lên 3 instances, `p99` giảm rõ rệt từ **7500 ms → 5600 ms**, `Average Response Time` giảm từ **705.6 ms → 636.8 ms**, và `RPS` tăng nhẹ (**48.78 → 49.80 req/s**), nhưng `p95` vẫn giữ ở **3200 ms**.
- **Nút thắt nằm ở đâu:** Tất cả các container (3 agent + nginx + prometheus + jaeger + locust) đều chạy chung trên cùng 1 máy host nên chia sẻ CPU; đồng thời `locustfile.py` giới hạn tốc độ sinh tải của 100 user (có wait time giữa các request) và các request dính `MOCK_SLOW_RATE=0.03` (+1500ms) vẫn tạo ra tail latency cao.

---

## 6. Kết luận

- 3 điều học được về benchmark & observability:
  1. **Average che giấu tail latency:** luôn đặt SLO và giám sát theo **p95 / p99** thay vì trung bình.
  2. **Eval & Quality Gate bắt lỗi âm thầm:** alert hạ tầng không phát hiện được suy giảm chất lượng model/prompt (`v1` vs `v2`), cần eval tự động trên golden dataset và so sánh `cost / câu đúng`.
  3. **3 trụ cột liên kết nhau:** Metrics (Grafana/Prometheus) báo *khi nào* có vấn đề → Traces (Jaeger) chỉ *bước nào* chậm/lỗi → Logs JSON (lọc theo `trace_id`) cho biết *chi tiết* chuyện gì xảy ra.
- Nếu đưa agent này lên production thật, sẽ thêm:
  - Alert cho **Redis connection / readiness failure** (`GET /ready` trả 503) và **Rate limit / Budget exceeded** (`429`, `402` spike).
  - Metric **TTFT (Time To First Token)** nếu dùng streaming và **Online Eval / User Feedback Score** để phát hiện suy giảm chất lượng theo thời gian thực.
