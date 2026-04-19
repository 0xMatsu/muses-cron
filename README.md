# Muses Cron Prompts

Version-controlled cron job prompts for Muses trading system.

## Sync

```bash
# Pull latest from Hermes
cd ~/muses-crons && python3 sync_pull.py

# Push changes to Hermes (edit file first, then):
python3 sync_push.py
```

## Jobs

| # | Job | Schedule | Status |
|---|-----|----------|--------|
| | Binance AI Signal Reviewer | `{'kind': 'cron', 'expr': '5,10,20,25,35,40,50,55 * * * *', 'display': '5,10,20,25,35,40,50,55 * * * *'}` | :white_check_mark: |
| | Binance Futures Scanner | `{'kind': 'cron', 'expr': '15,30,45,0 * * * *', 'display': '15,30,45,0 * * * *'}` | :white_check_mark: |
| | Binance Position Manager | `{'kind': 'interval', 'minutes': 10, 'display': 'every 10m'}` | :white_check_mark: |
| | Daily Report Evening | `{'kind': 'cron', 'expr': '0 20 * * *', 'display': '0 20 * * *'}` | :white_check_mark: |
| | Daily Report Morning | `{'kind': 'cron', 'expr': '0 8 * * *', 'display': '0 8 * * *'}` | :white_check_mark: |
| | Hyperliquid AI Trader | `{'kind': 'interval', 'minutes': 20, 'display': 'every 20m'}` | :white_check_mark: |
