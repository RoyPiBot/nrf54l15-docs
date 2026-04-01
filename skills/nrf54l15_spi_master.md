---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [peripheral]
difficulty: [intermediate]
keywords: [SPI, master, clock, chip-select, full-duplex, DMA]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-01
---

# nRF54L15 SPI 主控端：設定時脈、片選、全雙工傳輸

## 概述
SPI 主控端支援可配置的時脈頻率、自動片選 (CS) 管理、全雙工同步收發。本文涵蓋基本配置、DMA 模式、常見陷阱。

## Devicetree + Kconfig

**boards/nrf54l15dk_nrf54l15_cpuapp.overlay**
```dts
&spi30 {
	status = "okay";
	pinctrl-0 = <&spi30_default>;
	pinctrl-names = "default";

	cs-gpios = <&gpio0 10 GPIO_ACTIVE_LOW>;

	slave@0 {
		compatible = "dummy-slave";
		reg = <0>;
		spi-max-frequency = <8000000>;  /* 8 MHz */
	};
};

&pinctrl {
	spi30_default: spi30_default {
		group1 {
			psels = <NRF_PSEL(SPIM_SCK, 0, 6)>,
			        <NRF_PSEL(SPIM_MOSI, 0, 7)>,
			        <NRF_PSEL(SPIM_MISO, 0, 8)>;
		};
	};
};
```

**prj.conf**
```ini
CONFIG_SPI=y
CONFIG_SPI_NRF_SPIM=y
CONFIG_SPI_NRF_SPIM_DMA=y
CONFIG_SPI_STATS=y
```

## 程式碼範例

```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/spi.h>
#include <zephyr/logging/log.h>

LOG_MODULE_REGISTER(spi_master, LOG_LEVEL_INF);

int main(void) {
	// 取得 SPI 設備
	const struct device *spi_dev = DEVICE_DT_GET(DT_NODELABEL(spi30));
	if (!device_is_ready(spi_dev)) {
		LOG_ERR("SPI 設備未就緒");
		return -1;
	}

	// 配置 SPI 參數：頻率、極性、相位
	struct spi_config cfg = {
		.frequency = 8000000,        // 8 MHz 時脈
		.operation = SPI_WORD_SET(8) // 8 位元資料寬度
		           | SPI_TRANSFER_MSB // MSB 優先
		           | SPI_MODE_0,      // 極性=0, 相位=0
		.slave = 0,                   // 從設備編號
		.cs = NULL,                   // 使用 GPIO 片選
	};

	// 準備發送與接收緩衝區
	uint8_t tx_buf[16] = {0x55, 0xAA, 0x12, 0x34};
	uint8_t rx_buf[16] = {0};

	// 設置 SPI 緩衝區結構
	struct spi_buf tx = {.buf = tx_buf, .len = 4};
	struct spi_buf rx = {.buf = rx_buf, .len = 4};
	struct spi_buf_set tx_bufs = {.buffers = &tx, .count = 1};
	struct spi_buf_set rx_bufs = {.buffers = &rx, .count = 1};

	// 全雙工傳輸（同時發送與接收）
	int ret = spi_transceive(spi_dev, &cfg, &tx_bufs, &rx_bufs);
	if (ret < 0) {
		LOG_ERR("SPI 傳輸失敗: %d", ret);
		return -1;
	}

	// 輸出接收結果
	LOG_INF("接收到: 0x%02X 0x%02X 0x%02X 0x%02X",
		rx_buf[0], rx_buf[1], rx_buf[2], rx_buf[3]);

	return 0;
}
```

## 常見問題

**Q: 如何調整 SPI 時脈頻率？**
A: 修改 overlay 中 `spi-max-frequency` 或程式碼 `cfg.frequency`。常用值：1 MHz、4 MHz、8 MHz。超過 10 MHz 需確認硬體支援。

**Q: CS 信號異常或持續拉低？**
A: 檢查 GPIO 腳位編號、`GPIO_ACTIVE_LOW` 設定、外接上拉電阻。不使用 GPIO CS 時在 overlay 移除 `cs-gpios`。

**Q: 為何只收到 0xFF 或 0x00？**
A: 檢查 MISO 接線、模式設定 (SPI_MODE_0/1/2/3)、時脈頻率過高或過低。

## API 快速參考

| 函式 | 說明 |
|------|------|
| `spi_transceive()` | 全雙工同步傳輸（發送+接收） |
| `spi_write()` | 單向發送（MOSI 只） |
| `spi_read()` | 單向接收（MISO 只，發送 0x00） |
| `spi_release()` | 釋放 SPI 匯流排（用於多主控） |

**頻率計算**：nRF54L15 支援 0.5～32 MHz，實際值由硬體分頻器決定，最接近的值自動選擇。
