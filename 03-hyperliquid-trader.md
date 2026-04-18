---
{
  "job_id": "5e8676ac52e7",
  "name": "Hyperliquid Futures Trader",
  "schedule": "every 20m",
  "deliver": "local",
  "skills": [],
  "enabled": false,
  "state": "paused"
}
---

Run the Hyperliquid futures trading pipeline every 30 minutes.

Execute:
```bash
cd ~/trading-pipelines && PYTHONUNBUFFERED=1 python3 -m pipelines hyperliquid run 2>&1 | head -200
```

This scans Top 150 coins by volume on 15m klines, generates quant signals (trend_follow_long, mean_reversion_long only), and opens positions with SL. Short strategies are disabled. Per-side limits are disabled — all 5 slots available to LONG or SHORT.

Credentials are read from ~/wallets.json by exchange_ops.py — no bashrc needed.