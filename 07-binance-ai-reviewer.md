---
job_id: ab663a974444
schedule: "5,10,20,25,35,40,50,55 * * * *"
name: Binance AI Signal Reviewer
skills: ["binance-futures-trader"]
status: active
---

**⚠️ Skill 已通过 cron 自动加载，不要重复输出 skill 内容，直接执行任务。**

## 任务：Binance 合约信号 AI Review（每 5min）

### 执行流程

**Step 1**: 读 `~/trading-data/pending_signals.json`。信号列表为空则跳过信号审查，**但仍需执行 Step 3.6 持仓监控**。

**Step 2**: 查 Binance 实时持仓：
```
cd ~/trading-pipelines/pipelines && PYTHONPATH=.:.. python3 -c "
import json
from pipelines.binance.trading.futures_pipeline import get_positions
positions = get_positions()
live = [p for p in positions if abs(float(p.get('position_amt', 0))) > 0]
MAX_Positions = 8
print(json.dumps({'held': [p['symbol'] for p in live], 'slots': MAX_Positions - len(live), 'positions': [{'symbol': p['symbol'], 'position_amt': p['position_amt'], 'entry_price': p['entry_price'], 'mark_price': p['mark_price'], 'unrealized_profit': p['unrealized_profit'], 'leverage': p.get('leverage',0)} for p in live]}))
"
```
⚠️ 只取 `abs(float(p['position_amt'])) > 0` 的活仓。

**Step 3**: 对每个信号做 AI Review（无信号则跳过此步）
- 已持仓 → SKIP
- conflict≥2 或 score<15 → SKIP
- 无槽位 → SKIP
- 质量门控（加权）：动量反方 weight=2，流动性差(spread>0.3%)/资金费率拥挤(>0.002 for LONG)/成交反向 weight=1。总分≥2→SKIP，==1→CAUTION(50% margin)，==0→OPEN
- ⚠️ 每个 decision dict 必须含：`symbol, action, reason, review, direction, score, pct_24h`

**Step 3.5**: 对 OPEN/CAUTION 信号 re-run `analyze_symbol()`，NEUTRAL → SKIP

**Step 3.6**: 持仓监控 — 对每个 live 仓位跑 `analyze_symbol()`：
- 信号反转（LONG→SHORT 或 SHORT→LONG）：盈利中且 score<35 → 保留(trailing SL管)；亏损或 score≥35 → 写 `position_suggestions.json`（side=当前持仓方向！）
- NEUTRAL + 亏损>3% → 写 `position_suggestions.json`
- 格式：`{'symbol': sym, 'action_type': 'full', 'reason': 'signal_reversal/...: ...', 'side': current_side, 'reviewed': False}`

**Step 4**: 执行 OPEN/CAUTION — 每个开仓前重查 `get_positions()`，满则 STOP。CAUTION margin×0.5，`margin_usdt=` 参数。

**Step 5**: 写 `review_decisions.json`（完整 decisions），清理 `pending_signals.json`，`logger.batch_review(decisions)` 记录日志。无信号且无持仓监控动作时静默即可。

### 禁止
- 不要复写 skill 内容或确认规则
- 不要用 `positionAmt`（用 snake_case `position_amt`）
- 不要用 `position_manager.get_open_trades()` 查持仓
- 无信号 **且** 无持仓时 → `[SILENT]` 退出；否则正常输出监控结果
