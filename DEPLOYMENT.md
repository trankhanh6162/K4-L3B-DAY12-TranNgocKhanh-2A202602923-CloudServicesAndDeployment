# Thông Tin Deploy - Checkpoint 5

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Trần Ngọc Khánh |
| Mã học viên | 2A202602923 |
| Repo | https://github.com/trankhanh6162/K4-L3B-DAY12-TranNgocKhanh-2A202602923-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://agent-production-d5c9.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-09-29 |

## Biến Môi Trường Đã Set Trên Cloud

Tài liệu chỉ ghi tên biến và nguồn cấp, không ghi giá trị secret.

| Biến | Đã set | Nguồn |
|------|--------|-------|
| `PORT` | Có | Railway tự gán |
| `AGENT_API_KEY` | Có | Railway Variables, giá trị được đặt qua stdin |
| `REDIS_URL` | Có | Tham chiếu nội bộ đến Railway Redis add-on |
| `RATE_LIMIT_PER_MINUTE` | Có | Railway Variables |
| `MONTHLY_BUDGET_USD` | Có | Railway Variables |
| `LOG_LEVEL` | Có | Railway Variables |

## Lệnh Kiểm Tra

```bash
curl -i https://agent-production-d5c9.up.railway.app/health
curl -i https://agent-production-d5c9.up.railway.app/ready
curl -i -X POST https://agent-production-d5c9.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'
```

Request có xác thực được chạy bằng API key lấy từ secret local, không ghi giá
trị khóa vào tài liệu này.

## Kết Quả Chạy Thật

```text
GET /health
STATUS=200
BODY={"status":"ok","service":"day12-agent","version":"1.0.0"}

GET /ready
STATUS=200
BODY={"status":"ready","redis":true}

POST /ask không có API key
STATUS=401

POST /ask có API key
STATUS=200
BODY có answer, user_id, history_length, cost_usd và tokens.

Rate limit, 12 request liên tiếp với cùng user
200 200 200 200 200 200 200 200 200 200 429 429
```

## Ảnh Chụp Màn Hình

- `screenshots/dashboard.png`: Railway project gồm service `agent` và `Redis`.
- `screenshots/health.png`: kết quả gọi public endpoint `/health`.

## Ghi Chú Vận Hành

- Agent được build từ Dockerfile multi-stage trong repository.
- Redis chạy bằng Railway Redis add-on và chỉ được truy cập qua private network.
- Railway cấp biến `PORT`; Uvicorn bind `0.0.0.0` và đọc cổng này khi khởi động.
- Health check của Railway dùng endpoint `/health`.

## Bonus - CI/CD Với GitHub Actions

- Workflow: `.github/workflows/ci.yml`.
- Chạy tự động khi push hoặc mở pull request vào nhánh `main`.
- Job `test` cài dependency và chạy các test không phụ thuộc bản deploy.
- Job `build` build Docker image trên GitHub runner.
- Job `deploy` chỉ chạy khi `test` và `build` thành công, đồng thời chỉ chạy
  khi push vào `main`.
- Railway token được lưu trong GitHub Actions Secret `RAILWAY_TOKEN`, không nằm
  trong repository.
- Sau deploy, workflow gọi `${{ vars.PUBLIC_URL }}/health` để smoke test.
- GitHub Actions run `CI #1` đã hoàn thành thành công trên nhánh `main`.
- Kết quả kiểm tra local: `pytest tests/test_bonus_cicd.py -v` đạt `13/13` test.
- Badge trạng thái CI được hiển thị ở đầu `README.md` và đang báo `passing`.
