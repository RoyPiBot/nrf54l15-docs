---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [peripheral]
difficulty: [intermediate]
keywords: [watchdog, timeout, system reset, feed watchdog, reliability]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-01
---

# nRF54L15 看門狗計時器（Watchdog Timer）

## 概述

看門狗計時器用於監視系統執行狀況。若軟體陷入無限迴圈或崩潰無法回應，計時器超時時將自動觸發系統重置，確保系統健康性。透過定期"餵狗"（刷新計時器），應用程式證明自身運行正常。

## Devicetree + Kconfig

**nrf54l15dk.overlay**
```devicetree
/ {
    chosen {
        zephyr,watchdog = &wdt;
    };
};

&wdt {
    status = "okay";
};
```

**prj.conf**
```
CONFIG_WATCHDOG=y
CONFIG_WATCHDOG_LOG_LEVEL_INF=y
```

## 程式碼範例

```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/watchdog.h>
#include <zephyr/logging/log.h>

LOG_MODULE_REGISTER(watchdog_demo, LOG_LEVEL_INF);

static const struct device *wdt = DEVICE_DT_GET(DT_NODELABEL(wdt));

static int feed_watchdog(void) {
    int ret = wdt_feed(wdt, 0);
    if (ret != 0) {
        LOG_ERR("看門狗餵食失敗: %d", ret);
        return ret;
    }
    LOG_INF("看門狗已餵食");
    return 0;
}

int main(void) {
    LOG_INF("看門狗範例開始");

    if (!device_is_ready(wdt)) {
        LOG_ERR("看門狗裝置未就緒");
        return -ENODEV;
    }

    // 設定看門狗：超時 5 秒，超時後系統重置
    struct wdt_timeout_cfg timeout_config = {
        .window = {
            .min = 0U,
            .max = 5000U,  // 5000 毫秒 = 5 秒
        },
        .flags = WDT_FLAG_RESET_SOC,  // 超時時重置 SoC
    };

    int channel_id = wdt_install_timeout(wdt, &timeout_config);
    if (channel_id < 0) {
        LOG_ERR("安裝看門狗超時失敗: %d", channel_id);
        return channel_id;
    }

    // 啟動看門狗
    int ret = wdt_setup(wdt, 0);
    if (ret != 0) {
        LOG_ERR("設定看門狗失敗: %d", ret);
        return ret;
    }

    LOG_INF("看門狗已啟動，超時 5 秒");

    // 每 3 秒餵狗一次（小於 5 秒超時）
    for (int i = 0; i < 10; i++) {
        k_sleep(K_SECONDS(3));
        feed_watchdog();
    }

    LOG_INF("看門狗範例完成");
    return 0;
}
```

## 常見問題

**Q: 如何設定超時時間？**
A: 在 `timeout_config.window.max` 設定毫秒數，例如 5000 = 5 秒。系統必須在此時間內至少呼叫一次 `wdt_feed()`。

**Q: 餵狗頻率應設多少？**
A: 建議設為超時時間的 1/3 至 1/2。例如超時 5 秒，每 2-2.5 秒餵狗一次，留下安全邊界。

**Q: 超時後系統如何重置？**
A: 設定 `WDT_FLAG_RESET_SOC` 標誌可硬體重置整個 SoC。若未餵狗，計時器溢位立即觸發重置動作。

## API 快速參考

```c
// 取得看門狗裝置
const struct device *wdt = DEVICE_DT_GET(DT_NODELABEL(wdt));

// 檢查裝置就緒
bool device_is_ready(const struct device *dev);

// 安裝超時設定（回傳通道 ID）
int wdt_install_timeout(const struct device *dev,
                        const struct wdt_timeout_cfg *cfg);

// 啟動看門狗
int wdt_setup(const struct device *dev, uint8_t options);

// 餵狗（刷新計時器，防止重置）
int wdt_feed(const struct device *dev, int channel_id);
```

---

**目標板**：nrf54l15dk/nrf54l15/cpuapp
**建構方式**：`west build -b nrf54l15dk/nrf54l15/cpuapp -- -DOVERLAY_FILE=nrf54l15dk.overlay`
