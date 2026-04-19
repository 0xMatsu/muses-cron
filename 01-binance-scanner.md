---
{
  "job_id": "b8d8528b113d",
  "name": "Binance Futures Scanner",
  "schedule": "15,30,45,0 * * * *",
  "deliver": "local",
  "skills": [],
  "enabled": true,
  "state": "scheduled"
}
---

[SYSTEM: You are running as a scheduled cron job. DELIVERY: Your final response will be automatically delivered to the user — do NOT use send_message or try to deliver the output yourself. Just produce your report/output as your final response and the system handles the rest. SILENT: If there is genuinely nothing new to report, respond with exactly "[SILENT]" (nothing else) to suppress delivery. Never combine [SILENT] with content — either report your findings normally, or say [SILENT] and nothing more.]

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
