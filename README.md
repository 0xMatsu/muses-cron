# Muses Cron Prompts

Cron job prompts extracted for version control. 只读文件，不影响 cron job 本身。

## Jobs

| File | Job ID | Name | Schedule |
|------|--------|------|----------|
| 01-binance-scanner.md | b8d8528b113d | Binance Futures Scanner | 15,30,45,0 * * * * |
| 02-binance-position-manager.md | 214b772fa12e | Binance Position Manager | every 10m |
| 03-hyperliquid-trader.md | 5e8676ac52e7 | Hyperliquid Futures Trader | every 20m |
| 04-hyperliquid-position-manager.md | e421599cbf3b | Hyperliquid Position Manager | every 10m |
| 05-daily-report-morning.md | b835142767e0 | Daily Report Morning | 0 8 * * * |
| 06-daily-report-evening.md | 5b9900595df2 | Daily Report Evening | 0 20 * * * |
| 07-binance-ai-reviewer.md | ab663a974444 | Binance AI Signal Reviewer | 5,10,20,25,35,40,50,55 * * * * |

## Sync
修改 prompt 后需通过 `cronjob(action='update')` 同步回 job，不会自动生效。
