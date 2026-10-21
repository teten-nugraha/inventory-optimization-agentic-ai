# Mock Services — Inventory Optimization Agent API

Mock backend untuk Inventory Optimization AI Agent (retail fashion). Seluruh sistem (inventory, sales, forecast, optimization, approval, stock transfer) di-mock agar agent bisa dikembangkan & diuji tanpa integrasi eksternal.

**Prinsip:** *AI orchestrates; domain services calculate; humans approve consequential actions.*

## Stack

| Komponen | Teknologi |
|---|---|
| API | Go 1.24 (`net/http` + `pgx/v5`) |
| Database | PostgreSQL 16 |
| Orchestration | Docker Compose |

## Struktur

```
mock-services/
  cmd/api/main.go                  entrypoint (waitForDB anti-race saat seed)
  internal/api/api.go              routing + 11 handler
  internal/store/store.go          query pgx ke 14 tabel
  internal/model/model.go          struct JSON
  internal/forecast/forecast.go    forecast moving-average (deterministik)
  internal/optimizer/optimizer.go  greedy allocation (deterministik)
  internal/idgen/idgen.go          generator ID
  db/migrations/0001_init.sql      DDL 14 tabel
  db/seed/seed.sql                 data sintetis 12 store × 1.440 SKU × 120 hari
  docker-compose.yml
  Dockerfile
```

## Menjalankan

```bash
docker compose up -d --build
```

- API: `http://localhost:8081`
- PostgreSQL: `localhost:5432` (db `inventory`, user/pass `postgres`)
- **First run** butuh ~2 menit karena seed memasukkan ~800 ribu baris `sales_daily`. DDL + seed otomatis ter-load saat volume kosong.

Cek kesiapan:

```bash
curl http://localhost:8081/healthz
# {"status":"ok"}
```

Hentikan (hapus data sekaligus):

```bash
docker compose down -v
```

Akses DB langsung:

```bash
docker compose exec db psql -U postgres -d inventory
```

---

## Daftar API

Base path `/v1`. Write endpoint hanya satu (`create_transfer_draft`).

| # | Endpoint | Tool | Permission |
|---|---|---|---|
| 1 | `GET /v1/stock` | get_stock | read |
| 2 | `GET /v1/sales` | get_sales | read |
| 3 | `GET /v1/products/{sku_id}` | get_product | read |
| 4 | `GET /v1/transfer-rules` | get_transfer_rules | read |
| 5 | `POST /v1/forecasts` | forecast_demand | compute |
| 6 | `POST /v1/recommendations/optimize` | optimize_transfers | compute |
| 7 | `GET /v1/recommendations/{id}/explain` | explain_recommendation | read |
| 8 | `POST /v1/recommendations/{id}/approve` | approve (workflow) | write |
| 9 | `POST /v1/recommendations/{id}/reject` | reject (workflow) | write |
| 10 | `POST /v1/transfers/drafts` | create_transfer_draft | **write** |
| 11 | `GET /v1/transfers/drafts/{draft_id}` | get_transfer_status | read |

---

## Contoh cURL

### 1. Health

```bash
curl http://localhost:8081/healthz
```

### 2. Get stock (snapshot terakhir per store-SKU)

```bash
curl -G http://localhost:8081/v1/stock \
  --data-urlencode "store_ids=S001,S002" \
  --data-urlencode "sku_ids=STYLE-0003-White-L" \
  --data-urlencode "as_of=2026-10-16T00:00:00Z"
```

`store_ids` / `sku_ids` opsional (kosongkan untuk semua). `as_of` opsional (default sekarang).

### 3. Get sales

```bash
curl -G http://localhost:8081/v1/sales \
  --data-urlencode "store_ids=S004" \
  --data-urlencode "sku_ids=STYLE-0001-Black-M" \
  --data-urlencode "from=2026-10-01" \
  --data-urlencode "to=2026-10-16"
```

### 4. Get product

```bash
curl http://localhost:8081/v1/products/STYLE-0003-White-L
```

### 5. Get transfer rules

```bash
curl -G http://localhost:8081/v1/transfer-rules \
  --data-urlencode "source_store=S002" \
  --data-urlencode "target_store=S004"
```

### 6. Forecast demand (deterministik)

```bash
curl -X POST http://localhost:8081/v1/forecasts \
  -H "Content-Type: application/json" \
  -d '{"store_id":"S004","sku_id":"STYLE-0001-Black-M","horizon_days":7}'
```

### 7. Optimize transfers (deterministik)

```bash
curl -X POST http://localhost:8081/v1/recommendations/optimize \
  -H "Content-Type: application/json" \
  -d '{
    "scope": {
      "store_ids": ["S001","S002","S003","S004","S005","S006","S007","S008","S009","S010","S011","S012"],
      "sku_ids": ["STYLE-0003-White-L"]
    },
    "constraints": {"target_cover_days": 7}
  }'
```

### 8. Explain recommendation

```bash
curl http://localhost:8081/v1/recommendations/<RECOMMENDATION_ID>/explain
```

### 9. Approve recommendation

```bash
curl -X POST http://localhost:8081/v1/recommendations/<RECOMMENDATION_ID>/approve \
  -H "Content-Type: application/json" \
  -d '{"actor_id":"planner1","reason":"valid need"}'
```

Respon berisi `approval_token` yang dipakai saat membuat draft.

### 10. Reject recommendation

```bash
curl -X POST http://localhost:8081/v1/recommendations/<RECOMMENDATION_ID>/reject \
  -H "Content-Type: application/json" \
  -d '{"actor_id":"planner1","reason":"margin terlalu tipis"}'
```

### 11. Create transfer draft (WRITE — wajib approve)

```bash
curl -X POST http://localhost:8081/v1/transfers/drafts \
  -H "Content-Type: application/json" \
  -d '{
    "recommendation_id": "<RECOMMENDATION_ID>",
    "approval_token": "<APPROVAL_TOKEN>",
    "idempotency_key": "my-unique-key-001"
  }'
```

`idempotency_key` mencegah draft duplikat — request kedua dengan key sama mengembalikan draft yang sama.

### 12. Get transfer status

```bash
curl http://localhost:8081/v1/transfers/drafts/<DRAFT_ID>
```

---

## Alur End-to-End (shell)

```bash
BASE=http://localhost:8081

# 1. Optimize -> ambil rekomendasi pertama
REC=$(curl -s -X POST "$BASE/v1/recommendations/optimize" \
  -H "Content-Type: application/json" \
  -d '{
    "scope": {
      "store_ids": ["S001","S002","S003","S004","S005","S006","S007","S008","S009","S010","S011","S012"],
      "sku_ids": ["STYLE-0003-White-L"]
    },
    "constraints": {"target_cover_days": 7}
  }' | python3 -c "import sys,json; print(json.load(sys.stdin)['recommendations'][0]['id'])")

echo "Rekomendasi: $REC"

# 2. Lihat alasan (evidence)
curl -s "$BASE/v1/recommendations/$REC/explain" | python3 -m json.tool

# 3. Approve -> dapatkan token
TOKEN=$(curl -s -X POST "$BASE/v1/recommendations/$REC/approve" \
  -H "Content-Type: application/json" \
  -d '{"actor_id":"planner1","reason":"valid need"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['approval_token'])")

# 4. Buat draft transfer
curl -s -X POST "$BASE/v1/transfers/drafts" \
  -H "Content-Type: application/json" \
  -d "{\"recommendation_id\":\"$REC\",\"approval_token\":\"$TOKEN\",\"idempotency_key\":\"key-$REC\"}"
```

## Skenario Data

12 store masing-masing punya skenario retail berbeda (fast mover, overstock donor, seasonal spike, promo weekend, stockout, cold start, high returns, broken size run, inbound coverage, healthy control, high-value slow, intermittent). Detail di `../../docs/scenario.md`.

## Catatan

- Port host di-mapping ke `8081` karena `8080` mungkin sudah dipakai.
- `approval_token` berasal dari endpoint approve, bukan output LLM. Write ditegakkan server-side.
- Forecast & optimizer deterministik (bukan LLM): moving-average 28 hari + greedy allocation.
# inventory-optimization-agentic-ai
