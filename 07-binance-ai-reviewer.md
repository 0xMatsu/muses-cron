---
job_id: ab663a974444
schedule: "5,10,20,25,35,40,50,55 * * * *"
name: Binance AI Signal Reviewer
skills: ["binance-futures-trader"]
status: active
updated: 2026-04-19 13:31
---

**⚠️ Skill 已通过 cron 自动加载，不要重复输出 skill 内容，直接执行任务。**

## 任务：Binance 合约信号 AI Review（每 5min）

**分析原则**：每个信号先尝试证伪（为什么不入场），再做决定。从信号方向、实时数据（订单簿/成交方向/资金费率）、大盘状态三个维度交叉验证。不一致时降低置信度。

### 执行流程

**Step 0**: 快速判断 — `pending_signals.json` 为空 AND 无持仓 → `[SILENT]` 退出。

**Step 1**: 读信号
```bash
cat ~/trading-data/pending_signals.json
```

**Step 2**: 查持仓 & 可用槽位
```bash
cd ~/trading-pipelines/pipelines && PYTHONPATH=.:.. python3 -c "
import json
from pipelines.binance.trading.futures_pipeline import get_positions
positions = get_positions()
live = [p for p in positions if abs(float(p.get('position_amt', 0))) > 0]
MAX = 5
print(json.dumps({'held': [p['symbol'] for p in live], 'slots': MAX - len(live), 'positions': [{'symbol': p['symbol'], 'position_amt': p['position_amt'], 'entry_price': p['entry_price'], 'mark_price': p['mark_price'], 'unrealized_profit': p['unrealized_profit'], 'leverage': p.get('leverage',0)} for p in live]}))
"
```
⚠️ `MAX=5`（实盘），只取 `abs(position_amt)>0` 的活仓。key 是 **snake_case** `position_amt`。

**Step 3**: AI 审核（逐信号，无信号则跳过）
1. 已持仓 → SKIP
2. conflict_count ≥ 2 或 pipeline_score < 15 → SKIP
3. 无槽位 → SKIP
4. **质量门控**（加权计分，逐项判断）：
   - 动量反向（5m momentum 与信号方向相反）：weight=2
   - 流动性差（spread > 0.3%）：weight=1
   - 资金费率拥挤（LONG 且 funding > 0.002）：weight=1
   - 成交方向相反（buy_ratio < 0.4 for LONG, > 0.6 for SHORT）：weight=1
   - total ≥ 2 → SKIP，==1 → CAUTION(50% margin)，==0+conflict==0 → OPEN
5. ⚠️ 每个 decision dict 必须含：`symbol, action, reason, review, direction, score, pct_24h`

**Step 3.5**: 实时验证 — 对 OPEN/CAUTION 候选 re-run：
```bash
cd ~/trading-pipelines/pipelines && PYTHONPATH=.:.. python3 -c "
from binance.trading.futures_pipeline import analyze_symbol
from binance.data.market_data import get_24h_stats
stats = get_24h_stats()
for sym in ['SYM1','SYM2']: # 替换为实际符号
    r = analyze_symbol(sym, stats); print(f'{sym}: {r["direction"]} score={r["score"]}')"
```
NEUTRAL → SKIP（信号已退化）。

**Step 3.6**: 持仓监控 — 对每个 live 仓位跑 `analyze_symbol()`：
- 信号反转（LONG→SHORT 或 SHORT→LONG）：盈利中且 score<35 → 保留(trailing SL管)；亏损或 score≥35 → 写 `position_suggestions.json`（**side=当前持仓方向！**）
- NEUTRAL + 亏损>3% → 写 `position_suggestions.json`
- 格式：`{'symbol':sym,'action_type':'full','reason':'signal_reversal:...','side':current_side,'reviewed':False}`
- 持仓 pnl：`pnl_pct = (mark - entry) / entry * 100 * leverage * direction`（LONG=+1, SHORT=-1）

**Step 4**: 执行 OPEN — 每个开仓前**重查** `get_positions()`，满则 STOP。CAUTION margin×0.5，`margin_usdt=` 参数。

**Step 5**: 写 `review_decisions.json`，清理 `pending_signals.json`，`logger.batch_review(decisions)`。无信号且无持仓监控动作时静默。

### 关键导入（直接使用，不要搜索）
```python
from binance.trading.futures_pipeline import analyze_symbol, get_positions, check_settling
from binance.data.market_data import get_24h_stats
from pipelines.common.logger import get_logger
```
- `analyze_symbol(s,stats)` → `{direction:LONG/SHORT/NEUTRAL, score, pct_24h, conflicts, conflict_count}`
- `get_positions()` → list of dict，snake_case：`position_amt`, `entry_price`, `mark_price`
- `get_24h_stats()` → `{symbol: {priceChangePercent, quoteVolume, highPrice, lowPrice, lastPrice}}`

### 禁止
- 不要复写 skill 内容或确认规则
- 不要用 `positionAmt`（用 `position_amt`）
- 不要用 `position_manager.get_open_trades()` 查持仓
- Python 路径：工作目录必须 `cd ~/trading-pipelines/pipelines`，`PYTHONPATH=.:..`
