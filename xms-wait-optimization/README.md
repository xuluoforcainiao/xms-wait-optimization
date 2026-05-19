# xms-wait-optimization

> QoderWork skill — XMS system wait-time optimization strategy for browser automation.

## What

XMS (`cs-packet.i4px.com`) is a slow internal web application. When an Agent automates clicks too quickly, the page doesn't finish loading before the next action runs, causing failures.

This skill defines a **10-second wait + verification** pattern after every critical action to match human pacing.

## When to Use

- XMS export fails due to timing issues
- Adding wait/verification steps to browser automation
- Optimizing interactions with slow-loading internal systems

## Key Strategies

### 1. Chrome Startup Flag

Launch Chrome with `--safebrowsing-disable-download-protection` to prevent security popups on `.xlsx` downloads.

### 2. Wait + Verify Pattern

After every critical action:
1. Wait 10 seconds
2. Verify page state via DOM check (preferred) or screenshot (fallback)
3. Only proceed if verification passes

### 3. Six Critical Checkpoints

| # | Action | Wait | Verify |
|---|--------|------|--------|
| 1 | Navigate to `opAbnormal` | 10s | URL + screenshot |
| 2 | Clear shelf organization | — | Placeholder text |
| 3 | Click **查询** | 10s | Table row count |
| 4 | Click **导出原因(新)** | 10s | Handle "导出失败" dialog |
| 5 | Open **下载中心** | 10s | "执行中" status |
| 6 | Click **下载** | 3s | Chrome security popup |

### 4. Empty Table Retry

If query returns empty:
- Wait 10s → click 查询 again
- Repeat up to 3 times
- If still empty → full recovery (close tab → re-login → restart)

### 5. Export Timeout

`max_wait_minutes = 100` (based on historical range 12–90 minutes).

## File Structure

```
xms-wait-optimization/
├── README.md      # This file
├── SKILL.md       # Core instructions for the Agent
└── reference.md   # Historical data, comparison tables, flowcharts
```

## History

| Date | Event |
|------|-------|
| 2026-05-19 | Created from root-cause analysis of 08:00 failure vs 09:11 success |

## License

MIT
