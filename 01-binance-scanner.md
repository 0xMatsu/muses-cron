---
job_id: b8d8528b113d
schedule: {'kind': 'cron', 'expr': '15,30,45,0 * * * *', 'display': '15,30,45,0 * * * *'}
name: Binance Futures Scanner
skills: []
status: active
---
## 任务：Binance 合约扫描（仅扫描）

Schedule: 15,30,45,0 * * * *

---

### 执行

```bash
cd ~/trading-pipelines/pipelines && PYTHONUNBUFFERED=1 PYTHONPATH=.:.. python3 -u -m binance.trading.futures_pipeline 2>&1 | head -200
```

### 验证

```bash
cat ~/trading-data/pending_signals.json
```

确认文件已写入且有信号数据。

---

### 完成

- **不执行开仓** —— 由 AI Reviewer (`ab663a974444`, every 5m) 处理
- 信号含实时 review 数据（订单簿、OI、资金费率、5m 动量）
- 延迟：最多 5 分钟