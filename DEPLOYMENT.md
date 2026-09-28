# Thông Tin Deploy — Checkpoint 5

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Trịnh Đức Huy |
| Mã học viên | 2A202602865 |
| Repo | https://github.com/huytd2109/K4-L3A-DAY12-TrinhDucHuy-2A202602865-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://agent-production-6ffa.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-09-28 |
| Region | US West (SFO) |

## Biến Môi Trường Đã Set Trên Cloud

Chỉ liệt kê tên biến và nguồn cấp; giá trị secret không nằm trong tài liệu hoặc
repository.

| Biến | Đã set | Nguồn |
|------|--------|-------|
| `PORT` | ✅ | Railway tự cấp lúc chạy |
| `AGENT_API_KEY` | ✅ | Railway service variable, truyền bảo mật qua CLI stdin |
| `REDIS_URL` | ✅ | Reference variable từ Railway Redis service |
| `RATE_LIMIT_PER_MINUTE` | ✅ | Railway service variable |
| `MONTHLY_BUDGET_USD` | ✅ | Railway service variable |
| `LOG_LEVEL` | ✅ | Railway service variable |

## Lệnh Kiểm Tra

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i <URL>/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i <URL>/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST <URL>/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Kiểm tra từ máy cá nhân sau khi Railway báo deployment thành công:

```text
HTTP 200
{"status":"ok","service":"day12-agent","version":"1.0.0"}

HTTP 200
{"status":"ready","redis":true}

HTTP 401
{"detail":"invalid or missing API key"}

HTTP 200
Trả về answer, user_id="sv-test", token và cost_usd.

5. Rate limit
200 200 200 200 200 200 200 200 200 429 429 429 429 429 429
```

Deployment Railway: `8acbbc46-1c21-433c-bf38-fbb18622a393` — `SUCCESS`.

## Ảnh Chụp Màn Hình
- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl
