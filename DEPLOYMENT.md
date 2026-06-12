# Deployment Information

## Public URL
https://distinguished-reverence-production-a4e9.up.railway.app

## Platform
Railway

## Test Commands

### Health Check
```bash
curl https://distinguished-reverence-production-a4e9.up.railway.app/health
# Expected: {"status":"ok","uptime_seconds":...}
```

### API Test (without auth / simple ask)
```bash
curl -X POST https://distinguished-reverence-production-a4e9.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
```

## Environment Variables Set
- `PORT` (Injected automatically by Railway)
- `AGENT_API_KEY` (Not used/required in simple railway deployment example)

## Screenshots
- [Deployment dashboard](screenshots/dashboard.png)
- [Service running](screenshots/running.png)
- [Test results](screenshots/test.png)
