---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [peripheral, power]
difficulty: intermediate
keywords: [HFCLK, LFCLK, 時脈源, 校準, 精度, 功耗, RTC]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-02
---

# 時脈管理：HFCLK/LFCLK 選擇、校準、精度

## 概述

nRF54L15 提供兩個獨立的時脈樹：**HFCLK**（64 MHz，系統和無線電）和 **LFCLK**（32.768 kHz，RTC 和深睡眠計時）。選擇合適的時脈源（晶振 vs RC）直接影響精度、功耗和成本。本指南涵蓋源選擇、Zephyr 配置和校準策略。

## 時脈源對比

| 特性 | HFCLK（64 MHz 晶振） | HFCLK（外部） | LFCLK（32.768 kHz 晶振） | LFCLK（RC） |
|------|------|------|------|------|
| 頻率精度 | ±20 ppm | ±10 ppm | ±20 ppm | ±20% ⚠️ |
| 啟動時間 | ~100 µs | ~10 ms | 1 s | <100 µs |
| 功耗（活躍） | ~10 mA | ~8 mA | ~1 µA | <0.5 µA |
| 成本 | 低 | 高 | 低 | 內建 |

**選擇原則**：
- **精度優先** → 晶振（32.768 kHz LFCLK + 64 MHz HFCLK）
- **功耗優先** → RC（LFCLK）+ 外部 HFCLK
- **成本優先** → 內建 RC（LFCLK） + 晶振（HFCLK）

## Devicetree + Kconfig 配置

### 1. HFCLK 選擇

**board-nrf54l15dk.overlay**：
```devicetree
&clock {
    hfclk-source = "internal";  /* "internal" (晶振) 或 "external" */
};
```

### 2. LFCLK 選擇

**board-nrf54l15dk.overlay**：
```devicetree
&clock {
    lfclk-source = "lfxo";  /* "lfxo" (晶振) 或 "lfrc" (RC) */
};
```

### 3. Kconfig 設定

**prj.conf**：
```
# 系統時脈設定
CONFIG_SYS_CLOCK_HW_CYCLES_PER_SEC=64000000
CONFIG_SYS_CLOCK_TICKS_PER_SEC=1000

# LFCLK 設定（RTC 和定時器）
CONFIG_CLOCK_CONTROL=y
CONFIG_CLOCK_CONTROL_NRF=y
CONFIG_CLOCK_CONTROL_NRF_DRIVER=y

# RTC 和定時器驅動
CONFIG_RTC=y
CONFIG_RTC_NRF=y

# 可選：時脈校準
CONFIG_CALIBRATION_NRF=y
CONFIG_CALIBRATION_NRF_INTERVAL=128  # 每 128 秒校準一次
```

## 程式碼範例

```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/clock_control.h>
#include <zephyr/drivers/clock_control/nrf_clock_control.h>
#include <zephyr/sys/timeutil.h>
#include <stdio.h>

// HFCLK 和 LFCLK 源初始化與監控
int main(void)
{
    const struct device *clk_dev = DEVICE_DT_GET(DT_NODELABEL(clock));

    // 驗證時脈設備可用
    if (!device_is_ready(clk_dev)) {
        printf("❌ 時脈設備未就緒\n");
        return -1;
    }

    // 取得當前 HFCLK 和 LFCLK 源
    struct nrf_clock_control_status status;
    clock_control_get_status(clk_dev, (clock_control_subsys_t)&status);

    printf("=== nRF54L15 時脈狀態 ===\n");
    printf("系統時脈頻率: %u Hz\n", sys_clock_hw_cycles_per_sec());

    // 啟動 HFCLK（如果尚未啟動）
    clock_control_on(clk_dev, (clock_control_subsys_t)CLOCK_CONTROL_NRF_SUBSYS_HF);
    printf("✓ HFCLK 已啟動\n");

    // 啟動 LFCLK（深睡眠計時所需）
    clock_control_on(clk_dev, (clock_control_subsys_t)CLOCK_CONTROL_NRF_SUBSYS_LF);
    printf("✓ LFCLK 已啟動\n");

    // 等待 LFCLK 穩定（通常 <1 秒）
    k_sleep(K_SECONDS(2));

    // 讀取當前時間戳記（64 位 µs）
    uint64_t now_us = k_ticks_to_us_near64(k_uptime_ticks());
    printf("系統運行時間: %llu µs\n", now_us);

    // 簡單的定時迴圈（測試精度）
    for (int i = 0; i < 5; i++) {
        uint64_t tick_start = k_uptime_ticks();
        k_sleep(K_MSEC(1000));
        uint64_t tick_end = k_uptime_ticks();

        uint64_t elapsed_us = k_ticks_to_us_near64(tick_end - tick_start);
        printf("第 %d 次：實測 %llu µs (期望 1000000 µs)\n",
               i + 1, elapsed_us);
    }

    printf("✓ 時脈管理測試完成\n");
    return 0;
}
```

## 常見問題

### Q1: LFCLK RC 校準不夠精確怎麼辦？
**A:** 使用內建校準機制（`CONFIG_CALIBRATION_NRF`），或定期將 RTC 與外部高精度時間源（NTP/GPS）同步。

### Q2: 切換 LFCLK 源（晶振 ↔ RC）需要重啟嗎？
**A:** 在執行時無法切換。必須在裝置樹和重新編譯時指定。對於生產，使用 OTA 更新。

### Q3: HFCLK 啟動時的 ~100 µs 延遲會影響無線電嗎？
**A:** 不會。Zephyr 和 Nordic 驅動會自動預留 startup 緩衝。無線事件會等待 HFCLK 就緒。

## API 快速參考

```c
/* 取得系統時脈頻率（Hz） */
uint32_t freq = sys_clock_hw_cycles_per_sec();

/* 取得當前運行時間（毫秒） */
int64_t uptime_ms = k_uptime_get();

/* 轉換 tick → 微秒 */
uint64_t us = k_ticks_to_us_near64(ticks);

/* 取得時脈設備控制控制代碼 */
const struct device *clk = DEVICE_DT_GET(DT_NODELABEL(clock));

/* 啟用 HFCLK 或 LFCLK */
clock_control_on(clk, CLOCK_CONTROL_NRF_SUBSYS_HF);
clock_control_on(clk, CLOCK_CONTROL_NRF_SUBSYS_LF);
```

---

**精度測試建議**：每次功耗優化或時脈源變更後，運行時脈精度測試（對比 1000 次延遲平均值）。
