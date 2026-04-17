# Day 12 Lab - Mission Answers

> **Student Name:** Nguyễn Ngọc Khánh Duy
> **Date:** 17/04/2026

---

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found in `01-localhost-vs-production/develop/app.py`

1. **Hardcoded secrets** — `OPENAI_API_KEY` và `DATABASE_URL` được gán thẳng trong code. Nếu push lên GitHub, credentials bị lộ ngay lập tức.
2. **Secret bị log ra ngoài** — `print(f"[DEBUG] Using key: {OPENAI_API_KEY}")` in secret ra stdout, ai xem logs đều thấy.
3. **Dùng `print()` thay vì logging** — Không có log level, không có format chuẩn, không thể filter hay parse.
4. **Không có health check endpoint** — Platform không thể biết khi nào app crash để tự restart.
5. **Host cứng là `localhost`** — `uvicorn.run(host="localhost")` chỉ nhận kết nối từ chính máy đó, không nhận traffic từ ngoài.
6. **Port cứng là `8000`** — Không đọc từ `PORT` env var, sẽ conflict khi deploy lên Railway/Render (platform inject PORT khác).
7. **`reload=True` trong production** — Auto-reload chỉ dùng cho dev, gây overhead và không ổn định khi chạy production.

### Exercise 1.2: Chạy basic version

```bash
$ python app.py
INFO:     Will watch for changes in these directories: ['.../develop']
INFO:     Uvicorn running on http://localhost:8000 (Press CTRL+C to quit)
INFO:     Started reloader process [28624] using WatchFiles
INFO:     Started server process [13460]
INFO:     Application startup complete.

$ curl -X POST "http://localhost:8000/ask?question=Hello"
{"answer":"Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé."}
```

**Quan sát:** App chạy được! Nhưng KHÔNG production-ready vì:
- `question` truyền qua query string, không validate kiểu dữ liệu
- Không có `/health` endpoint → thử `GET /health` trả về `404`
- Logs chỉ là `print()` ra stdout, không có format chuẩn
- `reload=True` đang chạy — thấy "Will watch for changes" trong logs
- Secret `OPENAI_API_KEY` bị in ra stdout mỗi request

### Exercise 1.3: Comparison table

| Feature | Basic (develop) | Advanced (production) | Tại sao quan trọng? |
|---|---|---|---|
| **Config** | Hardcode trong code | Đọc từ environment variables | Secrets không bị lộ, dễ thay đổi per environment |
| **Health check** | Không có | `GET /health` + `GET /ready` | Platform biết khi nào restart container |
| **Logging** | `print()` | Structured JSON (`{"ts":...,"lvl":...,"msg":...}`) | Dễ parse, filter, monitor trên cloud |
| **Shutdown** | Đột ngột (SIGKILL) | Graceful (bắt SIGTERM, hoàn thành request rồi mới tắt) | Không mất request đang xử lý khi deploy |
| **Host** | `localhost` | `0.0.0.0` | Nhận traffic từ bên ngoài container |
| **Port** | Cứng `8000` | Đọc từ `os.getenv("PORT", "8000")` | Compatible với mọi cloud platform |
| **Debug mode** | `reload=True` luôn | Chỉ bật khi `DEBUG=true` trong env | Tránh overhead và lỗ hổng bảo mật ở production |
| **Authentication** | Không có | API Key bắt buộc (`X-API-Key` header) | Ai cũng gọi được = hết tiền OpenAI |

---

## Part 2: Docker

### Exercise 2.1: Dockerfile cơ bản

1. **Base image là gì?** — Base image là image gốc mà Dockerfile kế thừa, được khai báo bằng `FROM`. Nó cung cấp OS, runtime, và các tool cơ bản. Ví dụ `python:3.11` cung cấp sẵn Python 3.11 và pip, giúp không cần cài thủ công.

2. **Working directory là gì?** — `WORKDIR` đặt thư mục làm việc mặc định bên trong container cho tất cả các lệnh `RUN`, `COPY`, `CMD` phía sau. Nếu thư mục chưa tồn tại, Docker tự tạo. Giúp tránh dùng đường dẫn tuyệt đối dài và giữ code gọn gàng.

3. **Tại sao COPY requirements.txt trước khi COPY code?** — Docker build từng instruction thành một layer và cache lại. Dependencies thường ít thay đổi hơn code. Nếu `requirements.txt` không đổi, layer `pip install` được dùng từ cache → build nhanh hơn nhiều. Ngược lại, nếu COPY toàn bộ code trước, mỗi lần sửa bất kỳ file nào cũng invalidate cache và phải pip install lại từ đầu.

4. **CMD vs ENTRYPOINT khác nhau thế nào?** — `ENTRYPOINT` định nghĩa executable chính của container, không bị override bởi argument truyền vào `docker run`. `CMD` cung cấp lệnh/argument mặc định, có thể bị override hoàn toàn khi truyền command vào `docker run <image> <cmd>`. Thường kết hợp: `ENTRYPOINT` = binary, `CMD` = default arguments.

### Exercise 2.2: Build và run

```bash
$ docker build -f 02-docker/develop/Dockerfile -t my-agent:develop .
$ docker run -p 8000:8000 my-agent:develop

$ curl http://localhost:8000/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "What is Docker?"}'
{"answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!"}

$ docker images my-agent:develop
WARNING: This output is designed for human readability. For machine-readable output, please use --format.
my-agent:develop     5eadf0e1b361    1.66GB
```

**Quan sát:** Image size là **1.66 GB** — rất lớn vì dùng `python:3.11` full distribution kèm toàn bộ build tools. Không phù hợp cho production vì tốn băng thông khi push/pull, khởi động chậm hơn.

### Exercise 2.3: Image size comparison

```bash
$ docker images | grep my-agent
my-agent:advanced    8c936bc4ac44    236MB
my-agent:develop     5eadf0e1b361    1.66GB
```

| Image | Size thực tế | Lý do |
|---|---|---|
| `my-agent:develop` (single-stage `python:3.11`) | **1.66 GB** | Full Python distribution + build tools |
| `my-agent:advanced` (multi-stage `python:3.11-slim`) | **236 MB** | Chỉ giữ runtime, loại bỏ gcc, libpq-dev, pip cache |
| **Chênh lệch** | **~86% nhỏ hơn** | Multi-stage tách build environment ra khỏi runtime image |

**Multi-stage hoạt động như thế nào:**
- Stage 1 (builder): dùng `python:3.11-slim` + cài `gcc`, `libpq-dev`, pip install với `--user`
- Stage 2 (runtime): dùng `python:3.11-slim` sạch, chỉ `COPY --from=builder /root/.local` → không có build tools trong image cuối

### Exercise 2.4: Docker Compose architecture

```
Client
  │
  ▼
agent (FastAPI :8000)
  │
  ▼
redis (Redis :6379)
```

Services:
- **agent**: FastAPI app, build từ Dockerfile, expose port 8000, đọc env từ `.env`, phụ thuộc redis healthy
- **redis**: Redis 7 Alpine, giới hạn 128 MB RAM, dùng LRU eviction policy

Services communicate qua Docker internal network, agent kết nối redis qua `REDIS_URL=redis://redis:6379/0` (dùng service name `redis` làm hostname).

---

## Part 3: Cloud Deployment

### Exercise 3.1 & 3.2: Deployment

**Platform:** Render  
**URL:** https://ai-agent-production.onrender.com  

**Test kết quả:**
```bash
# Health check
curl https://ai-agent-production.onrender.com/health
# → {"status":"ok","version":"1.0.0","environment":"production",...}

# Không có key → 401
curl -X POST https://ai-agent-production.onrender.com/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
# → {"detail":"Invalid or missing API key..."}

# Có key → 200
curl -X POST https://ai-agent-production.onrender.com/ask \
  -H "X-API-Key: <AGENT_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"question": "What is deployment?"}'
# → {"question":"...","answer":"...","model":"gpt-4o-mini","timestamp":"..."}
```

**So sánh `render.yaml` vs `railway.toml`:**

| | `render.yaml` | `railway.toml` |
|---|---|---|
| Format | YAML | TOML |
| Runtime | Chỉ định `runtime: docker` | Tự detect hoặc dùng DOCKERFILE builder |
| Env vars | Khai báo trong file với `generateValue` | Set qua CLI: `railway variables set` |
| Health check | `healthCheckPath: /health` | `healthcheckPath = "/health"` |
| Region | Chọn được (`singapore`) | Tự động |
| Plan | Khai báo `plan: free` | Theo account |

---

## Part 4: API Security

### Exercise 4.1: API Key Authentication

API key được check tại `verify_api_key()` trong `app/auth.py`:
- Đọc header `X-API-Key`
- So sánh với `settings.agent_api_key` (từ env var `AGENT_API_KEY`)
- Nếu sai hoặc thiếu → raise `HTTPException(401)`

**Kết quả test:**
```bash
# Không có key
curl http://localhost:8000/ask -X POST -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
# → 401: {"detail":"Invalid or missing API key. Include header: X-API-Key: <key>"}

# Sai key
curl http://localhost:8000/ask -X POST -H "X-API-Key: wrong-key" \
  -H "Content-Type: application/json" -d '{"question": "Hello"}'
# → 401

# Đúng key
curl http://localhost:8000/ask -X POST -H "X-API-Key: dev-key-change-me-in-production" \
  -H "Content-Type: application/json" -d '{"question": "Hello"}'
# → 200: {"question":"Hello","answer":"...","model":"gpt-4o-mini",...}
```

**Rotate key:** Chỉ cần đổi giá trị `AGENT_API_KEY` trong env và restart service.

### Exercise 4.2: JWT Authentication

JWT flow trong `app/auth.py`:
1. `create_jwt_token(subject)` → tạo token với payload `{sub, iat, exp}`, ký bằng `JWT_SECRET`
2. Token expire sau 3600 giây (1 giờ)
3. `verify_jwt_token(token)` → decode và validate, raise 401 nếu expired hoặc invalid

### Exercise 4.3: Rate Limiting

**Algorithm:** Sliding window (cửa sổ trượt 60 giây)

**Cơ chế:**
- Mỗi API key có một `deque` lưu timestamp các request
- Mỗi request: xóa các timestamp cũ hơn 60 giây, đếm số còn lại
- Nếu ≥ `RATE_LIMIT_PER_MINUTE` (mặc định 20) → raise `HTTPException(429)` với header `Retry-After: 60`

**Limit:** 20 req/min mặc định, cấu hình qua env var `RATE_LIMIT_PER_MINUTE`

**Test:**
```bash
for i in {1..25}; do
  curl -s -o /dev/null -w "%{http_code}\n" \
    http://localhost:8000/ask -X POST \
    -H "X-API-Key: dev-key-change-me-in-production" \
    -H "Content-Type: application/json" \
    -d '{"question": "test"}'
done
# Request 1-20: 200
# Request 21+: 429
```

### Exercise 4.4: Cost Guard Implementation

Cost guard trong `app/cost_guard.py` dùng in-memory tracking theo ngày:
- Reset mỗi ngày mới (so sánh `time.strftime("%Y-%m-%d")`)
- Budget: `DAILY_BUDGET_USD` (mặc định $5.0/ngày)
- Chi phí ước tính: input tokens × $0.00015/1K + output tokens × $0.0006/1K
- Vượt budget → raise `HTTPException(503)`

```python
def check_and_record_cost(input_tokens: int, output_tokens: int) -> None:
    global _daily_cost, _cost_reset_day
    today = time.strftime("%Y-%m-%d")
    if today != _cost_reset_day:       # reset mỗi ngày
        _daily_cost = 0.0
        _cost_reset_day = today
    if _daily_cost >= settings.daily_budget_usd:
        raise HTTPException(503, "Daily budget exhausted. Try tomorrow.")
    cost = (input_tokens / 1000) * 0.00015 + (output_tokens / 1000) * 0.0006
    _daily_cost += cost
```

---

## Part 5: Scaling & Reliability

### Exercise 5.1: Health Checks

Đã implement 2 endpoints:

```python
@app.get("/health")   # Liveness probe
def health():
    return {"status": "ok", "uptime_seconds": ..., "checks": {...}}

@app.get("/ready")    # Readiness probe
def ready():
    if not _is_ready:
        raise HTTPException(503, "Not ready")
    return {"ready": True}
```

- `/health`: platform dùng để kiểm tra process còn sống → restart nếu fail
- `/ready`: load balancer dùng để kiểm tra sẵn sàng nhận traffic → không route request nếu fail (ví dụ khi đang khởi động)

### Exercise 5.2: Graceful Shutdown

```python
def _handle_signal(signum, _frame):
    logger.info(json.dumps({"event": "signal", "signum": signum}))

signal.signal(signal.SIGTERM, _handle_signal)
```

Uvicorn được start với `timeout_graceful_shutdown=30` — khi nhận SIGTERM, Uvicorn:
1. Ngừng nhận request mới
2. Chờ tối đa 30 giây cho các request đang xử lý hoàn thành
3. Sau đó shutdown

**Test:**
```bash
# Start app, gửi request dài, rồi kill -TERM
docker stop <container>  # Docker gửi SIGTERM trước SIGKILL
# → Request đang xử lý hoàn thành trước khi container dừng
```

### Exercise 5.3: Stateless Design

**Anti-pattern (trong memory):**
```python
conversation_history = {}   # mỗi instance có dict riêng
                             # scale ra 3 instances → 3 dict khác nhau
```

**Correct (trong Redis):**
```python
history = r.lrange(f"history:{user_id}", 0, -1)
# Tất cả instances đọc từ cùng 1 Redis
# Instance nào xử lý request cũng có đúng history
```

App hiện tại đã stateless — rate limiter và cost guard dùng in-memory (chấp nhận được vì là per-instance limit), conversation history không lưu state.

### Exercise 5.4: Load Balancing

```bash
docker compose up --scale agent=3
```

Nginx phân tán request theo round-robin. Nếu 1 instance fail health check, Nginx tự loại khỏi pool.

**Kết quả:** 10 request được xử lý bởi 3 instance khác nhau (kiểm tra qua `docker compose logs agent`).

### Exercise 5.5: Stateless Test

Kịch bản:
1. Gọi `/ask` → instance 1 xử lý
2. Kill instance 1
3. Gọi tiếp → instance 2 hoặc 3 xử lý → vẫn hoạt động bình thường

→ Confirm: stateless design cho phép horizontal scaling mà không mất session.
