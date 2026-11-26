# Day 14 Lab - Mission Answers

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found

File: `01-localhost-vs-production/develop/app.py`

1. **Hardcode secret** (dòng 17–18): `OPENAI_API_KEY` và `DATABASE_URL` (có cả mật khẩu `password123`) ghi thẳng trong code. Push lên GitHub là key bị lộ ngay, bot quét repo public sẽ lấy được trong vài phút.
2. **Log ra secret** (dòng 34): `print(f"[DEBUG] Using key: {OPENAI_API_KEY}")` đưa key vào log. Log thường được nhiều người và nhiều hệ thống đọc (CI, dashboard, nhà cung cấp log), nên secret bị lộ rộng hơn nữa.
3. **Dùng `print()` thay vì logging** (dòng 33, 38): không có level, timestamp hay cấu trúc, công cụ log không lọc hoặc parse được. Câu hỏi và câu trả lời của user cũng bị in nguyên văn ra log.
4. **Không có config management** (dòng 21–22): `DEBUG = True`, `MAX_TOKENS = 500` cố định trong code. Muốn đổi giữa dev/staging/prod phải sửa code rồi deploy lại.
5. **Không có health check endpoint**: không có `/health` hay `/ready`, nên khi agent treo hoặc crash, platform không biết để restart hay ngừng gửi traffic vào.
6. **Host = `localhost`** (dòng 51): app chỉ nhận kết nối từ chính máy đó. Chạy trong container hoặc trên cloud thì bên ngoài không gọi vào được.
7. **Port cố định 8000** (dòng 52): Railway/Render cấp port qua biến môi trường `PORT`. App không đọc biến này thì platform không route được traffic vào, và health check của platform fail.
8. **`reload=True`** (dòng 53): chế độ dev, theo dõi file và tự restart. Ở production nó tốn tài nguyên và có thể restart giữa chừng.
9. **Không xử lý graceful shutdown**: khi platform gửi SIGTERM (deploy mới, scale down), request đang chạy bị cắt ngang.

### Exercise 1.2: Chạy bản develop

```bash
cd 01-localhost-vs-production/develop
pip install -r requirements.txt
python app.py
curl -X POST "http://localhost:8000/ask?question=Hello"
```

Kết quả:

```
{"answer":"Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé."}
```

Nhận xét:
- App chạy được, nhưng chưa production-ready vì các vấn đề ở 1.1.
- Bản develop nhận `question` qua **query string**. Gửi JSON body sẽ bị **422** (`Field required`, `loc: ["query","question"]`).
- Terminal in ra cả API key (`[DEBUG] Using key: sk-...`): đây là anti-pattern số 2 nhìn thấy tận mắt.

### Exercise 1.3: Comparison table

| Feature | Develop | Production | Tại sao quan trọng? |
|---------|---------|------------|---------------------|
| Config | Hardcode trong code | Đọc env var qua `config.py` + `.env` (12-Factor) | Đổi cấu hình giữa dev/staging/prod không phải sửa code. Cùng một image chạy được ở mọi môi trường |
| Secrets | Ghi thẳng key, còn in ra log | Lấy từ env, log không chứa secret | Key lộ thì phải thu hồi gấp và có thể mất tiền (người khác dùng key gọi LLM). `.env` nằm trong `.gitignore` nên không bị commit |
| Health check | Không có | `/health` (liveness) + `/ready` (readiness) | Liveness cho platform biết khi nào cần restart. Readiness cho load balancer biết khi nào được đưa traffic vào instance |
| Logging | `print()` | JSON có cấu trúc, không log secret | Dễ tìm kiếm, lọc và đưa vào công cụ log (Loki, Datadog...). Lọc được theo field, ví dụ mọi request lỗi trong 5 phút qua |
| Shutdown | Đột ngột | Graceful: bắt SIGTERM, dọn dẹp trong `lifespan` | Mỗi lần deploy hoặc scale down không cắt ngang request đang chạy của user, không mất dữ liệu đang ghi |
| Host | `localhost` | `0.0.0.0` | Container và cloud cần nhận kết nối từ ngoài, `localhost` chỉ nhận từ chính máy đó |
| Port | Cố định 8000 | Đọc từ biến `PORT` | Cloud platform tự cấp port, app phải nghe đúng port đó thì mới nhận được traffic |
| Debug / reload | Luôn bật | Chỉ bật khi `DEBUG=true` | Reload ở production tốn tài nguyên. Debug mode có thể lộ stack trace và thông tin nội bộ |
| Validate input | Query param bắt buộc → 422 | `request.json()` thủ công: body không phải JSON hợp lệ → **500** | Điểm còn yếu của bản production (gặp khi test từ Windows terminal): lỗi do client nên trả 400/422. Dùng Pydantic model thì FastAPI tự trả 422 |

---

## Part 2: Docker

### Exercise 2.1: Dockerfile questions

File: `02-docker/develop/Dockerfile`

1. **Base image:** `python:3.11`, bản Python đầy đủ (Debian kèm compiler, header, công cụ build), khoảng 1 GB. Tiện cho dev nhưng nặng và nhiều thứ không dùng tới khi chạy.
2. **Working directory:** `/app`. Mọi lệnh `COPY`, `RUN`, `CMD` sau đó chạy tương đối với thư mục này trong container.
3. **Tại sao COPY requirements.txt trước:** Docker cache theo từng layer, và một layer chỉ build lại khi nó hoặc layer trước nó thay đổi. Copy `requirements.txt` rồi `pip install` trước, sau đó mới copy code, thì lần sau chỉ sửa code: layer `pip install` vẫn lấy từ cache, build mất vài giây. Nếu `COPY . .` đứng trước, mỗi lần sửa code đều làm cache mất hiệu lực và phải cài lại toàn bộ dependencies.
4. **CMD vs ENTRYPOINT:** `CMD` là lệnh hoặc tham số **mặc định**, bị ghi đè hoàn toàn khi chạy `docker run image <lệnh khác>`. `ENTRYPOINT` là chương trình **cố định** của container, phần gõ thêm sau `docker run image ...` được nối vào làm tham số (chỉ đổi được bằng `--entrypoint`). Hay dùng kết hợp: `ENTRYPOINT ["uvicorn"]` + `CMD ["app:app", "--port", "8000"]`. File này chỉ dùng `CMD ["python", "app.py"]`.

### Exercise 2.2: Build và run

```bash
docker build -f 02-docker/develop/Dockerfile -t my-agent:develop .
docker run --rm -p 8000:8000 my-agent:develop
curl -X POST "http://localhost:8000/ask?question=What%20is%20Docker"
```

Kết quả:

```
{"answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!"}
```

### Exercise 2.3: Image size comparison

Đo bằng `docker images my-agent`. Docker bản này in hai cột kích thước:

```
IMAGE               ID             DISK USAGE   CONTENT SIZE
my-agent:advanced   fd506b68147f        240MB         57.4MB
my-agent:develop    8e194a87c106       1.67GB          426MB
```

| | Develop | Production | Chênh lệch |
|---|---:|---:|---:|
| `CONTENT SIZE` (nén, dung lượng khi push/pull) | 426 MB | 57.4 MB | **86.52%** |
| `DISK USAGE` (giải nén, chiếm trên ổ đĩa) | 1670 MB | 240 MB | 85.63% |

Công thức: (develop − production) / develop × 100. Cả hai cách đo đều giảm khoảng 86%, image production đạt yêu cầu < 500 MB.

**Multi-stage giảm dung lượng ở đâu:**
- **Base image nhỏ hơn:** stage cuối dùng `python:3.11-slim` thay cho `python:3.11` (bản full), nên bỏ được compiler, header và tài liệu của Debian.
- **Công cụ build nằm lại ở stage `builder`:** `gcc`, `libpq-dev` và cache của `apt` chỉ được cài ở stage `builder`. Stage `builder` bị bỏ đi sau khi build, không đi vào image cuối.
- **Chỉ copy kết quả:** stage `runtime` chỉ lấy `/root/.local` (các package đã cài) qua `COPY --from=builder`, cộng thêm `main.py` và `utils/`. Không có source thừa, không có pip cache (`--no-cache-dir`).
- Ngoài dung lượng còn có lợi về bảo mật: ít package hơn nên ít lỗ hổng hơn, và chạy bằng user non-root `appuser`.

### Exercise 2.4: Docker Compose stack

Sơ đồ các service:

```bash
cd 02-docker/production
touch .env.local
docker compose -p k5-day14-docker up -d --build
```

Sơ đồ các service (file `02-docker/production/docker-compose.yml`):

```
                 ┌──────────────── network: internal (bridge) ────────────────┐
                 │                                                             │
 Client ──:80──▶ │  nginx (nginx:alpine)  ──proxy_pass──▶  agent:8000          │
 (curl/browser)  │  reverse proxy, LB,                      (FastAPI, build từ │
                 │  security headers                         production/Dockerfile)
                 │                                             │         │     │
                 │                                             ▼         ▼     │
                 │                                    redis:6379   qdrant:6333 │
                 │                                    (cache,      (vector DB, │
                 │                                    rate limit)   RAG)       │
                 └─────────────────────────────────────────────────────────────┘
```

- Chỉ `nginx` mở port ra ngoài (`80`, `443`). `agent`, `redis`, `qdrant` chỉ nằm trong network `internal`, bên ngoài không gọi thẳng vào được.
- `agent` có `depends_on` `redis` và `qdrant` với `condition: service_healthy`, nên chỉ khởi động khi hai service này đã qua healthcheck.
- `agent` tìm `redis` và `qdrant` bằng tên service, qua DNS nội bộ của Docker (`REDIS_URL=redis://redis:6379/0`, `QDRANT_URL=http://qdrant:6333`).

Trạng thái (`docker compose -p k5-day14-docker ps`): cả 4 container `Up`, trong đó `agent`, `redis`, `qdrant` ở trạng thái `healthy`.

Kết quả test qua Nginx:

```bash
curl http://localhost/health
```
```
{"status":"ok","uptime_seconds":409.0,"version":"2.0.0","timestamp":"2026-11-26T08:03:35.146391"}
```

```bash
curl -X POST http://localhost/ask -H "Content-Type: application/json" -d '{"question":"Hello"}'
```
```
{"answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic."}
```

Response header có `Server: nginx`, `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`, chứng tỏ request đã đi qua Nginx. `"version":"2.0.0"` cho thấy phía sau là app production (`main.py`), không phải app develop.

---

## Part 3: Cloud Deployment

### Exercise 3.1 / 3.2: Deploy agent Part 6

- Nền tảng: **Render** (Exercise 3.2). Deploy agent Part 6 (`06-lab-complete/`) bằng Blueprint `06-lab-complete/render.yaml`.
- URL: **https://production-agent-tqr2.onrender.com**
- Tài nguyên: web service `production-agent` (Docker) và Key Value `agent-redis` (Redis)
- Screenshot: `screenshots/dashboard.png`, `screenshots/running.png`, `screenshots/test.png`
- Chi tiết: xem `DEPLOYMENT.md`

Kết quả kiểm tra URL public:

```bash
BASE=https://production-agent-tqr2.onrender.com
curl -i "$BASE/health"
curl -i "$BASE/ready"
curl -i -X POST "$BASE/ask" -H 'Content-Type: application/json' -d '{"question":"Hello"}'
```
```
HTTP/1.1 200 OK
{"status":"ok","environment":"production"}

HTTP/1.1 200 OK
{"status":"ready"}

HTTP/1.1 401 Unauthorized
{"detail":"Missing or invalid API key"}
```

Có API key:
```bash
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$BASE/ask" -H 'Content-Type: application/json' \
  -H "X-API-Key: $AGENT_API_KEY" -d '{"question":"Hello"}'
```
```
200
```

Script chấm: `python check_production_ready.py --public-url https://production-agent-tqr2.onrender.com` cho kết quả `✅ Public URL: GET https://production-agent-tqr2.onrender.com/health → 200`.

`/ready` trả 200 chứng tỏ agent đã kết nối được Redis trên Render. Nếu `REDIS_URL` sai, endpoint này sẽ trả 503 (tiêu chí 5 của Part 6).


### Exercise 3.3: Pipeline Cloud Run

Đọc `03-cloud-deployment/production-cloud-run/cloudbuild.yaml` và `service.yaml`:

1. **Test:** Cloud Build chạy step `test` trong `python:3.11-slim`: cài `requirements.txt` + `pytest`, chạy `pytest tests/`. Test fail thì pipeline dừng, không build.
2. **Build:** step `build` (`waitFor: [test]`) build Docker image, gắn 2 tag `gcr.io/$PROJECT_ID/ai-agent:$COMMIT_SHA` và `:latest`, dùng `--cache-from ...:latest` để tận dụng layer cache.
3. **Push:** step `push` đẩy cả 2 tag lên Container Registry. Tag theo `$COMMIT_SHA` cho phép biết chính xác commit nào đang chạy và rollback được.
4. **Deploy:** step `deploy` chạy `gcloud run deploy ai-agent` với image theo đúng `$COMMIT_SHA`, region `asia-southeast1`, `--min-instances=1` (tránh cold start), `--max-instances=10` (giới hạn chi phí), 512Mi / 1 CPU, timeout 60 giây. `OPENAI_API_KEY` lấy từ **Secret Manager** (`--set-secrets`), không nằm trong code.
5. **`service.yaml`** khai báo cùng service đó theo kiểu Knative (declarative): autoscale 1–10 instance, `containerConcurrency: 80`, giới hạn tài nguyên, secret qua `secretKeyRef`, `livenessProbe` gọi `/health` và `startupProbe` gọi `/ready`, đúng mô hình liveness/readiness ở Part 5.

Nhận xét: step `test` chạy `pytest tests/`, nhưng cả repo chưa có thư mục `tests/` nào (`requirements.txt` cũng không nằm trong `production-cloud-run/`). Pipeline này sẽ fail ngay bước đầu nếu chạy nguyên trạng: muốn dùng thật phải thêm test, hoặc trỏ tới thư mục test đúng.

---

## Part 4: API Security


### Exercise 4.1: API Key authentication

Bản develop (`04-api-gateway/develop/app.py`) đọc key từ biến `AGENT_API_KEY`, client gửi qua header `X-API-Key`.

```bash
cd 04-api-gateway/develop
AGENT_API_KEY=secret-key-123 python app.py
```

**Không có key → 401**
```bash
curl -i -X POST http://localhost:8001/ask -H "Content-Type: application/json" -d '{"question":"Hello"}'
```
```
HTTP/1.1 401 Unauthorized
server: uvicorn
content-type: application/json

{"detail":"Missing API key. Include header: X-API-Key: <your-key>"}
```

**Sai key → 403**
```bash
curl -i -X POST http://localhost:8001/ask -H "X-API-Key: wrong-key" -H "Content-Type: application/json" -d '{"question":"Hello"}'
```
```
HTTP/1.1 403 Forbidden
server: uvicorn
content-type: application/json

{"detail":"Invalid API key."}
```

**Đúng key → 200**
```bash
curl -i -X POST http://localhost:8001/ask -H "X-API-Key: secret-key-123" -H "Content-Type: application/json" -d '{"question":"Hello"}'
```
```
HTTP/1.1 200 OK
server: uvicorn
content-type: application/json

{"question":"Hello","answer":"Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé."}
```

Nhận xét:
- Bản develop phân biệt **401** (thiếu key, "bạn là ai?") và **403** (có key nhưng sai, "biết bạn nhưng không cho vào").
- Lúc khởi động, app in ra `API Key: ...` trên terminal: vẫn là anti-pattern **log ra secret** như Part 1.
- So sánh key bằng `!=` thường, không dùng `secrets.compare_digest`, nên về lý thuyết có thể bị tấn công timing.
- `/health` và `/` để public vì platform cần gọi health check mà không có key.

### Exercise 4.2: JWT authentication

Bản production (`04-api-gateway/production/`), tài khoản demo `student` / `demo123`.

```bash
cd 04-api-gateway/production
python app.py
```

**Lấy token** (đã che token):
```bash
curl -i -X POST http://localhost:8000/auth/token -H "Content-Type: application/json" \
  -d '{"username":"student","password":"demo123"}'
```
```
HTTP/1.1 200 OK
server: uvicorn
content-type: application/json
x-content-type-options: nosniff
x-frame-options: DENY
x-xss-protection: 1; mode=block
referrer-policy: strict-origin-when-cross-origin

{"access_token":"eyJhbGciOiJIUzI...","token_type":"bearer","expires_in_minutes":60,"hint":"Include in header: Authorization: Bearer eyJhbGciOiJIUzI..."}
```

Lưu token vào biến để dùng tiếp:
```bash
TOKEN=$(curl -s -X POST http://localhost:8000/auth/token -H "Content-Type: application/json" \
  -d '{"username":"student","password":"demo123"}' | python -c "import sys,json;print(json.load(sys.stdin)['access_token'])")
```

**Không có token → 401**
```bash
curl -i -X POST http://localhost:8000/ask -H "Content-Type: application/json" -d '{"question":"Hello"}'
```
```
HTTP/1.1 401 Unauthorized
server: uvicorn
www-authenticate: Bearer
content-type: application/json

{"detail":"Authentication required. Include: Authorization: Bearer <token>"}
```

**Token giả → 403**
```bash
curl -i -X POST http://localhost:8000/ask -H "Authorization: Bearer abc.def.ghi" -H "Content-Type: application/json" -d '{"question":"Hello"}'
```
```
HTTP/1.1 403 Forbidden
server: uvicorn
content-type: application/json

{"detail":"Invalid token."}
```

**Có token → 200**
```bash
curl -i -X POST http://localhost:8000/ask -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" -d '{"question":"Hello"}'
```
```
HTTP/1.1 200 OK
server: uvicorn
content-type: application/json
x-content-type-options: nosniff
x-frame-options: DENY
x-xss-protection: 1; mode=block
referrer-policy: strict-origin-when-cross-origin

{"question":"Hello","answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận.","usage":{"requests_remaining":9,"budget_remaining_usd":0.000212}}
```

Nhận xét:
- JWT tốt hơn API key tĩnh ở chỗ: token **hết hạn sau 60 phút**, mang theo `username` và `role` (server không phải tra database mỗi request), và được ký bằng `JWT_SECRET` nên client không sửa được nội dung.
- Mọi response đều có security header (`X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`...) do middleware `security_headers` thêm vào.
- **Lỗi nhỏ phát hiện được:** field `usage.budget_remaining_usd` trong response `/ask` thực ra trả về `usage.total_cost_usd`, tức số **đã dùng** chứ không phải số **còn lại** (`app.py`, đoạn `return` của `ask_agent`). Endpoint `/me/usage` mới tính đúng `budget_remaining_usd = budget − cost`.

### Exercise 4.3: Rate limiting

Tài khoản `student` (role `user`) bị giới hạn **10 request / 60 giây** 

```bash
for i in $(seq 1 15); do
  echo -n "Request $i: "
  curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:8000/ask \
    -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{"question":"test"}'
done
```
```
Request 1: 200
Request 2: 200
Request 3: 200
Request 4: 200
Request 5: 200
Request 6: 200
Request 7: 200
Request 8: 200
Request 9: 200
Request 10: 429
Request 11: 429
Request 12: 429
Request 13: 429
Request 14: 429
Request 15: 429
```

Chỉ có 9 request `200` vì request `/ask` ở bước 4.2 đã dùng mất 1 lượt trong cùng cửa sổ 60 giây: 1 + 9 = 10 = giới hạn.

**Response 429 đầy đủ:**
```
HTTP/1.1 429 Too Many Requests
server: uvicorn
x-ratelimit-limit: 10
x-ratelimit-remaining: 0
x-ratelimit-reset: 1795681246
retry-after: 54
content-type: application/json

{"detail":{"error":"Rate limit exceeded","limit":10,"window_seconds":60,"retry_after_seconds":54}}
```

Nhận xét:
- Thuật toán là **sliding window**: mỗi user có một `deque` lưu timestamp các request, bỏ những timestamp cũ hơn 60 giây rồi đếm. Cách này không bị "dồn burst ở ranh giới phút" như fixed window.
- Response có `Retry-After` và `X-RateLimit-*` để client biết khi nào được gọi lại, thay vì cứ retry liên tục.
- **Điểm yếu:** state nằm trong RAM của process. Restart là reset bộ đếm, chạy nhiều instance thì mỗi instance đếm riêng (user thực tế được 10 × số instance).

### Exercise 4.4: Cost guard implementation

File `04-api-gateway/production/cost_guard.py`.

1. **Tính chi phí theo token**, giá kiểu GPT-4o-mini (dòng 20–21): $0.15 / 1M token input, $0.60 / 1M token output. App ước lượng token bằng `số từ × 2` (mock, chưa có tokenizer thật).
2. **Hai tầng ngân sách** (dòng 42–43): **$1/ngày cho mỗi user** và **$10/ngày cho toàn hệ thống**.
3. **`check_budget()` gọi TRƯỚC khi gọi LLM** (dòng 60):
   - user vượt budget riêng → **402 Payment Required**
   - tổng hệ thống vượt budget → **503 Service Unavailable**, chặn tất cả mọi người
   - dùng tới **80%** budget (`warn_at_pct`) → ghi log cảnh báo
4. **`record_usage()` gọi SAU khi gọi LLM** (dòng 93): cộng token vào `UsageRecord` của user và cộng tiền vào `_global_cost`.
5. **Reset theo ngày**: `_get_record()` tạo record mới khi `day` khác ngày hôm nay.

Kiểm tra bằng `/me/usage` sau khi chạy các bước trên:
```bash
curl -s http://localhost:8000/me/usage -H "Authorization: Bearer $TOKEN"
```
```
{"user_id":"student","date":"2026-11-26","requests":21,"input_tokens":42,"output_tokens":618,"cost_usd":0.000377,"budget_usd":1.0,"budget_remaining_usd":0.999623,"budget_used_pct":0.0}
```
21 request mới tốn $0.000377, khoảng 0.04% budget, nên muốn thấy 402 thật thì phải hạ `daily_budget_usd` xuống rất thấp.

**Vì sao cần cả rate limit lẫn cost guard?**
- **Rate limit bảo vệ tốc độ:** chặn spam và burst, giữ server không quá tải, chia công bằng tài nguyên giữa các user. Nhưng nó không biết mỗi request tốn bao nhiêu tiền.
- **Cost guard bảo vệ tiền:** một user gửi đều 10 request/phút (không vi phạm rate limit) trong 24 giờ là 14.400 request. Nếu mỗi request là prompt dài vài nghìn token, tiền LLM vẫn có thể vượt ngân sách. Cost guard chặn theo **tổng chi phí tích luỹ**, rate limit không làm được việc này.
- Budget toàn hệ thống ($10/ngày) là lớp bảo vệ cuối: kể cả khi có nhiều tài khoản cùng bị lạm dụng, hoá đơn LLM vẫn có trần.

**Điểm yếu và hướng cải thiện:**
- State trong RAM: restart là mất số liệu chi tiêu, nhiều instance thì mỗi instance tính riêng. Nên lưu vào **Redis** (`INCRBYFLOAT` theo key `cost:<ngày>`, kèm `EXPIRE`).
- `check_budget` chạy trước, `record_usage` chạy sau, nên request cuối cùng có thể làm tổng vượt budget một chút. Muốn chặt hơn thì ước lượng chi phí và trừ trước.
- Token đang ước lượng bằng số từ. Thực tế nên lấy `usage` mà API LLM trả về.

---

## Part 5: Scaling & Reliability

### Exercise 5.1: Health checks

File `05-scaling-reliability/develop/app.py`, chạy trong Git Bash:

```bash
cd 05-scaling-reliability/develop
pip install -r requirements.txt
python app.py
```

**`GET /health` (liveness)**
```bash
curl -i http://localhost:8000/health
```
```
HTTP/1.1 200 OK
server: uvicorn
content-type: application/json

{"status":"degraded","uptime_seconds":0.7,"version":"1.0.0","environment":"development","timestamp":"2026-11-26T08:45:11.938197+00:00","checks":{"memory":{"status":"degraded","used_percent":94.6}}}
```

**`GET /ready` (readiness)**
```bash
curl -i http://localhost:8000/ready
```
```
HTTP/1.1 200 OK
server: uvicorn
content-type: application/json

{"ready":true,"in_flight_requests":1}
```

**`POST /ask`**
```bash
curl -i -X POST "http://localhost:8000/ask?question=Hello"
```
```
HTTP/1.1 200 OK
server: uvicorn
content-type: application/json

{"answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic."}
```

**Liveness khác readiness ở đâu:**

| | `/health` — Liveness | `/ready` — Readiness |
|---|---|---|
| Câu hỏi | "Process còn sống không?" | "Có nên gửi traffic vào instance này không?" |
| Ai gọi | Platform / orchestrator (Railway, Render, K8s `livenessProbe`, Docker `HEALTHCHECK`) | Load balancer / K8s `readinessProbe` |
| Fail thì sao | **Restart** container | **Ngừng route traffic** vào instance, nhưng **không** restart |
| Trả 503 khi | Gần như không bao giờ (process treo thì không trả lời được) | Đang khởi động (`_is_ready = False`), đang shutdown, dependency (DB, Redis) chưa kết nối |
| Nên kiểm tra gì | Nhẹ, nhanh, không phụ thuộc bên ngoài | Các dependency cần để phục vụ request |

Quan sát từ output:
- `/health` báo `"status":"degraded"` vì RAM máy đang dùng 94.6% (> ngưỡng 90% trong code, kiểm bằng `psutil`), **nhưng vẫn trả HTTP 200**. Đây là lựa chọn đúng: restart container không giải quyết được việc máy thiếu RAM, nên liveness không được fail vì lý do này. Trạng thái `degraded` vẫn hiện ra để monitoring cảnh báo.
- `/ready` trả `in_flight_requests: 1`: chính request `/ready` đang được middleware `track_requests` đếm. Bộ đếm này được dùng khi shutdown (5.2).
- Nếu liveness kiểm tra cả Redis/DB thì khi Redis chết, **mọi** instance bị restart liên tục mà không sửa được gì. Kiểm tra dependency phải đặt ở readiness.

### Exercise 5.2: Graceful shutdown

Chạy trong **WSL (Ubuntu-22.04)**. Trên Windows, `kill -TERM` của Git Bash thực chất là giết cứng process (TerminateProcess), không gửi SIGTERM thật.

```bash
cd /mnt/d/.../05-scaling-reliability/develop
source ~/venv-day14/bin/activate

python app.py > /tmp/p5grace.log 2>&1 &
PID=$!
sleep 3

curl -s -X POST "http://localhost:8000/ask?question=Long%20task" &   # gửi request
sleep 0.03
kill -TERM $PID                                                      # SIGTERM ngay khi request đang chạy
wait
```

**Kết quả request:**
```
{"answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận."}  <- HTTP 200
```

**Exit code của process:** `0`

**Log server:**
```
INFO     Starting agent on port 8000
INFO:     Started server process [669]
INFO:     Waiting for application startup.
INFO     Agent starting up...
INFO     Loading model and checking dependencies...
INFO     ✅ Agent is ready!
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
INFO:     127.0.0.1:46770 - "GET /health HTTP/1.1" 200 OK
INFO:     Shutting down                                                  ← nhận SIGTERM
INFO:     127.0.0.1:46774 - "POST /ask?question=Long%20task HTTP/1.1" 200 OK   ← request đang chạy vẫn hoàn thành
INFO:     Waiting for application shutdown.
INFO     🔄 Graceful shutdown initiated...
INFO     ✅ Shutdown complete
INFO:     Application shutdown complete.
INFO:     Finished server process [669]
INFO     Received signal 15 — uvicorn will handle graceful shutdown
```

**Giải thích:** dòng `Shutting down` (uvicorn nhận SIGTERM) xuất hiện **trước** dòng `POST /ask ... 200 OK`. Nghĩa là request bắt đầu trước SIGTERM vẫn được xử lý xong và trả 200, không bị cắt ngang. Luồng tắt:
1. Platform gửi **SIGTERM** (deploy mới, scale down, `docker stop`).
2. Uvicorn **ngừng nhận kết nối mới** nhưng **chờ request đang chạy xong** (`timeout_graceful_shutdown=30`, dòng 198).
3. Phần sau `yield` trong `lifespan` chạy (dòng 52–66): đặt `_is_ready = False` để `/ready` trả 503 và load balancer ngừng gửi traffic. Sau đó chờ `_in_flight_requests` về 0 (tối đa 30 giây), rồi đóng kết nối và dọn dẹp.
4. Process thoát với **exit code 0**. Nếu không xử lý SIGTERM, sau thời gian chờ (Docker mặc định 10 giây) platform gửi **SIGKILL**, request bị cắt và exit code là 137.

Nhận xét: dòng `Received signal 15` của handler tự viết (`handle_sigterm`, dòng 175–187) in ra **cuối cùng**. Lý do là khi chạy, uvicorn cài handler SIGTERM của riêng nó, và chỉ gọi lại handler cũ của app sau khi đã tắt xong. Vì vậy tự đăng ký `signal.signal(SIGTERM, ...)` mà không gọi lại handler của uvicorn là sai: process sẽ không bao giờ tắt. Phần dọn dẹp nên đặt trong `lifespan`.

### Exercise 5.3: Stateless design

File `05-scaling-reliability/production/app.py`:
- Dòng 36–43: kết nối Redis qua `REDIS_URL`. Nếu không kết nối được thì app chuyển sang lưu trong RAM (`USE_REDIS = False`) và log `Storage: In-memory ⚠️`.
- Dòng 62–71: session lưu bằng `_redis.setex(f"session:{session_id}", ttl_seconds, ...)` và đọc lại bằng `_redis.get(...)`, không lưu trong biến Python.
- Dòng 52: mỗi instance có `INSTANCE_ID` riêng, trả về trong field `served_by` để thấy instance nào xử lý request.

**Vì sao state trong RAM hỏng khi chạy nhiều instance:**
- Mỗi instance là một process riêng, có bộ nhớ riêng. Request 1 vào instance A thì history nằm trong RAM của A. Request 2 bị load balancer (round-robin) đẩy sang instance B, mà B không có history đó, nên user bị "mất trí nhớ" giữa cuộc hội thoại.
- Instance bị restart, deploy lại hoặc scale down thì toàn bộ state trong RAM của nó mất theo.
- Nếu né bằng sticky session (luôn gửi một user về một instance) thì instance đó chết vẫn mất dữ liệu, và tải bị lệch giữa các instance.

**Redis giải quyết:** state được đẩy ra một nơi lưu chung. Mọi instance đọc và ghi cùng một key `session:<id>`, nên **bất kỳ instance nào cũng phục vụ được bất kỳ request nào**. Instance trở thành stateless, thêm, bớt hay restart thoải mái. `setex` kèm TTL để session cũ tự hết hạn, không làm đầy Redis.

### Exercise 5.4: Load balancing

```bash
cd 05-scaling-reliability/production
touch .env.local
docker compose -p k5-day14-scaling up -d --build --scale agent=3
docker compose -p k5-day14-scaling ps
```
```
NAME                       STATUS                   PORTS
k5-day14-scaling-agent-1   Up 2 minutes (healthy)   8000/tcp
k5-day14-scaling-agent-2   Up 2 minutes (healthy)   8000/tcp
k5-day14-scaling-agent-3   Up 2 minutes (healthy)   8000/tcp
k5-day14-scaling-nginx-1   Up 2 minutes             0.0.0.0:8080->80/tcp, [::]:8080->80/tcp
k5-day14-scaling-redis-1   Up 2 minutes (healthy)   6379/tcp
```

Stack: `nginx` (cổng **8080** → 80) → `upstream agent_cluster` (`server agent:8000`, Docker DNS trả về IP của cả 3 replica) → `redis`. Chỉ `nginx` mở port ra ngoài.

**Gửi 10 request, xem instance nào xử lý:**
```bash
for i in $(seq 1 10); do
  curl -s -X POST http://localhost:8080/chat -H "Content-Type: application/json" \
    -d "{\"question\": \"Request $i\"}" \
  | python -c "import sys,json; d=json.load(sys.stdin); print('Request', $i, '->', d['served_by'], '|', d['storage'])"
done
```
```
Request 1 -> instance-63b163 | redis
Request 2 -> instance-153b7b | redis
Request 3 -> instance-52f8d5 | redis
Request 4 -> instance-63b163 | redis
Request 5 -> instance-153b7b | redis
Request 6 -> instance-52f8d5 | redis
Request 7 -> instance-63b163 | redis
Request 8 -> instance-153b7b | redis
Request 9 -> instance-52f8d5 | redis
Request 10 -> instance-63b163 | redis
```

Nginx chia **round-robin** đều cho 3 instance theo vòng A → B → C → A... Header response cũng cho thấy backend nào xử lý:
```
HTTP/1.1 200 OK
Server: nginx/1.31.10
X-Served-By: 172.18.0.4:8000
```

**Log agent** (`docker compose -p k5-day14-scaling logs agent`), đã lọc bớt dòng `/health`: request `POST /chat` xuất hiện ở cả `agent-1`, `agent-2`, `agent-3`:
```
agent-1  | INFO:     172.18.0.6:39364 - "POST /chat HTTP/1.1" 200 OK
agent-3  | INFO:     172.18.0.6:50266 - "POST /chat HTTP/1.1" 200 OK
agent-2  | INFO:app:Starting instance instance-153b7b
agent-2  | INFO:app:Storage: Redis ✅
agent-2  | INFO:     172.18.0.6:45138 - "POST /chat HTTP/1.1" 200 OK
```
(`172.18.0.6` là IP của container nginx: agent chỉ nhận request từ nginx.)

**Thử failover: dừng 1 instance**
```bash
docker stop k5-day14-scaling-agent-1
# còn lại: agent-2 (healthy), agent-3 (healthy)
for i in $(seq 1 6); do curl -s -X POST http://localhost:8080/chat ... ; done
docker start k5-day14-scaling-agent-1
```
```
Request 1 -> instance-63b163
Request 2 -> instance-153b7b
Request 3 -> instance-63b163
Request 4 -> instance-153b7b
Request 5 -> instance-63b163
Request 6 -> instance-153b7b
```
Cả 6 request đều trả **200**, traffic tự chuyển sang 2 instance còn sống, client không thấy lỗi. Nginx làm được nhờ `proxy_next_upstream error timeout http_503` và `proxy_next_upstream_tries 3` (`nginx.conf` dòng 23–24): instance nào không kết nối được hoặc trả 503 thì nginx thử tiếp sang instance khác.

### Exercise 5.5: Test stateless

```bash
python test_stateless.py
```
```
============================================================
Stateless Scaling Demo
============================================================

Session ID: eeb0ae5e-67a8-4fd8-a77f-402661edf6fc

Request 1: [instance-52f8d5]
  Q: What is Docker?
  A: Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!...

Request 2: [instance-63b163]
  Q: Why do we need containers?
  A: Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé....

Request 3: [instance-153b7b]
  Q: What is Kubernetes?
  A: Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận....

Request 4: [instance-52f8d5]
  Q: How does load balancing work?
  A: Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé....

Request 5: [instance-63b163]
  Q: What is Redis used for?
  A: Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận....

------------------------------------------------------------
Total requests: 5
Instances used: {'instance-52f8d5', 'instance-63b163', 'instance-153b7b'}
✅ All requests served despite different instances!

--- Conversation History ---
Total messages: 10
  [user]: What is Docker?...
  [assistant]: Container là cách đóng gói app để chạy ở mọi nơi. Build once...
  [user]: Why do we need containers?...
  [assistant]: Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đ...
  [user]: What is Kubernetes?...
  [assistant]: Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã đư...
  [user]: How does load balancing work?...
  [assistant]: Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đ...
  [user]: What is Redis used for?...
  [assistant]: Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã đư...

✅ Session history preserved across all instances via Redis!
```

**Giải thích kết quả:**
- 5 câu hỏi trong **cùng một session** (`eeb0ae5e-...`) được xử lý bởi **cả 3 instance khác nhau** (round-robin).
- History đọc lại có đủ **10 message** (5 user + 5 assistant), đúng thứ tự, dù mỗi lượt do một instance khác ghi. Instance nào cũng đọc được những gì instance khác đã ghi.
- Chứng minh agent **stateless**: state nằm trong Redis chứ không nằm trong RAM của instance. Nếu lưu trong RAM, mỗi instance chỉ thấy phần history của mình, history trả về sẽ thiếu và phụ thuộc vào việc request `/history` rơi vào instance nào.
- Kết hợp với thử nghiệm failover ở 5.4: tắt 1 instance thì session vẫn còn, vì dữ liệu không nằm trong instance đó.

**Dọn dẹp:**
```bash
docker compose -p k5-day14-scaling down
```

---

## Part 6: Final Project — output `check_production_ready.py`

Chạy trong `06-lab-complete/`: bản đầy đủ, có Docker, trong **WSL2 Ubuntu-22.04** (`os.name == "posix"`), Docker Desktop đã bật WSL integration.

```bash
cd 06-lab-complete
source ~/venv-day14/bin/activate
pip install -r requirements.txt
python check_production_ready.py
```

```
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

Một vài số đo trong output chi tiết: image **273 MB** (< 500 MB), build 20 giây. Rate limit trả 429 sau 5 request (`RATE_LIMIT_PER_MINUTE=5` do script đặt). Restart agent vẫn bị 429, chứng tỏ state nằm trong Redis. Cost guard trả 402 ngay request đầu khi budget ≈ 0. Có 17 dòng log, tất cả là JSON hợp lệ. SIGTERM lúc đang có request: request vẫn 200, exit code 0. `docker stop` mất 0.8 giây, exit code 0.

**Những gì đã làm (theo từng file):**

| File | Tiêu chí | Cách làm |
|---|---|---|
| `app/config.py` | 9 | Mọi giá trị đọc bằng `os.getenv` (qua `default_factory`). `AGENT_API_KEY` mặc định rỗng, quên đặt key thì từ chối mọi request |
| `app/auth.py` | 6 | `APIKeyHeader("X-API-Key")`, thiếu hoặc sai đều trả **401**, so sánh bằng `secrets.compare_digest` |
| `app/redis_client.py` | 5 | `redis.from_url` không kết nối lúc import, nên app vẫn lên khi Redis chết. `redis_healthy()` ping với timeout 1 giây |
| `app/rate_limiter.py` | 7 | Sliding window bằng Redis sorted set (`ZREMRANGEBYSCORE` → `ZCARD` → `ZADD` → `EXPIRE`), key là hash của API key, trả 429 kèm `Retry-After` |
| `app/cost_guard.py` | 8 | `INCRBYFLOAT cost:<YYYY-MM-DD>` trong Redis (TTL 2 ngày), `check_budget()` trước khi gọi LLM, vượt `DAILY_BUDGET_USD` trả **402** |
| `app/main.py` | 4, 5, 10, 11 | `/health` luôn 200, `/ready` trả 503 khi Redis không ping được. Logger JSON riêng (`propagate=False`) và middleware log mỗi request. `lifespan` cho shutdown, không tự đăng ký handler SIGTERM (giữ handler của uvicorn). `RedisError` trả 503. Body dùng Pydantic nên JSON hỏng trả 422 |
| `Dockerfile` | 1, 11, 12 | 2 stage `python:3.11-slim` (venv ở builder, copy `/opt/venv` sang runtime), user `appuser` (uid 1000), `HEALTHCHECK` bằng `urllib`, `CMD sh -c "exec uvicorn ... --port ${PORT:-8000} --workers 2"` (`exec` để uvicorn là PID 1) |
| `.dockerignore` | 3 | `.env`, `.env.*` (giữ `.env.example`), `__pycache__/`, `*.pyc`, `venv/`, `.venv/`, `.git/`, `benchmark-observability/` |
| `docker-compose.yml` | 2 | `agent` (build `.`, `REDIS_URL=redis://redis:6379/0`, `depends_on: redis: service_healthy`, `env_file .env` với `required: false`) và `redis:7-alpine` (có healthcheck và volume) |
| `railway.toml` / `render.yaml` | 12 | Builder `DOCKERFILE` + `healthcheckPath = "/health"`. Render: `type: web`, `healthCheckPath: /health`, `AGENT_API_KEY` dùng `generateValue: true`, `REDIS_URL` dùng `sync: false` |

**Vì sao `/health` phải trả 200 khi Redis chết, còn `/ready` phải trả 503:** `/health` (liveness) fail thì platform restart container. Nhưng restart không làm Redis sống lại, chỉ khiến mọi instance restart liên tục. `/ready` (readiness) trả 503 thì load balancer tạm ngừng gửi traffic vào instance cho đến khi Redis kết nối lại, process vẫn sống và tự hồi phục.


---

## AI log

| # | Hỏi AI gì | Nhận được gì | Tự sửa / tự kiểm tra thế nào |
|---|-----------|--------------|------------------------------|
| 1 | Part 1: curl báo `Field required` / `Bad hostname` | Bản develop nhận `question` qua query, PowerShell/cmd xử lý dấu nháy khác bash | sửa lại câu lệnh và chạy thành công |
| 2 | Part 1: bản production trả 500 `JSONDecodeError` | cmd gửi nguyên dấu `'` trong body, cần escape `\"` hoặc dùng Git Bash | sửa lại theo cấu trúc chuẩn|
| 3 | Part 6: script chấm báo ❌ ở tiêu chí 5–9, 11 | Code đúng, lỗi do Docker chưa chạy được trong WSL (`E_UNEXPECTED`). Sửa bằng `wsl --shutdown`, khởi động lại Docker Desktop, bật WSL integration | Chạy lại bản đầy đủ → `PASS 12/12` |
