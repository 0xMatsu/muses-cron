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