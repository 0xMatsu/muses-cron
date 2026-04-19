---
{
  "job_id": "5b9900595df2",
  "name": "Daily Report Evening",
  "schedule": "0 20 * * *",
  "deliver": "local",
  "skills": [
    "trading-daily-report"
  ],
  "enabled": true,
  "state": "scheduled"
}
---

[SYSTEM: The user has invoked the "trading-daily-report" skill, indicating they want you to follow its instructions. The full skill content is loaded below.]

---
name: trading-daily-report
description: Generate daily trading reports (morning 8:00 / evening 20:00) covering cron status, recent accomplishments, Binance and Hyperliquid positions, history, parameters, and summaries.
category: trading
---

# Trading Daily Report Generator

按照用户提供的模板生成完整的交易日报，包含 Cron 任务状态、近日完成事项、Binance 和 Hyperliquid 的持仓状态、账户余额、交易回顾、策略参数和总结。

## Report Schedule

| Report Type | Time (UTC+8) | Time Window for History |
|-------------|--------------|------------------------|
| 晨间报告 | 08:00 | 上一日 20:00 - 今日 08:00 |
| 夜晚报告 | 20:00 | 今日 08:00 - 20:00 |

## Report Structure

```markdown
# 交易日报 [晨间/夜晚] - YYYY-MM-DD

## Cron 任务状态
| 任务名 | 开启状态 | 时间间隔 | 最后一次执行 |
|--------|----------|----------|-------------|

## 近日所完成事项
[从 memory 提取]

## Binance
（如果相关任务为关闭状态，写明任务未开启）

**Section 1 — 持仓状态（8 个仓位）**

| 币种 | 方向 | 数量 | 进场价 | 标记价 | 未实现盈亏 | 强平价 | 状态 |
|------|------|------|--------|--------|-----------|--------|------|

**Section 2 — 账户余额**

| 指标 | 数值 |
|------|------|
| USDT 总余额 | |
| 可用余额 | |
| 保证金余额 | |

**Section 3 — 交易回顾**

| 时间 | 币种 | 方向 | 操作 | 盈亏% | 原因 |
|------|------|------|------|-------|------|

**Section 4 — 交易策略参数**

| 参数 | 状态 | 说明 |
|------|------|------|

**Section 5 — 总结**

## Hyperliquid
（如果相关任务为关闭状态，写明任务未开启）

**Section 1 — 持仓状态（8 个仓位）**

| 币种 | 方向 | 数量 | 进场价 | 标记价 | 未实现盈亏 | 强平价 | 状态 |
|------|------|------|--------|--------|-----------|--------|------|

**Section 2 — 账户余额**

| 指标 | 数值 |
|------|------|
| 统一账户 USDC 余额 | |
| Perp Account Value | |

**Section 3 — 交易回顾**

| 时间 | 币种 | 方向 | 操作 | 盈亏% | 原因 |
|------|------|------|------|-------|------|

**Section 4 — 交易策略参数**

| 参数 | 状态 | 说明 |
|------|------|------|

**Section 5 — 总结**
```

## Data Sources

### Cron Jobs
List all trading-related cron jobs with status, schedule, and last execution time.

```bash
cronjob action=list
```

The script contains a static list of known trading jobs. In production, this would query the Hermes cron API.

### Binance Data

**Current Positions:**
```bash
binance-cli futures-usds position-information-v3
```

**Account Balance:**
```bash
binance-cli futures-usds balance
```

**Historical Positions (time window):**
```bash
# Convert time window to milliseconds
# 晨间：上一日 20:00 - 今日 08:00 (12 hours)
# 夜晚：今日 08:00 - 20:00 (12 hours)

binance-cli futures-usds get-income-history --income-type REALIZED_PNL --start-time <START_MS> --end-time <END_MS> --limit 500
```

**Strategy Parameters:**
Read from `~/trading-data/ga/genes.json` and `~/trading-pipelines/pipelines/binance/config.py`:
- `min_score`
- `cooldown_minutes`
- `max_24h_repeat_cap`
- `freq_cap_hours` / `freq_cap_max`
- `sl_from_liq` / `tp_from_liq`
- `max_positions`
- `leverage`

**Trade Log (for matching reasons):**
Read from `~/trading-data/binance/futures_trades.jsonl` — filter by timestamp for time window.

### Hyperliquid Data

**Current Positions:**
```python
from hyperliquid.info import Info
info = Info(skip_ws=True)
state = info.user_state("<wallet_address>")
positions = state["assetPositions"]
```

**Account Balance:**
```python
state = info.spot_user_state("<wallet_address>")
usdc_balance = sum(float(b["total"]) for b in state["balances"] if b["token"] == "USDC")
```

**Historical Positions:**
Read from `~/trading-data/hyperliquid/trades.jsonl` — filter by timestamp for time window.

**Strategy Parameters:**
Read from `~/trading-pipelines/pipelines/hyperliquid/config.py`:
- `min_score` (default: 60)
- `sl_pct` (default: 8.5%)
- `tp_pct` (default: 14.5%)
- `cooldown_minutes` (default: 90)
- `max_positions` (default: 5)
- `leverage` (default: 3)

### Recent Accomplishments (from Memory)

Query memory for recent trading-related accomplishments:
- Pipeline updates/fixes
- Strategy optimizations
- New features deployed
- Bug fixes

Use `--accomplishments` flag to auto-fetch from Hermes `session_search`.

## Usage

```bash
# Morning report (8:00) - auto-fetch accomplishments from session_search
python3 ~/.hermes/skills/trading/trading-daily-report/scripts/gen_daily_report.py morning ~/daily-reports/$(date +%Y-%m-%d)-morning.md --accomplishments

# Evening report (20:00) - auto-fetch accomplishments
python3 ~/.hermes/skills/trading/trading-daily-report/scripts/gen_daily_report.py evening ~/daily-reports/$(date +%Y-%m-%d)-evening.md --accomplishments

# With direct text (semicolon-separated)
python3 ~/.hermes/skills/trading/trading-daily-report/scripts/gen_daily_report.py morning report.md --accomplishments-text "修复了 Binance SL 精度 bug;优化了 Hyperliquid 缓存逻辑"

# From JSON file
python3 ~/.hermes/skills/trading/trading-daily-report/scripts/gen_daily_report.py evening report.md --accomplishments-json /path/to/accomplishments.json

# From stdin (pipe from another command)
echo -e "Fixed bug A\nCompleted feature B" | python3 ~/.hermes/skills/trading/trading-daily-report/scripts/gen_daily_report.py morning report.md --accomplishments-stdin

# Print to stdout (no output path)
python3 ~/.hermes/skills/trading/trading-daily-report/scripts/gen_daily_report.py morning
```

## Accomplishments Sources

| Method | Flag | Use Case |
|--------|------|----------|
| session_search | `--accomplishments` | Auto-fetch from Hermes memory |
| JSON file | `--accomplishments-json <path>` | Pre-curated list from external source |
| stdin | `--accomplishments-stdin` | Pipe from another script/command |
| Direct text | `--accomplishments-text "A;B;C"` | Quick manual entry |

**JSON format:**
```json
["Fixed Binance SL precision bug", "Optimized Hyperliquid candle cache", "Added Telegram notifications"]
```
Or:
```json
{"accomplishments": ["Item 1", "Item 2"]}
```

## Cron Integration

Create Hermes cron job for automated daily reports:

```bash
# Morning at 08:00 UTC+8 (UTC 00:00)
cronjob action=create schedule="0 0 * * *" prompt="Generate morning trading report with accomplishments" name="Daily Report Morning"

# Evening at 20:00 UTC+8 (UTC 12:00)
cronjob action=create schedule="0 12 * * *" prompt="Generate evening trading report with accomplishments" name="Daily Report Evening"
```

Note: Cron schedule uses UTC, so UTC+8 08:00 = UTC 00:00, UTC+8 20:00 = UTC 12:00.

## Output Location

Reports saved to: `~/daily-reports/YYYY-MM-DD-morning.md` or `evening.md`

## Test & Verification

```bash
# Quick test with sample accomplishments
cd ~/.hermes/skills/trading/trading-daily-report/scripts
python3 gen_daily_report.py morning --accomplishments-text "Test item 1;Test item 2"

# Verify output sections:
# ✓ Cron 任务状态 - 6 tasks with status, schedule, last_run
# ✓ 近日所完成事项 - from --accomplishments flag
# ✓ Binance Section 1 - 持仓状态 table format
# ✓ Binance Section 2 - 账户余额 (USDT total/available/margin)
# ✓ Binance Section 3 - 交易回顾 with reasons from local log
# ✓ Binance Section 4 - 策略参数 table
# ✓ Binance Section 5 - 总结 with PnL, WR, trade count
# ✓ Hyperliquid Section 1-5 - same structure
```

**Expected warnings (non-blocking):**
- `[WARN] Binance positions fetch failed` — if binance-cli not configured
- `[WARN] HL positions fetch failed` — if `~/wallets.json` lacks hyperliquid key
- `[WARN] HL history read failed` — if `~/trading-data/hyperliquid/trades.jsonl` doesn't exist

These are handled gracefully with "*当前无开仓*" and "*该时间段内无平仓记录*" fallbacks.

## Pitfalls

1. **Time zone**: Report times are UTC+8, but cron schedules use UTC
2. **API rate limits**: Binance income history limited to 7 days per query
3. **Empty history**: Time windows with no trades should show "*该时间段内无平仓记录*"
4. **Position filtering**: Binance `position-information-v3` returns positions with zero amount — filter these out
5. **Hyperliquid wallet**: Credentials from `~/wallets.json` — never hardcode
6. **Memory query limitations**: Cron jobs run in isolated sessions without direct `session_search` access. Use one of:
   - Hermes agent to generate reports (recommended for cron)
   - Pre-generated JSON file with `--accomplishments-json`
   - Wrapper script with Hermes CLI access
7. **Trade reason matching**: The `match_trade_reason()` function looks for trades within 5 minutes of the income event. If no match is found, defaults to "— " for opens and "主动平仓" for closes.
8. **Python buffering in cron**: Use `PYTHONUNBUFFERED=1 python3 -u` when running in cron to avoid truncated output
9. **Task status display**: If a trading task is paused/disabled, the report should explicitly state "（任务未开启）" in the section header

The user has provided the following instruction alongside the skill invocation: [SYSTEM: You are running as a scheduled cron job. DELIVERY: Your final response will be automatically delivered to the user — do NOT use send_message or try to deliver the output yourself. Just produce your report/output as your final response and the system handles the rest. SILENT: If there is genuinely nothing new to report, respond with exactly "[SILENT]" (nothing else) to suppress delivery. Never combine [SILENT] with content — either report your findings normally, or say [SILENT] and nothing more.]

## 任务：生成晚间交易日报

使用 `trading-daily-report` skill 完成以下工作：

### 1. 生成晚间报告
- 报告类型：`evening`
- 时间窗口：今日 08:00 - 20:00（UTC+8）
- 运行脚本：`python3 -u ~/.hermes/skills/trading/trading-daily-report/scripts/gen_daily_report.py evening ~/daily-reports/$(date +\%Y-\%m-\%d)-evening.md`

### 2. Git 提交并推送
```bash
cd ~/daily-reports
git add -A
git commit -m "Daily evening report $(date +\%Y-\%m-\%d)"
git push origin main
```

### 3. 推送通知到 Telegram
通过 send_message 推送简短通知到 Telegram：
"📊 晚间交易日报已生成 (YYYY-MM-DD) | 盈亏: +$XXX | Git 已提交。"
其中 +$XXX 从报告的"综合概览"中提取时段总计。

### 注意事项
- API 错误正常处理，不要中断流程
- 如果 git push 失败（远程更新），先 pull --rebase 再 push
