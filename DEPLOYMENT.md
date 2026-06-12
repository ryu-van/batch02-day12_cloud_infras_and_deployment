# Deployment Information

## Public URL
https://batch02-day12cloudinfrasanddeployment-production-6f2f.up.railway.app

## Platform
Railway

## Test Commands

### Health Check
```bash
curl https://batch02-day12cloudinfrasanddeployment-production-6f2f.up.railway.app/health
# Expected: {"status": "ok"}
```

### Readiness Check
```bash
curl https://batch02-day12cloudinfrasanddeployment-production-6f2f.up.railway.app/ready
# Expected: {"ready": true}
```

### API Test (without API key - expected 401)
```bash
curl -X POST https://batch02-day12cloudinfrasanddeployment-production-6f2f.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
```

### API Test (with X-API-Key)
```bash
curl -X POST https://batch02-day12cloudinfrasanddeployment-production-6f2f.up.railway.app/ask \
  -H "X-API-Key: dev-key-change-me" \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
```

## Environment Variables Set
- `ENVIRONMENT`: `production`
- `AGENT_API_KEY`: `dev-key-change-me`
- `JWT_SECRET`: `dev-jwt-secret`
- `PORT`: `8000`
