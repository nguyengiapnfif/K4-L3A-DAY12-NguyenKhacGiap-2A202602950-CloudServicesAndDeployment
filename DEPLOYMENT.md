# Thông Tin Deploy — Checkpoint 5

> `pytest tests/test_cp5.py` đọc file này để tìm địa chỉ service và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Nguyễn Khắc Giáp |
| Mã học viên | 2A202602950 |
| Repo | https://github.com/nguyengiapnfif/K4-L3A-DAY12-NguyenKhacGiap-2A202602950-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://gallant-rejoicing-production-2c9e.up.railway.app |
| Platform | Railway (build từ `Dockerfile` theo `railway.toml`) |
| Ngày deploy | 2026-09-28 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Redis service trong cùng project Railway, tham chiếu `${{Redis.REDIS_URL}}` (mạng nội bộ) |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

```bash
URL=https://gallant-rejoicing-production-2c9e.up.railway.app

# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i $URL/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i $URL/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST $URL/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST $URL/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-doc" \
  -d '{"question":"What is deploy?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST $URL/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-doc-rl" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Chạy ngày 2026-09-28 (status line + body):

```
$ curl -i $URL/health
HTTP/1.1 200 OK
{"status":"ok","service":"day12-agent","version":"1.0.0"}

$ curl -i $URL/ready
HTTP/1.1 200 OK
{"status":"ready","redis":true}

$ curl -i -X POST $URL/ask   (không có API key)
HTTP/1.1 401 Unauthorized
{"detail":"invalid or missing API key"}

$ curl -i -X POST $URL/ask   (có API key)
HTTP/1.1 200 OK
{"answer":"Với What is deploy, cách làm phổ biến trong production là đặt một lớp gateway phía trước để lo authentication, rate limiting và bảo vệ chi phí.","user_id":"sv-doc","history_length":0,"cost_usd":2.145e-05,"tokens":{"in":3,"out":35}}

$ rate limit: 15 lần POST /ask
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

Ghi chú: lệnh 4 dùng câu hỏi không dấu vì `curl` trong Git Bash trên Windows
gửi chuỗi tiếng Việt sai encoding (không phải UTF-8) nên server trả
`400 There was an error parsing the body`. Gửi cùng câu "Deploy là gì?" bằng
Python `httpx` (UTF-8) thì service trả 200 bình thường.

## Ảnh Chụp Màn Hình

- `screenshots/dashboard.png` — trang quản lý service trên Railway
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt
