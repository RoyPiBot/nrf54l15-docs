---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [wireless, peripheral, power]
difficulty: intermediate
keywords: [IoT Gateway, BLE, UART, Sensor Data, nRF54L15, Raspberry Pi]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-03
---

# NanoClaw IoT 閘道器：用 Pi + nRF54L15 建構智慧閘道

## 概述

NanoClaw IoT 閘道器是一個混合架構系統，利用 Raspberry Pi 為主控中樞，nRF54L15 為無線端點，實現：
- 遠端感測器資料的 BLE 無線收集
- 本地資料處理與雲端同步
- 低功耗邊緣計算能力

## Devicetree + Kconfig

**prj.conf**
```
CONFIG_BT=y
CONFIG_BT_CENTRAL=y
CONFIG_BT_GATT_CLIENT=y
CONFIG_UART=y
CONFIG_LOG=y
CONFIG_LOG_MODE_IMMEDIATE=y
CONFIG_SERIAL=y
```

**nrf54l15dk_nrf54l15_cpuapp.overlay**
```dts
&uart0 {
	status = "okay";
	current-speed = <115200>;
	tx-pin = <0 5>;
	rx-pin = <0 6>;
};

&timer0 {
	status = "okay";
};

&radio {
	status = "okay";
};
```

## 程式碼範例

```c
#include <zephyr/kernel.h>
#include <zephyr/bluetooth/bluetooth.h>
#include <zephyr/bluetooth/gatt.h>
#include <zephyr/logging/log.h>
#include <zephyr/drivers/uart.h>

LOG_MODULE_REGISTER(iot_gateway, LOG_LEVEL_INF);

// 感測器資料結構
struct sensor_data {
	int16_t temperature;
	uint16_t humidity;
	uint32_t timestamp;
};

static const struct device *uart_dev;

// UART 初始化函式
void uart_init(void) {
	uart_dev = DEVICE_DT_GET(DT_CHOSEN(zephyr_console));
	if (!device_is_ready(uart_dev)) {
		LOG_ERR("UART 設備未就緒");
		return;
	}
}

// BLE 回調函式：發現周邊設備
static void scan_callback(const bt_addr_le_t *addr, int rssi, uint8_t adv_type,
			  struct net_buf_simple *ad) {
	LOG_INF("[IoT] 發現設備: RSSI=%d", rssi);

	// 轉發到 Pi (UART)
	char msg[64];
	snprintf(msg, sizeof(msg), "DEVICE_FOUND:rssi=%d\r\n", rssi);
	for (int i = 0; msg[i]; i++) {
		uart_poll_out(uart_dev, msg[i]);
	}
}

// 初始化藍牙掃描
int bt_scan_start(void) {
	struct bt_le_scan_param scan_param = {
		.type = BT_LE_SCAN_TYPE_ACTIVE,
		.interval = BT_GAP_SCAN_FAST_INTERVAL,
		.window = BT_GAP_SCAN_FAST_WINDOW,
	};

	int err = bt_le_scan_start(&scan_param, scan_callback);
	if (err) {
		LOG_ERR("掃描啟動失敗: %d", err);
		return err;
	}
	LOG_INF("BLE 掃描已啟動");
	return 0;
}

// 主程式進入點
int main(void) {
	int err;

	uart_init();
	LOG_INF("=== NanoClaw IoT 閘道器啟動 ===");

	err = bt_enable(NULL);
	if (err) {
		LOG_ERR("藍牙初始化失敗: %d", err);
		return err;
	}

	err = bt_scan_start();
	if (err) {
		return err;
	}

	// 主迴圈：處理事件
	while (1) {
		k_sleep(K_SECONDS(10));
		LOG_INF("閘道器運行中...");
	}

	return 0;
}
```

## 常見問題

**Q: 如何將接收到的感測器資料上傳到雲端？**
A: 在 UART 回調中整合 HTTP/MQTT 客戶端（通過 Pi 處理），或在 nRF54L15 上直接實作 CoAP。

**Q: BLE 掃描會消耗多少電力？**
A: 主動掃描約 15-20mA，可用 `BT_GAP_SCAN_LOW_POWER_INTERVAL` 降低至 5mA。

**Q: 如何連接多個感測器？**
A: 使用 BLE 多重連接（配置 `CONFIG_BT_MAX_CONN`），或實作星型網路拓撲。

## API 快速參考

| 函式 | 用途 |
|------|------|
| `bt_enable(cb)` | 初始化藍牙堆疊 |
| `bt_le_scan_start(param, cb)` | 啟動 BLE 掃描 |
| `bt_le_scan_stop()` | 停止掃描 |
| `uart_poll_out(dev, c)` | UART 字元輸出 |
| `k_sleep(duration)` | 睡眠延遲 |
