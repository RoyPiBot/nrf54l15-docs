---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [peripheral, power]
difficulty: intermediate
keywords: [COMP, comparator, threshold detection, edge detection, low-power monitoring]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-02
---

# nRF54L15 類比比較器 (COMP)：閾值設定、交越偵測、低功耗監控

## 概述

COMP (Analog Comparator) 用於比較兩個類比輸入訊號，當超越設定的閾值時觸發中斷。
適用於電壓監控、光線感測、低功耗喚醒等應用。

## Devicetree 設定

在 `nrf54l15dk-nrf54l15-cpuapp.overlay` 中啟用 COMP：

```devicetree
&comp {
    status = "okay";
};
```

Kconfig (`prj.conf`):

```ini
CONFIG_COMP=y
CONFIG_COMP_NRFX=y
CONFIG_GPIO=y
CONFIG_LOG=y
CONFIG_LOG_DEFAULT_LEVEL=3
```

## 程式碼範例

```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/adc.h>
#include <zephyr/logging/log.h>

LOG_MODULE_REGISTER(comparator_demo);

// COMP 相關的暫存器定義（通常由 Nordic SDK 提供）
#define COMP_BASE           0x41006000U
#define COMP_ENABLE         0x500

volatile uint32_t *comp_base = (volatile uint32_t *)COMP_BASE;

// COMP 中斷處理函數
void comp_interrupt_handler(void)
{
    // 讀取中斷原因（RESULT 暫存器）
    uint32_t result = *(volatile uint32_t *)(COMP_BASE + 0x100);

    LOG_INF("COMP 交越偵測：結果 = 0x%x", result);

    // 清除中斷狀態（寫 1 清除）
    *(volatile uint32_t *)(COMP_BASE + 0x10C) = 0x1;
}

void comparator_init(void)
{
    const struct device *comp_dev = DEVICE_DT_GET_OR_NULL(DT_NODELABEL(comp));

    if (!comp_dev) {
        LOG_ERR("COMP 裝置初始化失敗");
        return;
    }

    // 啟用 COMP
    *(volatile uint32_t *)(COMP_BASE + COMP_ENABLE) = 1;

    // 設定 PSEL（正輸入）= AIN0，NSEL（負輸入）= 參考電壓
    *(volatile uint32_t *)(COMP_BASE + 0x000) = 0;  // PSEL = AIN0
    *(volatile uint32_t *)(COMP_BASE + 0x004) = 4;  // NSEL = VDD/2

    // 設定閾值敏感性（交越模式 = 兩向邊沿偵測）
    *(volatile uint32_t *)(COMP_BASE + 0x008) = 0;  // 雙邊沿模式

    // 啟用中斷
    *(volatile uint32_t *)(COMP_BASE + 0x104) = 0x1;  // INTENSET = READY

    LOG_INF("類比比較器已初始化（閾值監控已啟用）");
}

void main(void)
{
    comparator_init();

    while (1) {
        LOG_INF("系統執行中，等待 COMP 事件...");
        k_sleep(K_SECONDS(5));
    }
}
```

## Kconfig 說明

- `CONFIG_COMP=y` — 啟用類比比較器驅動
- `CONFIG_COMP_NRFX=y` — 使用 Nordic nrfx 後端

## 常見問題

**Q1：如何改變比較閾值？**
修改 `PSEL` 和 `NSEL` 暫存器。NSEL 可設為：
- `0` = GND
- `1` = VDD/8
- `4` = VDD/2
- `7` = VDD

**Q2：COMP 能用於喚醒系統嗎？**
是的，配置為 `CONFIG_GPIO_AS_PINCTRL=y` 後，COMP 的中斷可喚醒休眠的 CPU。

**Q3：COMP 消耗多少功率？**
典型 ~10 µA @ 3.3V（常開模式）。關閉外設時功耗更低。

## API 快速參考

| 函式 | 說明 |
|------|------|
| `device_get_binding("COMP")` | 取得 COMP 裝置句柄 |
| `COMP->ENABLE = 1` | 啟用比較器 |
| `COMP->INTENSET = (1 << 0)` | 啟用準備中斷 |
| `COMP->PSEL` | 設定正輸入通道 |
| `COMP->NSEL` | 設定負輸入（參考電壓） |

