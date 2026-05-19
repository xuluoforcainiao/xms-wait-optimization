---
name: xms-wait-optimization
description: XMS system wait-time optimization strategy for browser automation. Provides a 10-second wait + verification pattern after every critical action to handle slow web application responses, including Chrome startup flags, empty-table retry logic, download-center validation, and Chrome security popup handling. Use when XMS export fails due to timing issues, when adding wait/verification steps to browser automation workflows, or when optimizing interactions with slow-loading internal systems.
---

# XMS 等待时间优化策略

## Overview

XMS (`cs-packet.i4px.com`) 系统响应慢，Agent 操作过快会导致页面未加载完成就执行下一步，引发失败。本策略通过**每个关键操作后等待 10 秒 + 验证**的节奏，匹配人工操作的稳定性。

## Chrome Startup Optimization

Launch Chrome with the safe-browsing flag to prevent download security popups:

```powershell
Start-Process 'C:\Program Files\Google\Chrome\Application\chrome.exe' `
  -ArgumentList '--safebrowsing-disable-download-protection'
```

- Wait 5 seconds for Chrome and the QoderWork Extension to connect.
- If the flag does not prevent the popup (e.g., after a Chrome update), the Agent falls back to auto-clicking "保留" (Keep). See "Chrome Security Popup Handling" below.

## Wait + Verify Pattern

After every critical action, follow this pattern:

1. **Wait 10 seconds** (use `Start-Sleep -Seconds 10` in PowerShell or equivalent)
2. **Verify page state** using one of the methods below
3. **Only proceed if verification passes**

### Verification Methods (Priority Order)

| Priority | Method | When to Use |
|----------|--------|-------------|
| 1 | DOM check (`javascript_tool`) | Check element count, text content, button state. Fast and reliable. |
| 2 | Screenshot (`computer`) | Use when DOM check is inconclusive or the result needs visual confirmation. |

> DOM checks are preferred because they consume far less credit than screenshots and work regardless of window size.

## Critical Action Checkpoints

### Checkpoint 1: After Navigation to opAbnormal

- **Action**: Click 「自有业务异常件管理」 in the sidebar menu (or force-navigate to `/opAbnormal`)
- **Wait**: 10 seconds
- **Verify**: Check `window.location.href.includes('opAbnormal')` + take a screenshot to confirm the page shows the filter panel + data table layout
- **If NOT correct**: Close tab → recreate → re-navigate → wait 10s → verify again

### Checkpoint 2: After Clearing Shelf Organization

- **Action**: Clear `txtUpShelfOgCodeShow` + `txtUpShelfOgCode` via JS or computer click
- **Wait**: Immediately proceed (no extra wait needed for this step)
- **Verify**: Confirm field shows placeholder "双击选择上架组织" or `value === ''`

### Checkpoint 3: After Clicking 查询

- **Action**: Click `abnormalSearch` button
- **Wait**: 10 seconds
- **Verify**: Use JS to check table row count:
  ```javascript
  document.querySelectorAll('table tbody tr, .next-table-row').length;
  ```
- **If rows === 0 or shows "暂无数据"**:
  - Wait 10 seconds
  - Click 查询 again
  - Wait 10 seconds, recheck
  - **Repeat up to 3 times total**
  - If still empty after 3 attempts → execute full recovery (close tab → re-login → re-navigate → re-clear → re-query)
- **If shows "拼命加载中…"**: Wait up to 30 seconds, then recheck

### Checkpoint 4: After Clicking 导出原因(新)

- **Action**: Click the export button
- **Wait**: 10 seconds (allow the export task to be created on the server)
- **Verify**: None directly — proceed to Checkpoint 5
- **If "导出失败" dialog appears**:
  - Click "确定" to close the dialog
  - Close tab → re-login → restart the entire workflow from Phase 1
  - This counts toward `max_retries`

### Checkpoint 5: After Opening 下载中心

- **Action**: Click "下载中心" button
- **Wait**: 10 seconds
- **Verify**: Check if the drawer shows a task with "导出状态" = "执行中":
  ```javascript
  const drawer = document.querySelector('.next-drawer') || document.querySelector('.drawer');
  const rows = drawer ? drawer.querySelectorAll('table tbody tr, .next-table-row') : [];
  rows.length; // Should be > 0
  ```
- **If drawer shows "没有数据" or empty**:
  - Close drawer
  - Execute full recovery: close tab → re-login → re-navigate → re-clear shelf org → re-query → re-export
  - After re-export, wait 10s, reopen download center to verify

### Checkpoint 6: After Clicking 下载

- **Action**: Click the download button in the drawer's first row
- **Wait**: 3 seconds
- **Verify**: Check for Chrome security warning popup (see below)
- **Then wait**: At least 15 seconds for Chrome to complete the `.crdownload` → `.xlsx` rename

## Chrome Security Popup Handling

After clicking download, check for a popup with text like "xxx.xlsx 可能存在风险" and "保留" / "放弃" buttons.

**If popup appears**:
```javascript
const keepBtn = Array.from(document.querySelectorAll('button'))
  .find(el => ['保留', 'Keep', '保留文件'].includes(el.textContent.trim()));
if (keepBtn) {
  keepBtn.dispatchEvent(new MouseEvent('click', { bubbles: true, cancelable: true }));
}
```

**If no popup appears**: Proceed to file processing (expected behavior when Chrome is launched with `--safebrowsing-disable-download-protection`).

## Export Timeout Adjustment

- **Setting**: `max_wait_minutes`
- **Recommended value**: `100` (based on historical data: 12-90 minutes)
- **Rationale**: A value of 70 causes "false failures" — the export succeeds on the server but the Agent gives up before it completes.

## Empty Table Retry Flowchart

```
Click 查询
  ↓
Wait 10s
  ↓
Check rows
  ├─ rows > 0 → Proceed to export
  └─ rows === 0
       ↓
  Wait 10s → Click 查询 again
       ↓
  Wait 10s → Check rows
       ├─ rows > 0 → Proceed
       └─ rows === 0
            ↓
       Wait 10s → Click 查询 again (3rd attempt)
            ↓
       Wait 10s → Check rows
            ├─ rows > 0 → Proceed
            └─ rows === 0 → Full recovery (close tab → re-login → restart)
```

## Full Recovery Procedure

When any checkpoint fails repeatedly (e.g., empty table after 3 retries, no export task in download center, "导出失败" dialog):

1. Close the current tab (`tabs_close_mcp`)
2. Create a new tab (`tabs_create_mcp`)
3. Navigate to `http://cs.packet.i4px.com/`
4. Log in (Phase 1)
5. Navigate to opAbnormal and clear shelf organization (Phase 2 Steps 1-5)
6. Query and verify data (Phase 2 Step 6)
7. Export (Phase 2 Step 7)

> **Key insight**: The full recovery (re-login + re-query) is more reliable than simplified retries (close tab + re-navigate only). Evidence: 2026-05-19 08:00 simplified retry failed, while 09:11 full recovery succeeded (0 rows → 14,772 rows).

## Key Principles

1. **Wait 10s after every critical action** — do not rush.
2. **DOM check first, screenshot fallback** — save credits.
3. **Empty table ≠ permanent failure** — retry up to 3 times before recovery.
4. **No export task in download center = server hasn't processed yet** — use full recovery, not just re-export.
5. **Chrome security popup is expected on some versions** — always check for it and handle it.
6. **Match human rhythm** — the user's manual workflow succeeds because they wait for the page to load between each step.
