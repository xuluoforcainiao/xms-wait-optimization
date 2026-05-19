# XMS 等待时间优化 — 参考文档

## Historical Data: Export Duration

| Date | Export Duration | File Size | Notes |
|------|----------------|-----------|-------|
| 2026-05-11 | 38 min | - | |
| 2026-05-12 | 26 min | - | |
| 2026-05-13 | 30 min | - | |
| 2026-05-14 | 12 min | - | Fastest on record |
| 2026-05-15 | 15 min | - | |
| 2026-05-17 | 40 min | - | |
| 2026-05-19 | 90 min | - | Slowest on record |

**Average**: 35-40 minutes
**Range**: 12-90 minutes
**Recommended `max_wait_minutes`**: 100 (covers the historical maximum with a small buffer)

## 08:00 vs 09:11 Comparison (2026-05-19)

### 08:00 Failure

| Metric | Value |
|--------|-------|
| Trigger | Scheduled |
| Duration | 1600s (26 minutes) |
| Tool parts | 285 |
| Log size | 3,940 chars |
| Query result | Empty table (all attempts) |
| Export result | "导出失败" dialog |
| Recovery strategy | Retried 3 rounds, then gave up |
| Final state | Failed |

### 09:11 Success

| Metric | Value |
|--------|-------|
| Trigger | Manual (tool) |
| Duration | 3281s (54 minutes) |
| Tool parts | 523 |
| Log size | 21,427 chars |
| Query result | First empty → after full recovery → 14,772 rows |
| Export result | Normal export triggered |
| Recovery strategy | Full recovery (close tab → re-login → re-query) succeeded |
| Final state | Completed (4.59MB, pushed to 2 groups) |

### Key Difference

The 09:11 run used **full recovery** (re-login + re-query) when the download center showed no data, which loaded the data successfully (3 rows → 14,772 rows). The 08:00 run only did simplified retries without re-login, which never recovered.

**Root cause**: Upstream data import delay. The same query returned empty at 08:00 but 14,772 rows at 09:11, meaning data was imported between those times (likely a morning batch delay).

## User Manual Operation Reference

The user provided screenshots and timing for their manual workflow:

| Step | Action | Wait | Verify |
|------|--------|------|--------|
| 1 | Click navigation to opAbnormal | 10s | Screenshot: filter panel + data table area (Figure 1) |
| 2 | Clear shelf organization | - | Field shows placeholder "双击选择上架组织" |
| 3 | Click 查询 | 10s | Screenshot: data rows with red-highlighted abnormal records (Figure 2) |
| 4 | Click 导出原因(新) | 10s | - |
| 5 | Click 下载中心 | 10s | Screenshot: "执行中" status in download drawer (Figure 3) |
| 6 | Click 下载 | 3s | Check for Chrome security popup (Figure 4) |
| 7 | (If popup) Click 保留 | - | Download proceeds |

## Chrome Safe Browsing Flag

**Command**: `--safebrowsing-disable-download-protection`

**Purpose**: Prevents Chrome from showing the "xxx.xlsx 可能存在风险" (This file may be harmful) security popup when downloading files from internal systems like XMS.

**Fallback**: If the flag stops working (e.g., after a Chrome update that removes support), the Agent must:
1. Wait 3 seconds after clicking download
2. Take a screenshot or use `read_page` to check for the popup
3. If the popup appears, click the "保留" (Keep) button
4. Proceed to file processing

## Credit Impact Analysis

| Strategy | Before | After | Savings |
|----------|--------|-------|---------|
| Wait time per critical action | 0-3s | 10s | - |
| Empty table retry | 1 round (30s wait) | Up to 3 rounds (10s each) | Slight increase |
| Download center verification | None | 10s wait + DOM check | +1 verification |
| Chrome security popup handling | None | 3s wait + check | +1 check |
| Max wait timeout | 70 min | 100 min | +43% (prevents false failures) |

**Net effect**: The 10s waits add ~40-60 seconds per execution cycle, but prevent entire workflow failures. A single avoided failure saves the cost of 3-4 retry cycles (each ~200-300 credits). The credit ROI is very favorable.

## Related Files

- SKILL.md: Main skill instructions
- xms-export-automation/SKILL.md: The full XMS export automation workflow (this skill's optimizations are applied there)
- xms-cron-sync/SKILL.md: Cron payload synchronization guide
- webhook_config.json: Configuration file (contains `max_wait_minutes` setting)

## Change History

| Date | Change | Author |
|------|--------|--------|
| 2026-05-19 | Initial creation — extracted from 08:00 vs 09:11 root cause analysis | QoderWork |
