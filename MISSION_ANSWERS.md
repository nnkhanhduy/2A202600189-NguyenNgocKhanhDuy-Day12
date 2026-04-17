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

### Exercise 2.1: Dockerfile 

1. **Base image:** `python:3.11` — full Python distribution, khoảng ~1 GB
2. **Working directory:** `/app`
3. **Tại sao COPY requirements.txt trước?** — Docker build theo từng layer. Nếu `requirements.txt` không thay đổi, layer `pip install` được cache lại → build nhanh hơn. Nếu copy code trước, mỗi lần sửa code đều phải pip install lại.
4. **CMD vs ENTRYPOINT khác nhau thế nào?** — `CMD` cung cấp lệnh mặc định, có thể bị override khi chạy `docker run <image> <other-cmd>`. `ENTRYPOINT` cố định binary chính, không bị override dễ dàng — thường dùng cho executable container.

### Exercise 2.2: Build và run

```bash
$ docker images my-agent:develop
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
Client (HTTP :80 / HTTPS :443)
        │
        ▼
┌──────────────────┐
│  Nginx           │  Reverse proxy + Load balancer
│  (nginx:alpine)  │
└────────┬─────────┘
         │ internal network
         ▼
┌──────────────────┐
│  Agent           │  FastAPI AI agent (scalable)
│  (python:3.11)   │──────────────────────────────┐
│  :8000           │                              │
└────────┬─────────┘                              │
         │                                        │
         ▼                                        ▼
┌──────────────────┐                 ┌────────────────────┐
│  Redis           │                 │  Qdrant            │
│  (redis:7-alpine)│                 │  (qdrant:v1.9.0)   │
│  :6379           │                 │  :6333             │
│  Session cache   │                 │  Vector database   │
│  Rate limiting   │                 │  (RAG)             │
└──────────────────┘                 └────────────────────┘
```

**4 services:**
- **nginx**: Reverse proxy, expose port 80/443 ra ngoài, phân tán traffic đến agent. Client chỉ giao tiếp qua Nginx, không trực tiếp với agent.
- **agent**: FastAPI app, build từ Dockerfile stage `runtime`, không expose port trực tiếp ra host — chỉ accessible qua Nginx trong internal network.
- **redis**: Cache session và rate limiting, giới hạn 256 MB RAM, LRU eviction, có volume `redis_data` để persist data khi restart.
- **qdrant**: Vector database cho RAG (Retrieval-Augmented Generation), có volume `qdrant_data`.

**Tất cả services dùng chung network `internal` (bridge driver)** — isolated, không expose ra ngoài trừ Nginx. Agent kết nối Redis qua `redis://redis:6379/0` và Qdrant qua `http://qdrant:6333` bằng service name làm hostname.

---

## Part 3: Cloud Deployment

### Exercise 3.1 & 3.2: Deployment

**Platform:** Railway
**URL:** https://myagent-production-7f17.up.railway.app 

**Test kết quả:**
```bash
# Health check
curl https://myagent-production-7f17.up.railway.app/health
# → {"status":"ok","uptime_seconds":8920.1,"platform":"Railway","timestamp":"2026-04-17T10:55:00.804693+00:00"}
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

**Get token:**
```bash
curl http://localhost:8000/token -X POST \
  -H "Content-Type: application/json" \
  -d '{"username": "student", "password": "demo123"}'
# {"access_token": "eyJ...", "token_type": "bearer"}
```

**Use token:**
```bash
TOKEN="eyJ..."
curl http://localhost:8000/ask -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"question": "Explain JWT"}'
# {"answer": "..."}
```

### Exercise 4.3: Rate Limiting (`04-api-gateway/production/rate_limiter.py`)

**Algorithm:** Sliding Window Counter — mỗi user có một `deque` lưu timestamp các request. Mỗi lần request, các timestamp cũ hơn `window_seconds` bị xóa, sau đó so sánh số còn lại với `max_requests`.

**Giới hạn:**
- User thường: **10 requests / 60 giây** (`rate_limiter_user`)
- Admin: **100 requests / 60 giây** (`rate_limiter_admin`)

**Admin bypass limit như thế nào:**
JWT payload chứa field `role`. Endpoint đọc `user["role"]` và gọi `rate_limiter_admin.check(user_id)` hoặc `rate_limiter_user.check(user_id)` tương ứng — admin dùng bucket 100 req/min.

**Test — hit limit:**
```bash
for i in {1..15}; do
  curl http://localhost:8000/ask -X POST \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"question": "Test '$i'"}'
  echo ""
done
# Request 1–10:  200 OK, header X-RateLimit-Remaining đếm ngược
# Request 11+:   429 Too Many Requests
# {"detail": {"error": "Rate limit exceeded", "limit": 10, "retry_after_seconds": N}}
```

### Exercise 4.4: Cost Guard Implementation (`04-api-gateway/production/cost_guard.py`)

**Cách tiếp cận:**
`CostGuard` theo dõi token usage theo từng user theo ngày bằng in-memory `dict[user_id → UsageRecord]`. Mỗi `UsageRecord` tích lũy `input_tokens` và `output_tokens`; chi phí tính theo:

```
cost = (input_tokens / 1000) × $0.00015 + (output_tokens / 1000) × $0.0006
```

Hai mức giới hạn:
- **Per-user:** `daily_budget_usd = $1.00` — raise `HTTP 402` khi vượt
- **Global:** `global_daily_budget_usd = $10.00` — raise `HTTP 503` khi toàn service vượt budget

**Flow:**
1. `check_budget(user_id)` được gọi trước khi gọi LLM — block nếu đã vượt budget
2. LLM chạy và trả về số token
3. `record_usage(user_id, input_tokens, output_tokens)` cập nhật record và global counter
4. Khi đạt 80% budget per-user, ghi warning vào log

**Phiên bản dùng Redis (production-grade):**
```python
def check_budget(user_id: str, estimated_cost: float) -> bool:
    month_key = datetime.now().strftime("%Y-%m")
    key = f"budget:{user_id}:{month_key}"
    current = float(r.get(key) or 0)
    if current + estimated_cost > 10:
        return False
    r.incrbyfloat(key, estimated_cost)
    r.expire(key, 32 * 24 * 3600)  # tự reset sau ~1 tháng
    return True
```
Dùng Redis thay vì in-memory đảm bảo budget counter tồn tại sau khi restart và được chia sẻ giữa tất cả instances khi scale.

---

## Part 5: Scaling & Reliability

### Exercise 5.1: Health checks (`05-scaling-reliability/develop/app.py`)

**Implementation:**

```python
@app.get("/health")
def health():
    """Liveness probe — process còn sống không?"""
    return {
        "status": "ok",
        "uptime_seconds": round(time.time() - START_TIME, 1),
        "version": "1.0.0",
        "environment": os.getenv("ENVIRONMENT", "development"),
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "checks": { "memory": {"status": "ok", "used_percent": mem.percent} }
    }

@app.get("/ready")
def ready():
    """Readiness probe — app đã sẵn sàng nhận traffic chưa?"""
    if not _is_ready:
        raise HTTPException(503, "Agent not ready. Check back in a few seconds.")
    return {"ready": True, "in_flight_requests": _in_flight_requests}
```

**Sự khác nhau giữa `/health` và `/ready`:**

| Probe | Mục đích | Trả về 503 khi |
|---|---|---|
| `/health` (liveness) | Process còn sống không? Platform restart container nếu fail | Process bị crash hoặc treo |
| `/ready` (readiness) | App sẵn sàng nhận traffic chưa? Load balancer ngừng route nếu fail | Đang khởi động, shutdown, hoặc dependency chưa sẵn sàng |

### Exercise 5.2: Graceful shutdown (`05-scaling-reliability/develop/app.py`)

**Cách hoạt động:**
App dùng FastAPI `lifespan` context manager. Khi shutdown (SIGTERM → uvicorn → lifespan exit), app đặt `_is_ready = False` để ngừng nhận request mới, sau đó poll `_in_flight_requests` cho đến khi về 0 hoặc hết timeout 30 giây.

Signal handler `signal.signal(SIGTERM, handle_sigterm)` ghi log khi nhận tín hiệu; uvicorn's SIGTERM handler thực sự kích hoạt lifespan shutdown.

**Test:**
```bash
python app.py &
PID=$!

# Gửi request chậm
curl http://localhost:8000/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "Long task"}' &

# Ngay lập tức gửi SIGTERM
kill -TERM $PID

# Quan sát: request đang xử lý hoàn thành trước khi process thoát
# Log hiển thị: "Waiting for 1 in-flight requests..." rồi "Shutdown complete"
```

### Exercise 5.3: Stateless design (`05-scaling-reliability/production/app.py`)

**Anti-pattern (stateful — lưu trong memory):**
```python
# In-memory dict — mất khi process chết, không chia sẻ giữa các instances
conversation_history = {}

@app.post("/ask")
def ask(user_id: str, question: str):
    history = conversation_history.get(user_id, [])
```

**Đúng (stateless — lưu trong Redis):**
```python
# Redis-backed — mọi instance đều đọc được session của bất kỳ user nào
def load_session(session_id: str) -> dict:
    data = _redis.get(f"session:{session_id}")
    return json.loads(data) if data else {}

@app.post("/chat")
async def chat(body: ChatRequest):
    session_id = body.session_id or str(uuid.uuid4())
    append_to_history(session_id, "user", body.question)   # ghi vào Redis
    answer = ask(body.question)
    append_to_history(session_id, "assistant", answer)     # ghi vào Redis
    return {"session_id": session_id, "answer": answer, "served_by": INSTANCE_ID}
```

**Tại sao stateless quan trọng:**
Khi 3 agent instances chạy sau Nginx, mỗi request có thể rơi vào bất kỳ instance nào. Nếu session data nằm trong memory của instance, request thứ 2 của user rơi vào instance khác sẽ không có history. Redis là shared external store — instance nào cũng đọc được cùng key `session:{id}` bất kể instance nào xử lý request đầu tiên.

### Exercise 5.4: Load balancing

```bash
docker compose up --scale agent=3
```

**Quan sát:**
- Docker Compose tạo 3 container: `agent-1`, `agent-2`, `agent-3`
- Nginx upstream `agent_backend` resolve `agent:8000` — Docker's internal DNS trả về cả 3 IP, Nginx round-robin qua chúng
- Field `served_by` trong response `/chat` luân phiên giữa 3 `INSTANCE_ID`, xác nhận traffic được phân tán
- Nếu kill 1 instance, Nginx ngừng route đến nó khi upstream probe tiếp theo fail; 2 instance còn lại tiếp tục phục vụ

**Test:**
```bash
for i in {1..10}; do
  curl -s http://localhost/chat -X POST \
    -H "Content-Type: application/json" \
    -d '{"question": "Request '$i'"}' | python3 -m json.tool | grep served_by
done
# Output luân phiên giữa instance-abc123, instance-def456, instance-ghi789
```

### Exercise 5.5: Stateless test (`05-scaling-reliability/production/test_stateless.py`)

```bash
python test_stateless.py
```

**Script kiểm tra những gì:**
1. Tạo một conversation trên một instance — lưu `session_id`
2. Gửi các tin nhắn tiếp theo — mỗi tin có thể được xử lý bởi instance khác nhau
3. Kiểm tra conversation history còn nguyên vẹn bất kể instance nào xử lý từng lượt
4. Nếu Redis available, kill một container ngẫu nhiên và xác nhận session tồn tại trên các instance còn lại

**Kết quả mong đợi:** Tất cả lượt hội thoại đều có trong history cuối cùng, và `served_by` hiển thị các instance ID khác nhau — chứng minh state nằm trong Redis, không phải trong memory của bất kỳ process nào.
