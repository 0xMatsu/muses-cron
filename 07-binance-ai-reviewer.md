---
{
  "job_id": "ab663a974444",
  "name": "Binance AI Signal Reviewer",
  "schedule": "5,10,20,25,35,40,50,55 * * * *",
  "deliver": "local",
  "skills": ["binance-futures-trader"],
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
```

## File Paths

| What | Path |
|------|------|
| Pipeline code | `~/trading-pipelines/pipelines/binance/trading/futures_pipeline.py` |
| Position manager | `~/trading-pipelines/pipelines/binance/trading/position_manager.py` |
| Notifications | `~/trading-pipelines/pipelines/binance/core/notifications.py` |
| Data files | `~/trading-data/` (cooldown.json, futures_trades.jsonl, pending_signals.json, etc.) |
| Config | `config.py` within the binance package |
| Strategies | `futures_strategy.json` in DATA_DIR |

## Config Values (verified 2026-04-18)

| Parameter | Value |
|-----------|-------|
| MAX_POSITIONS | 8 |
| LEVERAGE | 3x |
| MARGIN_RANGE | (80, 100) — DYNAMIC via calc_margin |
| SL_FROM_LIQ | 0.4 |
| FREQ_CAP_HOURS | 8 |
| FREQ_CAP_MAX | 10 |
| COOLDOWN_MINUTES | 15 |
| COOLDOWN_REPEAT_PENALTY | 90 |

**Rule:** ALL thresholds live in `config.py`. NEVER hardcode strategy values.

## Hedge Mode (CRITICAL)

Account is in Hedge Mode (`dualSidePosition: true`).

- **open_position()**: `open_position(sym, side, signal, margin_usdt)`
- **close_position()**: `close_position(sym, reason="", record_pnl=True)`
- **cancel_algo_orders(sym)**: MUST call before placing new SL/algo order

## Signal Review Data (built into pipeline)

Each signal in `pending_signals.json` has a `review` key with: `current_price`, `momentum_5m`, `funding_rate`, `orderbook` (spread_pct, ask_pressure, bid_support), `recent_trades` (buy_ratio, micro_momentum_pct), `conflicts`, `conflict_count`, `suggestion`.

**SKIP** if `conflict_count >= 2` or `score < 15` or already holding this symbol.
**CAUTION** if `conflict_count == 1` — reduce margin 50%.
**OPEN** if `conflict_count == 0` AND `score >= 15` AND quality gate passes.

## Common Pitfalls

16. **get_positions() returns snake_case `position_amt`** — NOT camelCase `positionAmt`. Never use fallback `p.get('positionAmt', p.get('position_amt', 0))` — silent zero if camelCase absent = all positions misjudged as empty.

17. **notify_signal_results() requires: symbol, action, reason, direction, score, pct_24h** — KeyError if missing. Include `review` field for OPEN/CAUTION.

12. **cancel_algo_orders(sym) before open** — old algo orders cause Binance reject on re-entry.

## Position Signal Degradation Monitoring

AI Reviewer monitors existing positions every 5 min. If live `analyze_symbol()` returns NEUTRAL for a held position → write `position_suggestions.json` with exit suggestion.

## Notifications

`notify_signal_results()` called by cron Step 4 AFTER AI review.

Action mapping: OPEN → ✅, CAUTION → ⚠️, SKIP → 🚫

The user has provided the following instruction alongside the skill invocation: [SYSTEM: You are running as a scheduled cron job. DELIVERY: Your final response will be automatically delivered to the user. SILENT: If nothing to report, respond "[SILENT]".]

## 任务：Binance 合约信号 AI Review

Schedule: 5,10,20,25,35,40,50,55 * * * * | Skill: binance-futures-trader

---

### Step 1: 读取信号

```python
import json
from pathlib import Path
DATA_DIR = Path.home() / "trading-data"
signals_file = DATA_DIR / "pending_signals.json"
if not signals_file.exists(): print("No signals file"); exit(0)
with open(signals_file) as f: signals = json.load(f)
if not signals: print("No pending signals"); exit(0)
```

---

### Step 2: 检查仓位状态（直接查 Binance API）

⚠️ `get_positions()` 返回 **snake_case**（`position_amt`），不是 camelCase。
⚠️ 工作目录：`cd ~/trading-pipelines/pipelines && PYTHONPATH=.:..`

```bash
cd ~/trading-pipelines/pipelines && PYTHONPATH=.:.. python3 -c "
import json
from pipelines.binance.trading.futures_pipeline import get_positions
positions = get_positions()
live = [p for p in positions if abs(float(p.get('position_amt', 0))) > 0]
held_syms = {p['symbol'] for p in live}
MAX_POSITIONS = 8
available_slots = MAX_POSITIONS - len(live)
result = {
    'held_syms': list(held_syms),
    'available_slots': available_slots,
    'live_positions': [{'symbol': p['symbol'], 'position_amt': p.get('position_amt',0), 'entry_price': p.get('entry_price',0), 'mark_price': p.get('mark_price',0), 'unrealized_profit': p.get('unrealized_profit',0), 'leverage': p.get('leverage',0)} for p in live]
}
print(json.dumps(result))
"
```

解析 Step 2 的 JSON 输出获取 `held_syms`、`available_slots`、`live_positions`。

**禁止**使用 `position_manager.get_open_trades()` —— 读本地日志，可能不同步。

---

### Step 3: AI Review 决策

你是一名经验丰富的量化交易分析师，负责在实盘入场前做最后一道风控。质量优先原则：宁可错过一个中等质量的信号，也不放一个有隐患的进去。

每个 decision dict 必须包含 `reason`（你的分析判断，非仅机械规则）和 `review` 字段。

对每个信号做交叉验证：
1. **动量真实性** — momentum_5m 与方向是否同向？
2. **订单簿结构** — spread_pct > 0.2% 流动性差；ask_pressure 对 LONG 不利
3. **资金费率与 OI** — funding > 0.001 可能是拥挤交易

```python
decisions = []
to_process = []

for signal in signals:
    sym = signal['symbol']
    direction = signal['direction']
    score = signal.get('score', 0)
    review = signal.get('review', {})
    conflict_count = review.get('conflict_count', 0)
    spread_pct = review.get('orderbook', {}).get('spread_pct', 0)
    momentum_5m = review.get('momentum_5m', 0)
    buy_ratio = review.get('recent_trades', {}).get('buy_ratio', 0.5)
    funding_rate = review.get('funding_rate', 0)

    if sym in held_syms:
        decisions.append({'symbol': sym, 'action': 'SKIP', 'reason': 'already_holding'})
        to_process.append(sym); continue

    if conflict_count >= 2 or score < 15:
        decisions.append({'symbol': sym, 'action': 'SKIP', 'reason': f'conflict={conflict_count}, score={score}'})
        to_process.append(sym); continue

    if available_slots <= 0:
        decisions.append({'symbol': sym, 'action': 'SKIP', 'reason': 'positions_full',
            'overflow_flag': signal.get('overflow_flag', False),
            'direction': direction, 'score': score, 'pct_24h': signal.get('pct_24h', 0)})
        to_process.append(sym); continue

    # === 质量门控（conflict==0 也要过）===
    momentum_reversed = (direction == 'LONG' and momentum_5m < -1) or (direction == 'SHORT' and momentum_5m > 1)
    illiquid = spread_pct > 0.3
    crowded = funding_rate > 0.002 and direction == 'LONG'
    flow_against = (direction == 'LONG' and buy_ratio < 0.35) or (direction == 'SHORT' and buy_ratio > 0.65)
    issues = sum([momentum_reversed, illiquid, crowded, flow_against])

    if conflict_count == 1 or issues >= 1:
        downgrade = 'CAUTION' if issues == 1 else 'SKIP'
        reasons = []
        if momentum_reversed: reasons.append(f'动量反向 (momentum_5m={momentum_5m})')
        if illiquid: reasons.append(f'流动性差 (spread={spread_pct}%)')
        if crowded: reasons.append(f'多头拥挤 (funding={funding_rate})')
        if flow_against: reasons.append(f'成交方向反 (buy_ratio={buy_ratio})')
        if conflict_count == 1: reasons.append(review.get('suggestion', '存在1个冲突'))
        decisions.append({'symbol': sym, 'action': downgrade,
            'margin': 50 if downgrade == 'CAUTION' else 0,
            'reason': '，'.join(reasons) if reasons else review.get('suggestion', '综合质量评估未达标'),
            'review': review, 'direction': direction, 'score': score,
            'pct_24h': signal.get('pct_24h', 0)})
        to_process.append(sym); continue

    # 通过规则 → OPEN
    decisions.append({'symbol': sym, 'action': 'OPEN', 'reason': review.get('suggestion', '信号纯净，质量门控通过'),
        'review': review, 'direction': direction, 'score': score, 'pct_24h': signal.get('pct_24h', 0)})
    to_process.append(sym)
```

---

### Step 3.5: Live 信号验证（开仓前必做）

信号从扫描到执行可能退化。对每个待 OPEN/CAUTION 的信号，重新检查实时状态：

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

**规则：live analyze=NEUTRAL → SKIP，不管 review 说了什么。**

---

### Step 3.6: 持仓信号监控（每 5 分钟检查现有持仓）

对每持运行 live `analyze_symbol()`，检测信号退化并生成退出建议：

```python
from pipelines.binance.data.market_data import get_24h_stats
from pipelines.binance.trading.futures_pipeline import analyze_symbol

stats = get_24h_stats()
position_suggestions = []

for pos in live_positions:
    sym = pos['symbol']
    entry_price = float(pos['entry_price'])
    mark_price = float(pos['mark_price'])
    leverage = float(pos['leverage']) if pos['leverage'] > 0 else 10  # fallback
    position_amt = float(pos['position_amt'])
    is_long = position_amt > 0

    if leverage <= 0: leverage = 10
    # 杠杆化 PnL%: LONG=(mark-entry)/entry*leverage*100, SHORT=(entry-mark)/entry*leverage*100
    if is_long:
        pnl_pct = (mark_price - entry_price) / entry_price * leverage * 100
    else:
        pnl_pct = (entry_price - mark_price) / entry_price * leverage * 100

    result = analyze_symbol(sym, stats)
    if not result: continue
    direction = result.get('direction', '')
    live_score = result.get('score', 0)

    # 方向完全反向: 无条件建议退出
    if (is_long and direction == 'SHORT') or (not is_long and direction == 'LONG'):
        position_suggestions.append({
            'symbol': sym, 'action_type': 'full',
            'reason': f'signal_reversal: 持仓方向 {"LONG" if is_long else "SHORT"}, 当前信号 {direction}(score={live_score})',
            'side': position, 'reviewed': False
        })
        continue

    # 信号退化到 NEUTRAL → 仅在亏损时建议退出（避免微利时被震荡反复进出）
    if direction == 'NEUTRAL' and pnl_pct < -3:
        position_suggestions.append({
            'symbol': sym, 'action_type': 'full',
            'reason': f'signal_degradation: NEUTRAL(score={live_score}), 持仓亏损 {pnl_pct:.1f}%',
            'side': position, 'reviewed': False
        })

if position_suggestions:
    with open(DATA_DIR / "position_suggestions.json", 'w') as f:
        json.dump(position_suggestions, f, indent=2, ensure_ascii=False)
    print(f'Position suggestions: {len(position_suggestions)}')
```

**退出条件调整**：
- 信号方向反转 → **无条件**建议全仓退出（不管盈亏）
- 信号退化 NEUTRAL → **仅当 pnl < -3%** 时退出（避免震荡中反复进出）

---

### Step 4: 执行开仓

⚠️ **每次开仓前重新检查持仓数** — 不能在循环前算一次就用到底。

```python
from pipelines.binance.trading.futures_pipeline import open_position, get_positions, calc_margin, get_balance
from pipelines.common.logger import get_logger

logger = get_logger('review')
opened_count = 0
MAX_POSITIONS = 8

for d in decisions:
    if d['action'] not in ['OPEN', 'CAUTION']: continue
    sym = d['symbol']

    # 重新检查持仓上限（防止 batch 中超仓）
    positions = get_positions()
    if len(positions) >= MAX_POSITIONS:
        d['action'] = 'STOP_POSITIONS_FULL'
        d['reason'] = f'开仓时发现仓位已满 ({MAX_POSITIONS}/{MAX_POSITIONS})'
        break

    signal = next((s for s in signals if s['symbol'] == sym), None)
    if not signal: continue

    try:
        # 动态计算保证金
        balance = get_balance()
        base_margin = calc_margin(balance, signal, positions) if signal else 80
        margin = base_margin * 0.5 if d['action'] == 'CAUTION' else base_margin

        open_position(sym, signal['direction'], signal, margin_usdt=margin)
        d['action'] = f'OPENED_{signal["direction"]}'
        d['margin_used'] = margin
        opened_count += 1
        logger.info(f"Opened {sym} {signal['direction']} margin=${margin:.0f}")
    except Exception as e:
        d['action'] = f'FAILED_{d["action"]}'
        d['reason'] = f'开仓失败: {str(e)}'
        logger.error(f"Failed to open {sym}: {e}")
        raise  # 重新抛出，让 cron 看到错误
```

---

### Step 5: 记录 + 清理 + 通知

```python
from datetime import datetime
from pipelines.binance.core.notifications import notify_signal_results

# 确保 decisions 有完整字段
for d in decisions:
    sig = next((s for s in signals if s['symbol'] == d['symbol']), None)
    if sig:
        d.setdefault('direction', sig.get('direction', d.get('direction')))
        d.setdefault('score', sig.get('score', d.get('score', 0)))
        d.setdefault('pct_24h', sig.get('pct_24h', 0))
        d.setdefault('overflow_flag', sig.get('overflow_flag', d.get('overflow_flag', False)))
        if 'review' not in d: d['review'] = sig.get('review', {})

with open(DATA_DIR / "review_decisions.json", 'w') as f:
    json.dump({'timestamp': datetime.now().isoformat(), 'decisions': decisions}, f, indent=2)

remaining = [s for s in signals if s['symbol'] not in to_process]
with open(signals_file, 'w') as f:
    json.dump(remaining, f, indent=2)

print(f"Processed {len(to_process)} signals, {len(remaining)} remaining")
notify_signal_results(decisions)
```

---

### 关键检查清单

- ✅ 同符号任何方向持仓 → SKIP
- ✅ 持仓数实时确认 (`get_positions()`)，不是只在开头查一次
- ✅ **仓位上限 MAX_POSITIONS=8** → 满则 SKIP
- ✅ **open_position 前 cancel_algo_orders** → 清理旧 SL/TP
- ✅ conflict_count ≥2 或 score<15 → SKIP
- ✅ conflict_count ==1 → CAUTION (margin=calc_margin * 0.5)
- ✅ 质量门控：动量反向、流动性差、funding 拥挤、成交方向反 → 降级
- ✅ **开仓前 live 验证 `analyze_symbol()`**，NEUTRAL → SKIP
- ✅ **Step 3.6: 持仓信号监控** → 方向反转无条件退出，NEUTRAL 且亏损 <-3% 退出
- ✅ 每个 decision 必须带 reason, review, direction, score, pct_24h
- ✅ **get_positions() 用 snake_case `position_amt`**，禁止用 camelCase
