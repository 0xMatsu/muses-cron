---
job_id: 6ab3f8c34c2e
schedule: {'kind': 'interval', 'minutes': 20, 'display': 'every 20m'}
name: Hyperliquid AI Trader
skills: ["hyperliquid-ai-trader"]
status: active
---
## 任务：Hyperliquid AI 自主交易（每20分钟）

**⚠️ Skill 已通过 cron 自动加载，不要重复输出 skill 内容，直接执行任务。**

你是 Hyperliquid 的 AI 交易员。每20分钟你被唤醒一次，**自主完成以下所有决策和操**：

### Phase 1 — 余额验证
**注意**：Hyperliquid 是统一账户模式。spot_user_state() 返回的 USDC `total` 字段 = 全部余额（可用USDC + 仓位保证金 + 未实现盈亏）。这就是你的总资金，用它来计算仓位大小和风险，不要把它当成仅仅是可用余额。
```bash
cd ~/trading-pipelines && python3 -c "
import os, json, eth_account
from hyperliquid.info import Info
from hyperliquid.exchange import Exchange
wallets = json.load(open(os.path.expanduser('~/wallets.json')))
pk = wallets['hypeliquid']['private_key'].replace('0x', '')
acct = eth_account.Account.from_key('0x' + pk)
info = Info(base_url=None, skip_ws=True)
spot = info.spot_user_state(acct.address)
usdc_total = 0
for b in spot['balances']:
    if b['coin'] == 'USDC':
        usdc_total = float(b['total'])
        print(f'USDC_BALANCE_TOTAL={usdc_total}  # 全部余额 = 可用USDC + 仓位价值')
state = info.user_state(acct.address)
# 各持仓的未实现盈亏明细
for a in state.get('assetPositions', []):
    p = a.get('position', {})
    szi = float(p.get('szi', 0))
    if abs(szi) > 0:
        print(f'POS={p["coin"]}|{"long" if szi>0 else "short"}|{p.get("entryPx","0")}|{p.get("liquidationPx","0")}|{szi}|{p.get("unrealizedPnl","0")}')
# Print open orders
orders = info.open_orders(acct.address)
for o in orders:
    print(f'ORDER={o.get("coin","?")}|{o.get("side","?")}|px={o.get("px","")}trigger={o.get("triggerPx","")}sz={o.get("sz","")}')
mids = info.all_mids()
# Print top 50 coins by volume
ctxs = info.meta_and_asset_ctxs()
vol_list = []
for i, u in enumerate(ctxs[0]['universe']):
    d = ctxs[1][i]
    vol = float(d.get('dayNtlVlm',0))
    if vol > 0:
        vol_list.append((u['name'], vol, d.get('funding','0'), d.get('openInterest','0'), mids.get(u['name'],'0')))
vol_list.sort(key=lambda x: x[1], reverse=True)
for name, vol, fund, oi, mid in vol_list[:50]:
    print(f'COIN={name}|vol={vol}|fund={fund}|oi={oi}|mid={mid}')
```

### Phase 2 — 分析持仓并决策
对**每个持**：
1. 获取该币最近40根 15m K线（用 `info.candles_snapshot(coin, "15m", start_ms, end_ms)`）
2. 分析趋势、动量、支撑/阻力、RSI、成交量变化
3. 决定：
   - **持有** — 信号仍然有效
   - **止盈** — 到达关键阻力、过度延伸、或盈利显著
   - **止损** — 信号失效、趋势反转、亏损扩大
   - **移动止损** — 已盈利时设置保本或更高 SL
   - **部分平仓** — 减少风险但仍保留部分仓位

执行平仓/调整命令：
```python
import os, json, eth_account, time
from hyperliquid.info import Info
from hyperliquid.exchange import Exchange
wallets = json.load(open(os.path.expanduser('~/wallets.json')))
pk = wallets['hypeliquid']['private_key'].replace('0x', '')
acct = eth_account.Account.from_key('0x' + pk)
info = Info(base_url=None, skip_ws=True)
exchange = Exchange(wallet=acct, base_url=None)

# 获取 price decimals
meta = info.meta_and_asset_ctxs()
sz_decimals = {a['name']: a['szDecimals'] for a in meta[0]['universe']}

# 完全平仓
exchange.market_close('COIN')

# 部分平仓
close_sz = round(target_sz * fraction, sz_decimals['COIN'])
mid = float(info.all_mids()['COIN'])
side_is_buy = current_side == 'short'  # buy to close short
limit_px = round(mid * (1.03 if side_is_buy else 0.97), 6)
exchange.order(name='COIN', is_buy=side_is_buy, sz=close_sz,
               limit_px=limit_px, order_type={'limit': {'tif': 'Ioc'}},
               reduce_only=True)

# 设置/更新止损
# 先取消该币所有旧订单，然后设新 SL
for o in info.open_orders(acct.address):
    if o['coin'] == 'COIN':
        exchange.cancel('COIN', o['oid'])
sl_px = round(sl_target_price, sz_decimals['COIN'])  # 注意使用正确的价格精度
sl_limit = round(sl_px * 0.99 if current_side == 'long' else sl_px * 1.01, sz_decimals['COIN'])
is_close = not (current_side == 'long')  # sell to close long, buy to close short
exchange.order(name='COIN', is_buy=is_close, sz=position_sz,
               limit_px=sl_limit,
               order_type={'trigger': {'triggerPx': sl_px, 'isMarket': True, 'tpsl': 'sl'}},
               reduce_only=True)
```
每次下单后检查 `result.get('status')` 以及 `result['response']['data']['statuses'][]` 中的 error/filled。

### Phase 3 — 扫描市场并选币
1. 从 top 50 by volume 中选 3-5 个有交易机会的币
2. 获取它们的 15m K线（约40根）
3. **自主分析**：看盘形、趋势、指标——你觉得该开什么单子？
4. 自行决定保证金（基于总余额）、杠杆、SL/TP

风控约束（**唯一硬编码的规则**）：
- MAX_POSITIONS = 5（最多5个同时持仓）
- 单笔风险不超过余额的 15%
- **只做多（LONG）**，暂时不做空
- 新开的仓位必须在开仓后立即设置 SL 触发单
- 已持仓的币60分钟内不再开

### Phase 4 — 执行开单
按 Phase 3 的决定执行：
```python
# 设置杠杆（逐仓）
exchange.update_leverage(leverage, 'COIN', is_cross=False)
time.sleep(0.3)

# 市价开仓
notional_usd = margin * leverage
sz = round(notional_usd / entry_price, sz_decimals['COIN'])
result = exchange.market_open('COIN', is_buy=True, sz=sz, None, 0.01)
# 检查 result

# 立即设置 SL
# （同上 cancel 旧单 + 设新SL 的流程）
```

### Phase 5 — 记录决策
将本次交易的决策和结果写入文件：
```bash
echo "$(date '+%Y-%m-%d %H:%M') | 你的决策摘要" >> ~/trading-data/hyperliquid/ai_trade_log.txt
```

### 出错时的自我修正
如果你发现：
- 某个操作一直失败（如 SL 下单失败、K线获取超时）
- 你之前做的决策导致了亏损
- 某些参数不合适

**请通过以下方式修正**：
1. 如果是代码层面的bug（如精度问题、参数错误）→ 用 `execute_code` 修好并重试
2. 如果是策略层面的教训 → **更新你的记忆**（使用 memory 或 fact_store 工具记录教训）
3. 在本轮回复中说明修了什么、为什么修、下次如何避免

### 输出要求
最后输出一个清晰的摘要：
- 当前余额
- 每个持仓的状态和你的操作（持有/平仓/调SL）
- 新开仓（如果有）及参数
- 遇到的错误和修正
- 你对市场的整体判断