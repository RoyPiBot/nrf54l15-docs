---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [build-system, peripheral]
difficulty: intermediate
keywords: [board porting, devicetree, custom hardware, pin mapping]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-03
---

# 自訂電路板移植：從 DK 到量產板的完整流程

## 概述
本指南介紹如何從官方開發板（DK）移植到自訂電路板。包含 Pin mapping 調整、Devicetree overlay 編寫、Kconfig 設定、以及常見陷阱排查。

---

## 1. Devicetree 與 Pin Mapping

**核心概念**：自訂板需要新建 `.overlay` 檔案，覆蓋預設的 Pin 定義與週邊設定。

### 自訂板目錄結構
```
boards/
├── nrf54l15dk/
├── my_custom_board/           # 新增
│   ├── board.cmake
│   ├── board.yml
│   ├── nrf54l15_cpuapp.dts
│   └── nrf54l15_cpuapp.yaml
```

### nrf54l15_cpuapp.dts 最小範例
```devicetree
/dts-v1/;
#include "nrf54l15_cpuapp.dtsi"

/ {
    model = "Custom Board";

    /* 覆蓋 LED GPIO */
    leds {
        compatible = "gpio-leds";
        led0: led_0 {
            gpios = <&gpio0 5 GPIO_ACTIVE_HIGH>;  // 改為 GPIO 5（原DK為 GPIO 7）
            label = "Green LED";
        };
    };

    /* 覆蓋按鍵 GPIO */
    buttons {
        compatible = "gpio-keys";
        button0: button_0 {
            gpios = <&gpio0 25 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>;
            label = "Reset";
        };
    };

    /* UART 重新對應 */
    chosen {
        zephyr,console = &uart0;
        zephyr,shell-uart = &uart0;
    };
};

&uart0 {
    status = "okay";
    current-speed = <115200>;
    pinctrl-0 = <&uart0_default>;
    pinctrl-1 = <&uart0_sleep>;
    pinctrl-names = "default", "sleep";
};

/* Pin 控制配置 */
&pinctrl {
    uart0_default: uart0_default {
        group1 {
            psels = <NRF_PSEL(UART_TX, 0, 20)>,   // TX 改為 P0.20
                    <NRF_PSEL(UART_RX, 0, 22)>;   // RX 改為 P0.22
        };
    };
    uart0_sleep: uart0_sleep {
        group1 {
            psels = <NRF_PSEL(UART_TX, 0, 20)>,
                    <NRF_PSEL(UART_RX, 0, 22)>;
            low-power-enable;
        };
    };
};
```

---

## 2. Kconfig 設定

### prj.conf（產品組態）
```ini
# 基礎設定
CONFIG_NRF_MODEM=y
CONFIG_UART_CONSOLE=y
CONFIG_SERIAL=y

# LED 與按鍵驅動
CONFIG_GPIO=y
CONFIG_LED=y

# 針對自訂硬體調整（如有不同的晶振頻率）
CONFIG_CLOCK_CONTROL=y
CONFIG_SYS_CLOCK_HW_CYCLES_PER_SEC=320000000
```

### board.cmake
```cmake
board_runner_args(nrfjprog "--nrf-family=NRF54L15")
```

---

## 3. 程式碼範例：LED + 按鍵測試

```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/gpio.h>
#include <zephyr/drivers/uart.h>

/* 取得 Device Structure */
static const struct gpio_dt_spec led = GPIO_DT_SPEC_GET(DT_ALIAS(led0), gpios);
static const struct gpio_dt_spec button = GPIO_DT_SPEC_GET(DT_ALIAS(button0), gpios);

/* 按鍵回呼 */
void button_pressed(const struct device *dev, struct gpio_callback *cb, uint32_t pins) {
    printk("按鍵按下\n");
    gpio_pin_toggle_dt(&led);  // 切換 LED
}

int main(void) {
    struct gpio_callback button_cb_data;

    if (!gpio_is_ready_dt(&led)) {
        printk("LED 設備未就緒\n");
        return -1;
    }
    if (!gpio_is_ready_dt(&button)) {
        printk("按鍵設備未就緒\n");
        return -1;
    }

    /* 初始化 LED */
    gpio_pin_configure_dt(&led, GPIO_OUTPUT);
    gpio_pin_set_dt(&led, 0);

    /* 初始化按鍵與中斷 */
    gpio_pin_configure_dt(&button, GPIO_INPUT);
    gpio_init_callback(&button_cb_data, button_pressed, BIT(button.pin));
    gpio_add_callback(button.port, &button_cb_data);
    gpio_pin_interrupt_configure_dt(&button, GPIO_INT_EDGE_TO_ACTIVE);

    printk("自訂板啟動完成 - 等待按鍵\n");

    while (1) {
        k_sleep(K_MSEC(1000));
    }

    return 0;
}
```

---

## 4. 常見問題 (FAQ)

**Q: Devicetree 修改後仍找不到 GPIO？**
A: 檢查 `gpio_dt_spec_get()` 是否正確對應 DTS 中的 alias。執行 `west build -b my_custom_board` 時，系統會自動編譯 `.overlay`。若無效果，清除 build 資料夾重新編譯。

**Q: UART 無輸出，只有亂碼？**
A: 確認 Pinctrl 的 Pin 編號無誤；檢查波特率設定（prj.conf 中的 CONFIG_UART_CONSOLE_ON_DEV_NAME）；驗證 DTS 中 `current-speed` 與開發環境配置一致（通常 115200）。

**Q: 如何在多個自訂板間快速切換？**
A: 使用 `west build -b <board_name>` 指定不同的板子；或在 `CMakeLists.txt` 中設 `set(BOARD my_custom_board)` 預設值。

---

## 5. API 快速參考

| 函式 | 用途 |
|------|------|
| `GPIO_DT_SPEC_GET(node, property)` | 從 DTS 取得 GPIO pin 定義 |
| `gpio_pin_configure_dt(&spec, flags)` | 設定 GPIO 模式（輸入/輸出） |
| `gpio_pin_set_dt(&spec, value)` | 設定 GPIO 電位 |
| `gpio_pin_toggle_dt(&spec)` | 切換 GPIO 狀態 |
| `gpio_pin_interrupt_configure_dt(&spec, flags)` | 設定中斷邊界觸發 |

---

## 移植檢查清單

- [ ] Devicetree overlay 檔案已建立（`.dts` 或 `.dtsi`）
- [ ] Pin 編號已驗證與實際硬體一致
- [ ] Pinctrl 配置中 TX/RX（UART）或 SDA/SCL（I2C）已對應新 Pin 編號
- [ ] Kconfig 預設值（晶振、電源、模組）已與自訂板規格相符
- [ ] 編譯無警告或錯誤（`west build -b my_custom_board`）
- [ ] 硬體測試：LED 點亮、按鍵回應、UART 輸出正常
