# Thong Tin Deploy - Checkpoint 5

## Thong Tin Hoc Vien

| Muc | Noi dung |
|-----|----------|
| Họ và tên | Trần Ngọc Khánh |
| Mã học viên | 2A202602923 |
| Repo | https://github.com/trankhanh6162/K4-L3B-DAY12-TranNgocKhanh-2A202602923-CloudServicesAndDeployment |

## Service

| Muc | Noi dung |
|-----|----------|
| Public URL | https://agent-production-d5c9.up.railway.app |
| Platform | Railway |
| Ngay deploy | 2026-09-29 |

## Bien Moi Truong Da Set Tren Cloud

Tai lieu chi ghi ten bien va nguon cap, khong ghi gia tri secret.

| Bien | Da set | Nguon |
|------|--------|-------|
| `PORT` | Co | Railway tu gan |
| `AGENT_API_KEY` | Co | Railway Variables, gia tri duoc dat qua stdin |
| `REDIS_URL` | Co | Tham chieu noi bo den Railway Redis add-on |
| `RATE_LIMIT_PER_MINUTE` | Co | Railway Variables |
| `MONTHLY_BUDGET_USD` | Co | Railway Variables |
| `LOG_LEVEL` | Co | Railway Variables |

## Lenh Kiem Tra

```bash
curl -i https://agent-production-d5c9.up.railway.app/health
curl -i https://agent-production-d5c9.up.railway.app/ready
curl -i -X POST https://agent-production-d5c9.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'
```

Request co xac thuc duoc chay bang API key lay tu secret local, khong ghi gia
tri khoa vao tai lieu nay.

## Ket Qua Chay That

```text
GET /health
STATUS=200
BODY={"status":"ok","service":"day12-agent","version":"1.0.0"}

GET /ready
STATUS=200
BODY={"status":"ready","redis":true}

POST /ask khong co API key
STATUS=401

POST /ask co API key
STATUS=200
BODY co answer, user_id, history_length, cost_usd va tokens.

Rate limit, 12 request lien tiep voi cung user
200 200 200 200 200 200 200 200 200 200 429 429
```

## Anh Chup Man Hinh

- `screenshots/dashboard.png`: Railway project gom service `agent` va `Redis`.
- `screenshots/health.png`: ket qua goi public endpoint `/health`.

## Ghi Chu Van Hanh

- Agent duoc build tu Dockerfile multi-stage trong repository.
- Redis chay bang Railway Redis add-on va chi duoc truy cap qua private network.
- Railway cap bien `PORT`; Uvicorn bind `0.0.0.0` va doc cong nay khi khoi dong.
- Health check cua Railway dung endpoint `/health`.
