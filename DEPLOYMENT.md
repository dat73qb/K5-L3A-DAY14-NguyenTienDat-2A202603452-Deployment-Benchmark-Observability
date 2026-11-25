# Deployment Information

## Public URL
https://k5-l3a-day14-nguyentiendat-2a202603452.onrender.com

## Platform
Render (Web Service - Docker + Managed Redis / Key Value)

## Test Commands

### 1. Health & Readiness Check
```bash
curl -sS -o /dev/null -w '%{http_code}\n' https://k5-l3a-day14-nguyentiendat-2a202603452.onrender.com/health
# Expected: 200

curl -sS -o /dev/null -w '%{http_code}\n' https://k5-l3a-day14-nguyentiendat-2a202603452.onrender.com/ready
# Expected: 200
```

### 2. Authentication Check (without API key)
```bash
curl -sS -o /dev/null -w '%{http_code}\n' \
  -X POST https://k5-l3a-day14-nguyentiendat-2a202603452.onrender.com/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
# Expected: 401
```

### 3. API Test (with authentication)
```bash
curl -X POST https://k5-l3a-day14-nguyentiendat-2a202603452.onrender.com/ask \
  -H "X-API-Key: YOUR_AGENT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"user_id": "test", "question": "Hello"}'
# Expected: 200
```

## Environment Variables Set
- `PORT` (injected automatically by Render)
- `REDIS_URL` (Internal Redis URL from Render Key Value / Redis)
- `AGENT_API_KEY` (secret, set on Render dashboard)
- `ENVIRONMENT=production`
- `RATE_LIMIT_PER_MINUTE=10`
- `DAILY_BUDGET_USD=1.0`

## Screenshots
- [Deployment dashboard](screenshots/dashboard.png)
- [Service running](screenshots/running.png)
- [Test results](screenshots/test.png)
