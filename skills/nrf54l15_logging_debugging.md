---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [tool, power]
difficulty: intermediate
keywords: [RTT, logging, debugging, core dump, segger, J-Link]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-02
---

# nRF54L15 日誌與除錯：RTT 即時追蹤、核心傾印、除錯技巧

## 概述

**RTT (Real-Time Transfer)** 是 Nordic/Segger 提供的即時日誌和除錯工具，無需 UART，直接透過 J-Link 偵錯器經 SWD 接口傳輸日誌。支援多達 16 個獨立的上行（log 輸出）和下行（命令輸入）通道。nRF54L15 內建 SEGGER RTT 支援，可用於實時追蹤變數、效能分析、核心傾印蒐集和例外處理。

## Devicetree + Kconfig

### prj.conf
```ini
# 啟用 RTT 日誌後端
CONFIG_RTT_CONSOLE=y
CONFIG_USE_SEGGER_RTT=y

# 設定 RTT 緩衝區大小（KB）
CONFIG_SEGGER_RTT_BUFFER_SIZE_UP=1024
CONFIG_SEGGER_RTT_BUFFER_SIZE_DOWN=16

# 啟用核心傾印支援
CONFIG_KERNEL_MEM_PROTECTION=y
CONFIG_EXCEPTION_FATAL_ERROR_HOOKS=y

# 可選：延遲日誌模式，減輕 MCU 負擔
CONFIG_LOG_MODE_DEFERRED=y
CONFIG_LOG_PROCESSING=y
```

### 無需 .overlay
RTT 在 SoC 層級自動啟用，無需 devicetree 修改。

## 程式碼範例

```c
#include <zephyr/kernel.h>
#include <zephyr/logging/log.h>
#include <zephyr/sys/printk.h>
#include <zephyr/arch/arm/aarch32/cortex_m/cmsis.h>

LOG_MODULE_REGISTER(demo, LOG_LEVEL_INF);

// 自訂例外鉤子（核心傾印捕捉）
void exception_hook(unsigned int reason, const z_arch_esf_t *esf) {
    LOG_ERR("=== 例外偵測 ===");
    LOG_ERR("reason: %d, PC: 0x%x, SP: 0x%x",
            reason, esf->basic.pc, esf->basic.sp);
    // 可在此進行額外的狀態蒐集
}

// 系統監控執行緒
void monitor_thread(void) {
    while (1) {
        uint32_t tick = k_uptime_get_32();
        uint32_t mem_free = k_mem_free_get();

        // 定期輸出系統狀態
        LOG_INF("[Monitor] uptime=%u ms, mem_free=%u bytes",
                tick, mem_free);

        k_sleep(K_SECONDS(5));
    }
}

K_THREAD_DEFINE(mon_tid, 1024, monitor_thread, NULL, NULL, NULL,
                7, 0, 0);

int main(void) {
    LOG_INF("=== nRF54L15 RTT 除錯示例啟動 ===");

    // 測試各級日誌
    for (int i = 0; i < 3; i++) {
        LOG_DBG("迴圈 %d: 偵錯訊息", i);
        LOG_INF("迴圈 %d: 訊息等級", i);
        LOG_WRN("迴圈 %d: 警告等級", i);
        k_sleep(K_MSEC(500));
    }

    LOG_ERR("測試錯誤級別");

    // 示範：可手動觸發核心傾印（取消註解測試）
    // k_fatal_halt(K_ERR_KERNEL_PANIC);

    LOG_INF("主程式進入睡眠（監控執行緒運行中）...");
    k_sleep(K_FOREVER);

    return 0;
}
```

## 常見問題

**Q: RTT 連線超時或無法取得日誌？**
確認 J-Link 驅動版本 ≥ 7.50，檢查 SWD 接線（SWCLK、SWDIO、GND）。嘗試 `nrfjprog -i` 列舉裝置。如用 J-Link RTT Viewer，需載入正確的 ELF 檔案並設定 RTT 搜尋範圍。

**Q: 日誌丟失或輸出卡頓？**
增加 RTT 緩衝區大小（`CONFIG_SEGGER_RTT_BUFFER_SIZE_UP`）、改用延遲日誌模式（`CONFIG_LOG_MODE_DEFERRED`）、降低日誌頻率。高速日誌需要足夠的緩衝空間和 RTT 讀取頻率。

**Q: 如何蒐集並分析核心傾印？**
配置 `CONFIG_KERNEL_MEM_PROTECTION` 和 `CONFIG_EXCEPTION_FATAL_ERROR_HOOKS`。觸發例外後，用 `nrfjprog --readcode dump.bin` 提取原始記憶體。搭配 GDB 和 ELF 符號表（`arm-none-eabi-gdb`）分析堆疊追蹤。

## API 快速參考

| 函式 | 用途 |
|------|------|
| `LOG_INF(fmt, ...)` | 列印 INFO 級日誌 |
| `LOG_DBG(fmt, ...)` | 列印 DEBUG 級日誌 |
| `LOG_ERR(fmt, ...)` | 列印 ERROR 級日誌 |
| `k_uptime_get_32()` | 取得系統啟動後時間（ms） |
| `k_mem_free_get()` | 取得可用動態記憶體（bytes） |
| `SEGGER_RTT_WriteString(0, str)` | 直接寫入 RTT（低階 API） |
| `k_fatal_halt(code)` | 手動觸發核心傾印 |

---

**延伸資源**
- Zephyr Logging: https://docs.zephyrproject.org/latest/services/logging/index.html
- Nordic nRF54L15 Datasheet: https://infocenter.nordicsemi.com/
- SEGGER RTT 文件（J-Link 附帶）
