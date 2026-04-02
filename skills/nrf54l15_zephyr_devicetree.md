---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [build-system, tool]
difficulty: intermediate
keywords: [devicetree, overlay, kconfig, DTS, board-level-configuration]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-02
---

# Zephyr Devicetree：覆蓋檔、板級設定、硬體抽象

## 概述
Devicetree 是 Zephyr 的硬體描述語言，用於在編譯時定義外設配置（GPIO、UART、SPI 等）。覆蓋檔（.overlay）和 Kconfig 結合，允許不修改主板級定義即可靈活調整硬體。

## Devicetree + Kconfig

### 1. 覆蓋檔 (`boards/nrf54l15dk.overlay`)
```dts
/* 啟用 UART 0，設定引腳 */
&uart0 {
	status = "okay";
	current-speed = <115200>;
	pinctrl-0 = <&uart0_default>;
	pinctrl-names = "default";
};

/* 定義 GPIO 按鈕 */
/ {
	buttons {
		compatible = "gpio-keys";
		button0: button_0 {
			gpios = <&gpio0 4 GPIO_ACTIVE_LOW>;
			label = "Button 0";
		};
	};
};
```

### 2. Kconfig (`prj.conf`)
```
# 啟用基礎功能
CONFIG_CONSOLE=y
CONFIG_UART_CONSOLE=y
CONFIG_SERIAL=y
CONFIG_GPIO=y
CONFIG_PINCTRL=y

# 啟用 GPIO 驅動
CONFIG_GPIO_NRF_P0=y

# 可選：列表驅動
CONFIG_DT_HAS_ARM_CORTEX_M_NVIC_ENABLED=y
```

## 程式碼範例

```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/gpio.h>
#include <zephyr/drivers/uart.h>

/* 透過 Devicetree 別名取得設備 */
#define BUTTON_NODE DT_ALIAS(button0)
#define LED_NODE DT_ALIAS(led0)

/* GPIO 規範結構（由 Devicetree 編譯產生） */
static const struct gpio_dt_spec button =
	GPIO_DT_SPEC_GET(BUTTON_NODE, gpios);

static void button_isr_handler(
	const struct device *dev,
	struct gpio_callback *cb,
	uint32_t pins)
{
	/* 按鈕中斷回呼 */
	printk("按鈕按下\n");
}

int main(void)
{
	struct gpio_callback button_cb_data;

	/* 檢驗 GPIO 設備是否存在 */
	if (!device_is_ready(button.port)) {
		printk("GPIO 設備未就緒\n");
		return -1;
	}

	/* 配置 GPIO 方向 */
	gpio_pin_configure_dt(&button, GPIO_INPUT);

	/* 設定中斷觸發方式 */
	gpio_pin_interrupt_configure_dt(&button,
		GPIO_INT_EDGE_FALLING);

	/* 註冊中斷回呼 */
	gpio_init_callback(&button_cb_data,
		button_isr_handler,
		BIT(button.pin));
	gpio_add_callback(button.port, &button_cb_data);

	/* 主迴圈 */
	while (1) {
		k_sleep(K_SECONDS(1));
	}

	return 0;
}
```

## 常見問題

**Q: .overlay 和 .dts 有何差別？**
A: `.dts` 是完整的板級定義；`.overlay` 是片段，覆蓋或擴展 `.dts`。應用層通常用 `.overlay` 自訂。

**Q: Devicetree 別名（alias）是什麼？**
A: `DT_ALIAS(led0)` 透過別名取得設備節點，避免硬編碼路徑。在根節點中定義：`led0 = &gpio0_3;`

**Q: CONFIG_PINCTRL_DYNAMIC 何時需要啟用？**
A: 執行時需動態改變引腳配置時啟用（如切換 UART 到不同引腳）。靜態設定不需要。

## API 快速參考

| 函式 | 功能 |
|------|------|
| `GPIO_DT_SPEC_GET(node, gpios)` | 從 Devicetree 取得 GPIO 規範 |
| `gpio_pin_configure_dt(spec, flags)` | 設定 GPIO 方向與模式 |
| `gpio_pin_set_dt(spec, value)` | 設定 GPIO 輸出電平 |
| `gpio_pin_get_dt(spec)` | 讀取 GPIO 輸入電平 |
| `gpio_pin_interrupt_configure_dt(spec, flags)` | 配置 GPIO 中斷邊沿 |
