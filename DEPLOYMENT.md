# Thông Tin Deploy — Checkpoint 5

## Thông tin học viên

| Mục | Nội dung |
| --- | --- |
| Họ và tên | Nguyễn Nam Phong |
| Mã học viên | 2A202601320 |
| Repository | `K4-Day12-2A202601320-NguyenNamPhong` |

## Thông tin triển khai

| Mục | Nội dung |
| --- | --- |
| Nền tảng | Render |
| Public URL | https://day12-chat-tjqf.onrender.com |
| Blueprint | `day12-nguyen-nam-phong` |
| Web service | `day12-chat` — Docker — Deployed |
| Data service | `day12-chat-redis` — Render Key Value (Valkey 8) — Available |
| Region | Singapore |
| Nhánh deploy | `main` |
| Ngày deploy | 2026-08-10 |

## Biến môi trường trên Render

Chỉ liệt kê tên biến và nguồn cấu hình; không lưu giá trị bí mật trong repository.

| Biến | Trạng thái | Nguồn |
| --- | --- | --- |
| `PORT` | Đã cấu hình | Render tự cấp |
| `API_TOKEN` | Đã cấu hình | Secret trên Render Dashboard |
| `REDIS_URL` | Đã cấu hình | Internal connection string của Render Key Value |
| `BUCKET_CAPACITY` | Đã cấu hình | `render.yaml` |
| `REFILL_PER_MINUTE` | Đã cấu hình | `render.yaml` |
| `DAILY_BUDGET_USD` | Đã cấu hình | `render.yaml` |
| `LOG_LEVEL` | Đã cấu hình | `render.yaml` |

## Kiểm tra endpoint live

```bash
BASE_URL="https://day12-chat-tjqf.onrender.com"

# 1. Liveness — mong đợi 200 OK
curl -i "$BASE_URL/health"

# 2. Readiness — mong đợi 200 OK và redis=true
curl -i "$BASE_URL/ready"

# 3. Không có API token — mong đợi 401 Unauthorized
curl -i -X POST "$BASE_URL/ask" \
  -H "Content-Type: application/json" \
  -d '{"message":"Hello"}'

# 4. Có API token hợp lệ — mong đợi 200 OK
# DEPLOY_API_TOKEN chỉ đặt trong môi trường local, không ghi vào repository.
curl -i -X POST "$BASE_URL/ask" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $DEPLOY_API_TOKEN" \
  -H "X-Client-Id: cp5-evidence" \
  -d '{"message":"Kiểm tra CP5"}'
```

## Kết quả xác minh thực tế

| Request | Kết quả |
| --- | --- |
| `GET /health` | `200 OK` — service đang hoạt động |
| `GET /ready` | `200 OK` — Redis/Valkey đã kết nối (`redis: true`) |
| `POST /ask` không có token | `401 Unauthorized` |
| `POST /ask` có Bearer token hợp lệ | `200 OK` — trả về nội dung `reply` |

Checkpoint Cloud đã được kiểm tra trực tiếp trên public URL. Token xác thực không xuất hiện trong tài liệu hoặc log.

## Minh chứng

- `screenshots/dashboard.png`: Render Blueprint hiển thị `day12-chat` ở trạng thái **Deployed** và `day12-chat-redis` ở trạng thái **Available**.
- `screenshots/live-endpoints.txt`: log bốn request live, đã loại bỏ token.

## Chạy checkpoint

Trong `.env` local:

```env
LOCAL_FALLBACK=false
DEPLOY_API_TOKEN=<cùng giá trị API_TOKEN đang dùng trên Render>
```

Sau đó chạy:

```bash
pytest tests/test_cp5.py -v
```

Kết quả đã xác minh: **9 passed, 4 skipped**. Bốn test bị bỏ qua thuộc nhánh Local Fallback vì bài sử dụng deployment Cloud thật trên Render.
