# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Nguyễn Việt Hưng |
| Mã học viên | 2A202601275 |
| Repo | https://github.com/hungnv-2811/K3-Day12-2A202601275-NguyenVietHung |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://agent-production-af92.up.railway.app |
| Platform | Railway (build từ `Dockerfile`, service `agent` + database `Redis` riêng biệt trong cùng project) |
| Ngày deploy | 2026-08-10 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | Railway tự gán, app đọc qua `${PORT:-8000}` trong `CMD` của Dockerfile |
| `AGENT_API_KEY` | ✅ | đặt qua `railway variables --set`, không nằm trong repo |
| `REDIS_URL` | ✅ | tham chiếu `${{Redis.REDIS_URL}}` tới database Redis add-on của Railway (cùng project, network nội bộ `railway.internal`) |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

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

Chạy trên `https://agent-production-af92.up.railway.app` (Railway, deploy thật):

```
PS> curl.exe -i "https://agent-production-af92.up.railway.app/health"
HTTP/1.1 200 OK
Content-Type: application/json
{"status":"ok","service":"day12-agent","version":"1.0.0"}

PS> curl.exe -i "https://agent-production-af92.up.railway.app/ready"
HTTP/1.1 200 OK
Content-Type: application/json
{"status":"ready","redis":true}

PS> Invoke-RestMethod -Uri "https://agent-production-af92.up.railway.app/ask" -Method Post `
      -ContentType "application/json" -Body (@{question="Hello"} | ConvertTo-Json)
Invoke-RestMethod : {"detail":"invalid or missing API key"}
    + CategoryInfo          : InvalidOperation: (...) [Invoke-RestMethod], WebException
    + FullyQualifiedErrorId : WebCmdletWebResponseException,...
```

(`Invoke-RestMethod` ném exception khi HTTP không phải 2xx — đây là hành vi
đúng, thông báo lỗi bên trong xác nhận server trả `401 invalid or missing API key`
như mong đợi khi không có `X-API-Key`.)

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — kết quả `docker compose ps` lúc kiểm thử local trước khi deploy cloud (agent + redis đều healthy)
- `screenshots/health.png` — kết quả gọi `/health` lúc kiểm thử local

---

## Nếu Dùng Phương Án Dự Phòng

Không áp dụng — đã deploy thành công lên Railway thật (xem mục Service và Kết
Quả Chạy Thật ở trên). Trước khi deploy thành công, có kiểm thử qua Docker
Compose ở máy (phương án dự phòng) để xác nhận code chạy đúng; chi tiết quá
trình khắc phục sự cố Docker Desktop / Railway CLI được ghi trong câu 10 của
`exercises.md`.
