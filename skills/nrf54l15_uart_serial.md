---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [peripheral]
difficulty: [intermediate]
keywords: [UART, 串口, DMA, 波特率, 非阻塞]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-03-31
---

# UART 串口通訊：設定波特率、DMA 模式、收發資料

## 概述
UART 是異步串列通訊介面，用於裝置間點對點資料傳輸。nRF54L15 支援多個 UART 實例，可配置波特率（9600～1M bps）、DMA 傳輸以降低 CPU 負載。本文涵蓋從基礎配置到非阻塞收發的完整流程。

## Devicetree 配置

編輯 `boards/nrf54l15dk_nrf54l15_cpuapp.overlay`：

```devicetree
/ {
	chosen {
		zephyr,console = &uart0;
	};
};

&uart0 {
	status = "okay";
	current-speed = <115200>;
	pinctrl-0 = <&uart0_default>;
	pinctrl-1 = <&uart0_sleep>;
	pinctrl-names = "default", "sleep";
};

&pinctrl {
	uart0_default: uart0_default {
		group1 {
			psels = <NRF_PSEL(UART_TX, 0, 26)>,
				<NRF_PSEL(UART_RX, 0, 27)>;
		};
	};
	uart0_sleep: uart0_sleep {
		group1 {
			psels = <NRF_PSEL(UART_TX, 0, 26)>,
				<NRF_PSEL(UART_RX, 0, 27)>;
			low-power-enable;
		};
	};
};
```

## Kconfig 配置

編輯 `prj.conf`：

```ini
# 基礎 UART
CONFIG_CONSOLE=y
CONFIG_UART_CONSOLE=y
CONFIG_SERIAL=y
CONFIG_UART_0=y

# 非阻塞與 DMA
CONFIG_UART_ASYNC_API=y
CONFIG_UART_0_ASYNC_RX_DMA=y
CONFIG_UART_0_ASYNC_TX_DMA=y
CONFIG_DMA=y

# 日誌
CONFIG_LOG=y
CONFIG_LOG_BACKEND_UART=y
```

## 程式碼範例

```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/uart.h>
#include <zephyr/logging/log.h>

LOG_MODULE_REGISTER(uart_demo, LOG_LEVEL_INF);

const struct device *uart_dev = DEVICE_DT_GET(DT_CHOSEN(zephyr_console));

#define RX_BUF_SIZE 256
static uint8_t rx_buf[RX_BUF_SIZE];
static uint8_t tx_msg[] = "UART with DMA enabled\r\n";

// UART 事件回調函式
static void uart_event_handler(const struct device *dev,
                               struct uart_event *evt,
                               void *user_data) {
	switch (evt->type) {
	case UART_RX_RDY:
		// 接收到資料，長度在 evt->data.rx.len
		LOG_INF("接收 %d 位元組", evt->data.rx.len);
		break;
	case UART_RX_DISABLED:
		LOG_INF("RX 已停止");
		break;
	case UART_TX_DONE:
		LOG_INF("TX 完成");
		break;
	case UART_RX_BUF_RELEASED:
		// DMA 緩衝區已釋放，可重新使用
		break;
	default:
		break;
	}
}

int main(void) {
	if (!device_is_ready(uart_dev)) {
		LOG_ERR("UART 裝置未就緒");
		return -1;
	}

	// 設定波特率（可選，覆蓋 DT 設定）
	struct uart_config cfg = {
		.baudrate = 115200,
		.parity = UART_CFG_PARITY_NONE,
		.stop_bits = UART_CFG_STOP_BITS_1,
		.data_bits = UART_CFG_DATA_BITS_8,
		.flow_ctrl = UART_CFG_FLOW_CTRL_NONE,
	};
	uart_configure(uart_dev, &cfg);

	// 註冊非同步回調，啟用 DMA 接收
	uart_callback_set(uart_dev, uart_event_handler, NULL);
	uart_rx_enable(uart_dev, rx_buf, RX_BUF_SIZE, 100000); // 100ms 超時

	// DMA 傳送
	uart_tx(uart_dev, tx_msg, sizeof(tx_msg) - 1, K_FOREVER);

	// 主迴圈
	while (1) {
		k_sleep(K_SECONDS(2));
		LOG_INF("系統運行中...");
	}

	return 0;
}
```

## 常見問題

**Q: 波特率設定無效？**
確認 Devicetree `current-speed` 與晶片能力相符。nRF54L15 支援 9600/19200/38400/57600/115200/230400/460800/921600 bps。若需動態修改，使用 `uart_configure()` API。

**Q: DMA 傳輸為什麼失敗？**
檢查 `CONFIG_UART_ASYNC_API=y` 與相應的 `CONFIG_UART_0_ASYNC_RX_DMA=y`、`CONFIG_UART_0_ASYNC_TX_DMA=y` 是否啟用，確保 DMA 控制器可用。

**Q: 如何處理接收超時？**
`uart_rx_enable()` 最後參數為超時（微秒），超時後觸發 `UART_RX_DISABLED` 事件，需重新呼叫 `uart_rx_enable()` 恢復接收。

## API 快速參考

| 函式 | 說明 |
|------|------|
| `uart_configure(dev, &cfg)` | 設定波特率、停止位、奇偶校驗、流控 |
| `uart_tx(dev, buf, len, timeout)` | 傳送資料（DMA 或同步模式） |
| `uart_rx_enable(dev, buf, len, timeout_us)` | 啟用非阻塞接收 |
| `uart_callback_set(dev, callback, user_data)` | 註冊事件回調函式 |
| `uart_rx_disable(dev)` | 停止接收 |
