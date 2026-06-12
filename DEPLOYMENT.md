# Deployment Information

## Public URL
https://perfect-art-production-d49b.up.railway.app

## Platform
Railway

## Test Commands

### Health Check
```bash
curl https://perfect-art-production-d49b.up.railway.app/health
# Expected: {"status": "ok"}
```

### Readiness Check
```bash
curl https://perfect-art-production-d49b.up.railway.app/ready
# Expected: {"ready": true}
```

### API Test (without API key - expected 401)
```bash
curl -X POST https://perfect-art-production-d49b.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
```

### API Test (with X-API-Key)
```bash
curl -X POST https://perfect-art-production-d49b.up.railway.app/ask \
  -H "X-API-Key: dev-key-change-me" \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
```

## Environment Variables Set
- `ENVIRONMENT`: `production`
- `AGENT_API_KEY`: `dev-key-change-me`
- `JWT_SECRET`: `dev-jwt-secret`
- `PORT`: `8000`

## Run Streamlit GUI Dashboard (Local)
To launch the interactive Enterprise HR ReAct Agent Workspace interface locally:
```bash
# Activate your virtual environment and run streamlit
.venv\Scripts\python -m streamlit run src/Gui/app.py
```
This will open the dashboard UI at http://localhost:8501.

