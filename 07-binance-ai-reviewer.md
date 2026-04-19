---
{
  "job_id": "ab663a974444",
  "name": "Binance AI Signal Reviewer",
  "schedule": "5,10,20,25,35,40,50,55 * * * *",
  "deliver": "local",
  "skills": [
    "binance-futures-trader"
  ],
  "enabled": true,
  "state": "scheduled"
}
---

[SYSTEM: The user has invoked the "binance-futures-trader" skill, indicating they want you to follow its instructions. The full skill content is loaded below.]

---
name: binance-futures-trader
description: Live futures trading pipeline for demon coins on Binance USDT-M perp — Hedge Mode, liquidation-aware SL/TP, position management, auto-optimization
category: trading
---

# Binance Futures Demon Trader

Live trading pipeline that scans for demon coins and executes perpetual futures trades on Binance.

## Quick Start

```bash
# Run pipeline scan (every 15m cron)
cd ~/trading-pipelines/pipelines && PYTHONUNBUFFERED=1 PYTHONPATH=.:.. python3 -u -m binance.trading.futures_pipeline 2>&1 | head -200

# Position manager (every 5m cron)
cd ~/trading-pipelines && PYTHONUNBUFFERED=1 python3 -u -m pipelines.binance.trading.position_manager 2>&1 | head -100

# Dry-run to see signals when positions full
cd ~/trading-pipelines/pipelines && PYTHONPATH=. python3 -c "
from binance.trading.futures_pipeline import *
from binance.data.market_data import get_24h_stats; stats = get_24h_stats(); settling = check_settling()
extreme = [s for s in stats if abs(stats[s]['priceChangePercent']) > 20 and s not in settling]
for sym in extreme[:15]:
    r = analyze_symbol(sym, stats)
    if r and r['score'] >= 30: print(f'{sym:<20} score={r[\"score\"]:>3} {r[\"direction\"]} 24h={r[\"pct_24h\"]:+.1f}%')
"
```

## File Paths

| What | Path |
|------|------|
| Pipeline code | `~/trading-pipelines/pipelines/binance/trading/futures_pipeline.py` |
| Position manager | `~/trading-pipelines/pipelines/binance/trading/position_manager.py` |
| Notifications | `~/trading-pipelines/pipelines/binance/core/notifications.py` |
| Data files | `~/trading-data/` (cooldown.json, futures_trades.jsonl, pending_signals.json, etc.) |
| Config | `config.py` within the binance package (~60 named constants) |
| Strategies | `futures_strategy.json` in DATA_DIR (min_score, cooldown, caps) |
| Optuna search | `~/trading-pipelines/pipelines/binance/optimize/optuna_search.py` |
| Git repo | `git@github.com:0xMatsu/trading-pipelines.git` |

## Config Values (verified 2026-04-18, live run)

| Parameter | Value | Notes |
|-----------|-------|-------|
| MAX_POSITIONS | 8 (实盘=回测=8) | Config confirmed 2026-04-18 |
| LEVERAGE | 3x | Live confirmed |
| MARGIN_RANGE | (80, 100) | DYNAMIC via calc_margin, $80-100 |
| SL_FROM_LIQ | 0.4 | SL = 清算距离 × 40% |
| TP_FROM_LIQ | 0.8 | DYNAMIC via position_manager, NOT algo order |
| FREQ_CAP_HOURS | 8 | Optuna-optimized |
| FREQ_CAP_MAX | 10 | Max opens per 6h window |
| COOLDOWN_MINUTES | 15 | Base cooldown |
| COOLDOWN_REPEAT_PENALTY | 90 | If 2+ of last 5 closes were losses |

**Rule:** ALL thresholds live in `config.py`. NEVER hardcode strategy values in modules.

## Hedge Mode (CRITICAL)

**Account is in Hedge Mode** (`dualSidePosition: true`). ALL orders MUST include `--position-side`:

| Operation | `--side` | `--position-side` |
|-----------|----------|-------------------|
| Open LONG | BUY | LONG |
| Close LONG | SELL | LONG |
| Open SHORT | SELL | SHORT |
| Close SHORT | BUY | SHORT |

**open_position()** signature: `open_position(sym, side, signal, margin_usdt)` — uses `margin_usdt` NOT `margin`.

**close_position()** signature: `close_position(sym, reason="", record_pnl=True)` — does NOT accept `pos_info` parameter.

**SL/TP via new-algo-order:** use `binance-cli futures-usds new-algo-order -i --json` with JSON stdin. `triggerPrice` (not `stopPrice`), `closePosition: true`, positionSide = the position being CLOSED.

## Signal Review Data (built into pipeline)

Pipeline's `_enrich_signal()` fetches real-time data inline (orderbook, trades, 5m klines, OI, funding). Each signal in `pending_signals.json` has a `review` key:

```json
{
  "review": {
    "current_price": 0.16607,
    "momentum_5m": 1.2,
    "funding_rate": 0.0001,
    "orderbook": {"spread_pct": 0.05, "ask_pressure": false, "bid_support": true},
    "recent_trades": {"buy_ratio": 0.65, "micro_momentum_pct": 0.3},
    "conflicts": [],
    "conflict_count": 0,
    "suggestion": "OK — 实时数据与信号方向一致，可以入场"
  }
}
```

**SKIP** if `conflict_count >= 2` or `pipeline_score < 15` or already holding this symbol.
**CAUTION** if `conflict_count == 1` — reduce margin 50%.
**OPEN** if `conflict_count == 0` AND `pipeline_score >= 15`.

**Before opening:** re-run `analyze_symbol()` live — if `direction == 'NEUTRAL'`, SKIP (signal degraded).

## Trade Flow (Separated Architecture)

1. **Scanner** → 24h stats for all USDT-M perps
2. **Filter** → extreme price moves (abs>20%) + volume leak detection (5m)
3. **Analyze** → scoring model + directional signal (LONG/SHORT/NEUTRAL)
4. **Enrich** → real-time review data (orderbook, trades, 5m, OI, funding)
5. **Write** → `pending_signals.json` with review + conflicts
6. **AI Reviewer (independent cron)** → Every 5m reads signals
7. **Decision** → OPEN/CAUTION/SKIP based on rules (see AI Review Architecture)
8. **Execute** → Reviewer calls `open_position()` for approved signals
9. **Cleanup** → Reviewer removes processed signals from file
10. **Notify** → Reviewer sends results via `notify_signal_results()`

## AI Review Architecture (Updated 2026-04-19)

**AI Review is separated from Pipeline into independent cron jobs.** This is the current architecture:

| Component | Cron Job | Schedule | Responsibility |
|-----------|----------|----------|----------------|
| **Scanner** | `b8d8528b113d` | 15,30,45,0 * * * * | Scan markets → write `pending_signals.json` |
| **AI Reviewer** | `ab663a974444` | every 5m | Read signals → AI decision → execute → cleanup |
| **Position Manager** | `214b772fa12e` | every 10m | Trailing SL/TP → close positions |

**Benefits of separation:**
1. **Reduced latency**: Max 5min delay (was 20min) — signals processed within 5min of scan
2. **Separation of concerns**: Scanner stays fast (~30s), Reviewer handles complex logic
3. **Independent tuning**: Can adjust scan frequency vs review frequency independently
4. **Cleaner failure modes**: If Reviewer fails, signals persist for next run

**AI Reviewer Decision Logic:**
```python
# Check order: already_holding → conflict/score → position_limit → execute
if sym in held_syms: SKIP + cleanup
if conflict_count >= 2 or score < 15: SKIP + cleanup
if available_slots <= 0: SKIP + cleanup  # positions_full 直接清理，不保留
if conflict_count == 1: CAUTION (50U) + cleanup → reason 带 `suggestion` 文案
if conflict_count == 0: OPEN (100U) + cleanup → reason 带 `suggestion` 文案
```

**Key files:**
- `~/trading-data/pending_signals.json` — Scanner writes, Reviewer reads+cleans
- `~/trading-data/review_decisions.json` — Reviewer writes audit log

## Symbol Precision

Each contract has different precision. Query: `binance-cli futures-usds exchange-information`

- `LOT_SIZE.stepSize` — quantity precision (many use `1` = integer ONLY)
- `PRICE_FILTER.tickSize` — price precision (use `tickSize`, NOT `pricePrecision`)
- `MIN_NOTIONAL.notional` — minimum order value (usually $5)

## Common Pitfalls (references/v6-history.md for full history)

1. **SETTLING symbols** — ~100 contracts in SETTLING status, CANNOT trade. Always `check_settling()`.
2. **`position-information-v3` returns leverage=0** — fallback: `notional / isolatedWallet`
3. **SL trigger price must be far from current price** — Binance rejects immediate-trigger orders
4. **Cooldown file** — never manually delete `cooldown.json` during debugging
5. **margin type cannot change when position exists** — set ISOLATED before opening
6. **Python buffering** — always use `PYTHONUNBUFFERED=1 python3 -u`
7. **`manage_positions()` must return `[]`** not `None`
8. **`triggerPrice` not `stopPrice`** — Binance algo order response uses `triggerPrice`
9. **Import path & PYTHONPATH**: pipeline uses `from pipelines.cli import CLI` and `from pipelines.common.logger import get_logger` — `PYTHONPATH=.:..` is **required** (imports `binance.*` from current dir + `pipelines.*` from parent). `PYTHONPATH=.` alone fails with `ModuleNotFoundError: No module named 'pipelines'`. Works from `~/trading-pipelines/pipelines` with `PYTHONPATH=.:..` or from `~/trading-pipelines` with `PYTHONPATH=.`.
10. **Optuna** requires `python3.12` (`/usr/bin/python3.12`), NOT Hermes venv `python3`
11. **Cron job command confirmed**: `cd ~/trading-pipelines/pipelines && PYTHONUNBUFFERED=1 PYTHONPATH=.:.. python3 -u -m binance.trading.futures_pipeline 2>&1 | head -200` (note `PYTHONPATH=.:..`, not just `PYTHONPATH=.`)
12. **SL algo order conflict — Binance reject**: Binance error `"An open stop or take profit order with GTE and closePosition in the direction is existing."` occurs when a previous algo order for the same symbol+positionSide wasn't cleaned up. This happens after re-entry (partial closes + new open) because `close_position()` only does market close, does NOT cancel algo orders. Fix: call `cancel_algo_orders(sym)` BEFORE placing new SL in `open_position()`.
13. **Duplicate open via pending_signals.json**: If a symbol already has an open position, old signals for that symbol can remain in `pending_signals.json` and get re-approved by the AI review cron, causing a duplicate open (Hedge Mode merges same-direction positions → doubled margin). Fix (2026-04-19): pipeline now filters `held_syms = {p["symbol"] for p in positions}` out of pending_signals.json in BOTH the full-position branch (L944) and the normal cleanup branch (L1042). Additionally, PM no longer triggers pipeline scans — see pitfall 14.
14. **PM no longer triggers pipeline scans**: As of 2026-04-19, the Position Manager (`position_manager.py`) does NOT call `trigger_pipeline()` — even when positions are empty or slots are available. Scanning is delegated to the pipeline cron only. This was changed by removing `trigger_pipeline()` calls from L525 and L540 of `position_manager.py`.
15. **Separated Scanner/Reviewer architecture (2026-04-19)**: Scanner (`b8d8528b113d`) only writes signals, Reviewer (`ab663a974444`) reads+decides+executes+cleans up. Reviewer checks: (1) already holding same symbol, (2) position limit MAX_POSITIONS=8, (3) conflict_count ≥2 or score<15 → SKIP. **All SKIP signals are cleaned** — no signals persist between runs.

16. **获取真实 Binance 持仓（CRITICAL for AI Reviewer）** — `position_manager.py` 里有 `get_open_trades()`，但它读的是**本地交易日志**（futures_trades.jsonl），不是 Binance 真实 API。如果本地日志和 Binance 实际持仓不同步，会导致 `available_slots` 计算错误。**正确做法**：用 `futures_pipeline.py` 里的 `get_positions()` 直接查 Binance `position-information-v3` API：

```bash
cd ~/trading-pipelines/pipelines && PYTHONPATH=.:.. python3 -c "
from pipelines.binance.trading.futures_pipeline import get_positions
positions = get_positions()  # 返回 list，每个元素有 symbol/positionAmt/leverage 等
# ⚠️ KEY NAMING: get_positions() wraps raw Binance API — keys might be camelCase (positionAmt)
# OR snake_case (position_amt) depending on the pipeline version. Check both:
amt = float(p.get('positionAmt', p.get('position_amt', 0)))
live = [p for p in positions if abs(float(p.get('positionAmt', p.get('position_amt', 0)))) > 0]
print(f'Current positions: {len(live)}')
print(f'Symbols: {[p[\"symbol\"] for p in live]}')
MAX_POSITIONS = 8
available_slots = MAX_POSITIONS - len(live)
print(f'Available slots: {available_slots}')
"
```

**禁止**使用 `position_manager.get_open_trades()` 查持仓计数。

17. **`notify_signal_results()` requires complete decision dict fields** — Every decision dict passed to `notify_signal_results()` MUST contain: `symbol`, `action`, `reason`, `direction`, `score`, `pct_24h`. The notification code reads `r['score']` and `r['pct_24h']` directly — KeyError if missing. For OPEN/CAUTION decisions, explicitly include `reason` (use `review.get('suggestion', 'default')` if no explicit reason) and `review` (the full review object from the signal). Without `reason`, the "信号依据" field in Discord embeds will be empty.

18. **Profit lock SL bugfix (2026-04-19)** — Two critical bugs fixed in `binance/strategy/exit_logic.py` `compute_trailing_sl()`:
  - **(a) Leverage missing from pnl calculation**: `pnl_pct` was computed as `(entry - close)/entry * 100` WITHOUT leverage. SHORT positions with +40% leveraged pnl showed only +14% raw price change, failing to trigger the 20% profit lock threshold. Fix: pass `leverage` param and multiply pnl by it.
  - **(b) SL = entry only保本, not profit lock**: Old logic `new_sl = min(new_sl, entry_price)` only broke even, didn't lock profits. Fix: `SL = peak + (entry - peak) * fraction` where `fraction = (eff_buffer + 20)/100`. SL sits between peak and entry, actually locking profit. SHORT uses peak (lowest price), LONG uses peak (highest price).
  - Callers updated in `binance/core/position.py` to pass both `leverage` and `peak_price`. Hyperliquid NOT modified (different codebase, fix deferred).

## Architecture Gap: Position Signal Degradation Monitoring (Added 2026-04-19)

**Problem discovered**: PM only manages trailing SL / partial TP / timeout exit. It does NOT do signal analysis. If a held position's signal degrades to NEUTRAL or reverses, nobody notices until SL hits or timeout (8h).

Example: PROMUSDT opened at 07:06, by 10:40 was -31.9% with `analyze_symbol()` returning NEUTRAL/score=18. PM kept trailing SL but never noticed the signal died.

**Solution**: AI Reviewer cron (`ab663a974444`) now monitors existing positions for signal degradation every 5 minutes. Implemented as a new step in the cron job prompt (Step 3.6: 持仓信号监控).

**Logic**: For each live position, re-run `analyze_symbol(sym, stats)`. If `direction == 'NEUTRAL'`:
- **Any PnL** → write to `position_suggestions.json` with `action_type: "full"`, `reason: "signal_degradation:neutral"`
- AI Review cron then reads this file and executes the close

This closes the architectural gap between "signal scanner only looks for new trades" and "position manager only looks at price/SL".

## Notifications

Pipeline has dual-channel Discord notifications:
- **Trade channel**: position opens/closes, SL failures, raw signals, overflow via `notify_new_signals()`, `notify_open_position()`, `notify_close_position()`, etc.
- **Signal channel**: AI review results for ALL signals via `notify_signal_results()`

**Important**: `notify_signal_results()` is called by the **cron job's Step 4** AFTER AI review, NOT by the pipeline itself. The pipeline only sends raw signal notifications via `notify_new_signals()`.

Action mapping: `OPEN`/`OPENED_*` → ✅ green embed, `SKIP`/`FILTERED` → 🚫 grey (汇总), `CAUTION` → ⚠️ amber embed. Each passed signal gets its own embed with "信号依据" field. Signal basis priority: (1) explicit `reason` field (split by `，` into multiple lines), (2) `review.suggestion` text, (3) auto-generated from review data (conflict_count, momentum_5m, funding_rate, orderbook spread/pressure, buy_ratio).

## Cron Job IDs (Updated 2026-04-19)

| Job | ID | Schedule |
|-----|----|----------|
| Binance Futures Scanner | `b8d8528b113d` | 15,30,45,0 * * * * |
| Binance AI Signal Reviewer | `ab663a974444` | every 5m |
| Binance Position Manager | `214b772fa12e` | every 10m |
| Binance Optuna 日优化 | `c4cdfaa10600` | 09:00 daily |
| Daily Report Morning | `b835142767e0` | 0 8 * * * |
| Daily Report Evening | `5b9900595df2` | 0 20 * * * |

## Full v2→v6 History

See `references/v6-history.md` for complete change logs, bug fixes, and historical trade results.

The user has provided the following instruction alongside the skill invocation: [SYSTEM: You are running as a scheduled cron job. DELIVERY: Your final response will be automatically delivered to the user — do NOT use send_message or try to deliver the output yourself. Just produce your report/output as your final response and the system handles the rest. SILENT: If there is genuinely nothing new to report, respond with exactly "[SILENT]" (nothing else) to suppress delivery. Never combine [SILENT] with content — either report your findings normally, or say [SILENT] and nothing more.]

## 任务：Binance 合约信号 AI Review

Schedule: 5,10,20,25,35,40,50,55 * * * * (每小时 8 次，与 Scanner 错开) | Skill: binance-futures-trader

---

### Step 1: 读取信号

```python
import json
from pathlib import Path
DATA_DIR = Path.home() / "trading-data"
signals_file = DATA_DIR / "pending_signals.json"

if not signals_file.exists():
    print("No signals file"); exit(0)

with open(signals_file) as f:
    signals = json.load(f)

if not signals:
    print("No pending signals"); exit(0)
```

---

### Step 2: 检查仓位状态（直接查 Binance API）

```bash
cd ~/trading-pipelines/pipelines && PYTHONPATH=.:.. python3 -c "
from pipelines.binance.trading.futures_pipeline import get_positions
positions = get_positions()
# 过滤非零持仓（Binance API 返回84行，只有 few 个 real）
live = [p for p in positions if abs(float(p.get('positionAmt', 0))) > 0]
held_syms = {p['symbol'] for p in live}
current_count = len(live)
MAX_POSITIONS = 8
available_slots = MAX_POSITIONS - current_count
print(f'Current positions: {current_count}')
print(f'Held symbols: {held_syms}')
print(f'Available slots: {available_slots}')
"
```

**禁止**使用 `position_manager.get_open_trades()` —— 它读本地日志，和 Binance 真实持仓可能不同步。

---

### Step 3: AI Review 决策

**你是一名经验丰富的量化交易分析师，负责在实盘入场前做最后一道风控。** 不要机械地按分数决定——你看到的是市场实时的脉搏。pipeline 扫描的是过去的快照，而你的职责是用当前市场状态确认（或推翻）信号。

**分析框架（在规则判断前先思考）：**

对每个信号，从三个维度做交叉验证：
1. **动量真实性** — `momentum_5m` 与信号方向是否同向？micro_momentum_pct 是否支持？如果 LONG 信号但 momentum_5m < 0 且 buy_ratio < 0.45，说明上涨动能已衰竭，考虑降级。
2. **订单簿结构** — spread_pct > 0.2% 说明流动性差、滑点大；ask_pressure=True 对 LONG 不利，bid_support=True 对 SHORT 不利。
3. **资金费率与OI** — 极高 funding（>0.001）可能是拥挤交易；OI 与 price 反向变动暗示主力在减仓。

**质量优先原则：宁可错过一个中等质量的信号，也不放一个有隐患的进去。** 如果某个信号让你感到"说不清楚哪里不对但就是不舒服"，相信直觉 — 降级或跳过。

**CRITICAL**: 每个 decision dict 必须包含 `reason` 和 `review` 字段，供 `notify_signal_results()` 生成通知中的"信号依据"。reason 应该反映你的分析判断（如"动量衰竭，已入场"、"OI背离，信号弱化"），而非仅机械规则。

```python
decisions = []
to_process = []  # 标记要清理的信号

for signal in signals:
    sym = signal['symbol']
    direction = signal['direction']
    score = signal.get('score', 0)
    review = signal.get('review', {})
    conflict_count = review.get('conflict_count', 0)
    
    # 规则 1: 已持仓（任何方向）→ SKIP
    if sym in held_syms:
        decisions.append({'symbol': sym, 'action': 'SKIP', 'reason': 'already_holding'})
        to_process.append(sym)
        continue
    
    # 规则 2: 冲突≥2 或分数<15 → SKIP
    if conflict_count >= 2 or score < 15:
        decisions.append({'symbol': sym, 'action': 'SKIP', 'reason': f'conflict={conflict_count}, score={score}'})
        to_process.append(sym)
        continue
    
    # 规则 3: 仓位已满 → SKIP + 清理（保留 overflow_flag 供通知用）
    if available_slots <= 0:
        decisions.append({
            'symbol': sym,
            'action': 'SKIP',
            'reason': 'positions_full',
            'overflow_flag': signal.get('overflow_flag', False),
            'direction': direction,
            'score': score,
            'pct_24h': signal.get('pct_24h', 0),
        })
        to_process.append(sym)
        continue
    
    # 规则 4: 冲突==1 → CAUTION (margin=50)
    if conflict_count == 1:
        decisions.append({
            'symbol': sym,
            'action': 'CAUTION',
            'margin': 50,
            'reason': review.get('suggestion', '存在 1 个冲突，保证金减半'),
            'review': review,   # 必须带！通知用
            'direction': direction,
            'score': score,
            'pct_24h': signal.get('pct_24h', 0),
        })
        to_process.append(sym)
        available_slots -= 1
        continue
    
    # 规则 5: 无冲突 → OPEN (margin=100)
    decisions.append({
        'symbol': sym,
        'action': 'OPEN',
        'margin': 100,
        'reason': review.get('suggestion', '无冲突，信号纯净'),
        'review': review,   # 必须带！通知用
        'direction': direction,
        'score': score,
        'pct_24h': signal.get('pct_24h', 0),
    })
    to_process.append(sym)
    available_slots -= 1
```

---

### Step 3.5: Live 信号验证（开仓前必做）

**⚠️ 信号从扫描到执行可能已经退化。对每个待 OPEN/CAUTION 的信号，必须重新检查实时状态：**

```python
from pipelines.binance.data.market_data import get_24h_stats
from pipelines.binance.trading.futures_pipeline import analyze_symbol

stats = get_24h_stats()
for d in decisions:
    if d['action'] in ['OPEN', 'CAUTION']:
        result = analyze_symbol(d['symbol'], stats)
        if result and result.get('direction') == 'NEUTRAL':
            d['action'] = 'SKIP_LIVE_DEGRADED_NEUTRAL'
            d['reason'] = f'信号退化：live analyze 显示 NEUTRAL（原方向 {d["direction"]}）'
```

**规则：如果 review 说 OK 但 `analyze_symbol().direction == 'NEUTRAL'`，SKIP。信号已退化，不要入场。**

---

### Step 4: 执行开仓

```python
from pipelines.binance.trading.futures_pipeline import open_position

for d in decisions:
    if d['action'] in ['OPEN', 'CAUTION']:
        signal = next((s for s in signals if s['symbol'] == d['symbol']), None)
        if signal:
            print(f"{d['action']}: {d['symbol']} margin={d['margin']}")
            open_position(d['symbol'], signal['direction'], signal, margin_usdt=d['margin'])
```

---

### Step 5: 记录 + 清理

```python
from datetime import datetime
from pipelines.binance.core.notifications import notify_signal_results

# 记录决策（包含完整信号数据供 notify_signal_results 使用）
for d in decisions:
    sig = next((s for s in signals if s['symbol'] == d['symbol']), None)
    if sig:
        d['direction'] = sig.get('direction', d.get('direction'))
        d['score'] = sig.get('score', d.get('score', 0))
        d['pct_24h'] = sig.get('pct_24h', 0)
        d['overflow_flag'] = sig.get('overflow_flag', d.get('overflow_flag', False))
        if 'review' not in d:
            d['review'] = sig.get('review', {})

with open(DATA_DIR / "review_decisions.json", 'w') as f:
    json.dump({'timestamp': datetime.now().isoformat(), 'decisions': decisions}, f, indent=2)

# 清理已处理信号
remaining = [s for s in signals if s['symbol'] not in to_process]
with open(signals_file, 'w') as f:
    json.dump(remaining, f, indent=2)

print(f"Processed: {len(to_process)}, Remaining: {len(remaining)}")

# 通知
notify_signal_results(decisions)
```

---

### 关键检查清单

- ✅ 同符号任何方向持仓 → 跳过
- ✅ 仓位上限 MAX_POSITIONS=8 → 满则跳过
- ✅ conflict_count ≥2 或 score<15 → 跳过
- ✅ conflict_count ==1 → CAUTION (50 USDT)
- ✅ 已处理信号 → 清理出文件
- ✅ Hedge Mode → 开仓带 positionSide
- ✅ 每个 decision 必须带 `reason`, `review`, `direction`, `score`, `pct_24h` 字段
- ✅ **开仓前必须 live 验证 `analyze_symbol()`，NEUTRAL 则 SKIP**
