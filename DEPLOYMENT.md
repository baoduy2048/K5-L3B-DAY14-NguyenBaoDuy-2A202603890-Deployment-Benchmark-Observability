# Deployment Information

## Public URL
https://production-agent-tqr2.onrender.com

## Platform
**Render**: Blueprint `06-lab-complete/render.yaml`, region Singapore, gói free.

| Tài nguyên | Loại | Vai trò |
|---|---|---|
| `production-agent` | Web Service (Docker) | Agent Part 6, build từ `06-lab-complete/Dockerfile` (`rootDir: 06-lab-complete`), healthcheck `GET /health` |
| `agent-redis` | Key Value (Redis) | State dùng chung cho rate limiter và cost guard. Chỉ truy cập được trong mạng nội bộ Render (`ipAllowList: []`) |

## Cách deploy

1. Push repo lên GitHub (`main`).
2. Render Dashboard → **New +** → **Blueprint** → chọn repo → **Blueprint Path** = `06-lab-complete/render.yaml` → **Apply**.
3. Render tạo 2 tài nguyên trên, build image từ Dockerfile và deploy. `autoDeploy: true` nên mỗi lần push lên `main` sẽ deploy lại.

## Test Commands

```bash
BASE=https://production-agent-tqr2.onrender.com
AGENT_API_KEY=<key lấy ở Render Dashboard → production-agent → Environment>
```

### Health Check (liveness)
```bash
curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/health"     # 200
curl "$BASE/health"
# {"status":"ok","environment":"production"}
```

### Readiness (Redis đã kết nối)
```bash
curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/ready"      # 200
curl "$BASE/ready"
# {"status":"ready"}
```

### API Test: không có API key → 401
```bash
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$BASE/ask" \
  -H 'Content-Type: application/json' -d '{"question":"Hello"}'                                   # 401
# {"detail":"Missing or invalid API key"}
```

### API Test: có API key → 200
```bash
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$BASE/ask" \
  -H 'Content-Type: application/json' -H "X-API-Key: $AGENT_API_KEY" \
  -d '{"user_id": "test", "question": "Hello"}'                                                    # 200
```

### Kiểm tra bằng script chấm
```bash
cd 06-lab-complete
python check_production_ready.py --public-url https://production-agent-tqr2.onrender.com
# ✅ Public URL: GET https://production-agent-tqr2.onrender.com/health → 200
```

## Environment Variables Set

Đặt trong `render.yaml` / Render Dashboard. **Không** biến nào có giá trị secret nằm trong repo.

| Biến | Nguồn | Ghi chú |
|---|---|---|
| `AGENT_API_KEY` | `generateValue: true` | Render tự sinh key ngẫu nhiên lúc tạo service |
| `REDIS_URL` | `fromService` → `agent-redis.connectionString` | Render tự điền URL nội bộ của Key Value |
| `ENVIRONMENT` | `production` | |
| `RATE_LIMIT_PER_MINUTE` | `10` | |
| `DAILY_BUDGET_USD` | `1.0` | |
| `PORT` | Render tự cấp | `CMD` trong Dockerfile lắng nghe `${PORT:-8000}` |

## Lưu ý
- Gói free của Render cho service **ngủ sau ~15 phút không có request**. Request đầu tiên sau đó mất khoảng 30–60 giây để container khởi động lại (cold start).
- Key Value gói free không lưu xuống đĩa: Redis restart thì bộ đếm rate limit và chi phí bị reset.

## Screenshots
- [Deployment dashboard](screenshots/dashboard.png)
- [Service running](screenshots/running.png)
- [Test results](screenshots/test.png)
