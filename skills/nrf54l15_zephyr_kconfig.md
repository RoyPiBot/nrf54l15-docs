---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [build-system]
difficulty: [intermediate]
keywords: [Kconfig, prj.conf, CONFIG_, 功能開關, 記憶體優化]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-02
---

# Zephyr Kconfig 設定系統

## 概述

Kconfig 是 Zephyr 的層級設定系統，透過 `prj.conf` 文件控制功能開關、記憶體大小、編譯優化等。編譯時配置，無需修改代碼。

## Devicetree + Kconfig

### prj.conf
```ini
# 記憶體
CONFIG_HEAP_MEM_POOL_SIZE=4096

# 功能
CONFIG_GPIO=y
CONFIG_UART_CONSOLE=y
CONFIG_LOG=y
CONFIG_LOG_DEFAULT_LEVEL=2

# 電源
CONFIG_PM=y
```

### nrf54l15dk.overlay
```dts
/ {
    chosen {
        zephyr,console = &uart0;
    };
};

&uart0 {
    status = "okay";
    current-speed = <115200>;
};

&gpio0 {
    status = "okay";
};
```

## 程式碼範例

```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/gpio.h>
#include <zephyr/logging/log.h>

LOG_MODULE_REGISTER(kconfig_demo);

#ifdef CONFIG_GPIO
#define LED0_NODE DT_ALIAS(led0)
static const struct gpio_dt_spec led = GPIO_DT_SPEC_GET(LED0_NODE, gpios);

static void init_led(void) {
    // 檢查 GPIO 設備是否就緒
    if (!gpio_is_ready_dt(&led)) return;
    // 設定為輸出腳
    gpio_pin_configure_dt(&led, GPIO_OUTPUT);
}

static void toggle_led(void) {
    gpio_pin_toggle_dt(&led);
}
#else
// GPIO 已禁用，提供空實作
static void init_led(void) { }
static void toggle_led(void) { }
#endif

int main(void) {
    // CONFIG_HEAP_MEM_POOL_SIZE 由 prj.conf 定義，編譯時常數
    LOG_INF("堆記憶體: %d bytes", CONFIG_HEAP_MEM_POOL_SIZE);

    init_led();

    while (1) {
        toggle_led();
        k_sleep(K_MSEC(1000));
        LOG_INF("心跳 - 記錄等級: %d", CONFIG_LOG_DEFAULT_LEVEL);
    }

    return 0;
}
```

## 常見問題

**Q：如何檢視編譯後的所有 CONFIG 值？**
```bash
find build -name "autoconf.h" -exec grep "CONFIG_" {} \; | head -20
```

**Q：prj.conf 與 Kconfig 衝突怎麼辦？**
prj.conf 優先級最高，會覆蓋預設值。確保選項在板級 Kconfig 中已定義。

**Q：如何根據 CONFIG 條件編譯不同代碼？**
使用 `#ifdef CONFIG_FEATURE` 檢查布林值，或 `#if CONFIG_VALUE > 100` 檢查整數值。

## API 快速參考

| CONFIG 選項 | 說明 |
|-------------|------|
| `CONFIG_FEATURE` | 布林值，用 `#ifdef` 檢查啟用狀態 |
| `CONFIG_HEAP_MEM_POOL_SIZE` | 堆記憶體大小（位元組） |
| `CONFIG_LOG_DEFAULT_LEVEL` | 預設記錄等級（0=NONE, 1=ERR, 2=INF, 3=DBG） |
| `CONFIG_MAIN_STACK_SIZE` | 主堆棧大小 |
| `CONFIG_PM` | 電源管理功能開關 |

## 進階：記憶體優化配置

```ini
# 最小化配置用於受限環境
CONFIG_HEAP_MEM_POOL_SIZE=2048
CONFIG_MAIN_STACK_SIZE=2048
CONFIG_ISR_STACK_SIZE=512
CONFIG_SYSTEM_WORKQUEUE_STACK_SIZE=1024
CONFIG_LOG=n              # 禁用 LOG 以節省記憶體
CONFIG_PM=y               # 啟用電源管理
```

## 檔案組織

```
apps/myapp/
├── CMakeLists.txt
├── prj.conf              ← 編譯時設定
├── src/
│   └── main.c
└── boards/
    └── nrf54l15dk.overlay    ← 板級設定覆蓋
```
