---
{
  "job_id": "e421599cbf3b",
  "name": "Hyperliquid Position Manager",
  "schedule": "every 10m",
  "deliver": "local",
  "skills": [],
  "enabled": false,
  "state": "paused"
}
---

Run the Hyperliquid position manager every 15 minutes.

Execute:
```bash
cd ~/trading-pipelines && PYTHONUNBUFFERED=1 python3 -m pipelines hyperliquid manage_live 2>&1
```

This manages open Hyperliquid positions:
- Tiered trailing stops
- Partial close at +10%
- Signal decay re-scan
- Time stop (>8h and PnL < 0)

If there are errors, report them. If no positions or success, local delivery only.