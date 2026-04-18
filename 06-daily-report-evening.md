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