---
job_id: 214b772fa12e
schedule: {'kind': 'interval', 'minutes': 10, 'display': 'every 10m'}
name: Binance Position Manager
skills: []
status: active
---
Run the Binance futures position manager every 5 minutes. This script handles trailing stops, dynamic TP/SL, timeout exits, and partial take-profit tiers — all autonomously. NO AI review needed; the PM executes all close/partial-close decisions directly.

## Step 1: Run position manager

```bash
cd ~/trading-pipelines && PYTHONUNBUFFERED=1 python3 -u -m pipelines.binance.trading.position_manager 2>&1 | tail -80
```

## Response

If positions exist, report status briefly:
- List each position: symbol, side, PnL%, SL status
- If PM took any actions (close/partial/trail SL update), report them
- If no positions: report "No open positions"

If the PM ran normally with no notable changes, respond with exactly "[SILENT]" to suppress delivery.