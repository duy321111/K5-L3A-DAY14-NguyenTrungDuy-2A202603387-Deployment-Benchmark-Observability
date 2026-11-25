# Deployment Information

## Public URL
https://day14-ai-agent-ug77.onrender.com

## Platform
Render (Web Service — Docker runtime + Render Key Value / Redis Free tier, Root Directory: `06-lab-complete`)

## Test Commands

### 1. Health Check (Liveness)
```bash
curl -sS https://day14-ai-agent-ug77.onrender.com/health
# Expected: HTTP 200 — {"status": "ok", "environment": "production"}
```

### 2. Readiness Check (Redis connection)
```bash
curl -sS https://day14-ai-agent-ug77.onrender.com/ready
# Expected: HTTP 200 — {"status": "ready"}
```

### 3. Authentication Check (without API key → 401)
```bash
curl -i -X POST https://day14-ai-agent-ug77.onrender.com/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
# Expected: HTTP 401
```

### 4. API Test (with API key → 200)
```bash
curl -i -X POST https://day14-ai-agent-ug77.onrender.com/ask \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"user_id": "test", "question": "Hello"}'
# Expected: HTTP 200
```

### 5. Quick 4-line Self-Test (CP6)
```bash
BASE="https://day14-ai-agent-ug77.onrender.com"
curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/health"   # 200
curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/ready"    # 200
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$BASE/ask" -H 'Content-Type: application/json' -d '{"question":"Hello"}'   # 401
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$BASE/ask" -H 'Content-Type: application/json' -H "X-API-Key: $AGENT_API_KEY" -d '{"question":"Hello"}'   # 200
```

## Environment Variables Set
Configured on Render Dashboard (not committed to Git):
- `ENVIRONMENT` (`production`)
- `LOG_LEVEL` (`INFO`)
- `REDIS_URL` (Internal Render Key Value URL)
- `AGENT_API_KEY` (secret, set via Render Environment Variables)
- `RATE_LIMIT_PER_MINUTE` (`10`)
- `DAILY_BUDGET_USD` (`10.0`)

## Screenshots
- [Deployment dashboard](screenshots/dashboard.png)
- [Service running](screenshots/running.png)
- [Test results](screenshots/test.png)
