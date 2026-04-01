---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [peripheral]
difficulty: [intermediate]
keywords: [ADC, 取樣, 通道設定, 過取樣, 參考電壓, SAADC]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-01
---

# nRF54L15 14-bit ADC 取樣：通道設定、過取樣、參考電壓

## 概述
nRF54L15 內建 SAADC（逐次逼近型 ADC），提供 14-bit 分辨率、8 個獨立輸入通道、可配置的過取樣倍率、多種參考電壓選項（內部 0.6V、VDD/4 等）。支援同步及非同步讀取。

## Devicetree 設定

**boards/nrf54l15dk_nrf54l15_cpuapp.overlay**
```devicetree
&adc {
	status = "okay";
	#address-cells = <1>;
	#size-cells = <0>;

	channel@0 {
		reg = <0>;
		zephyr,gain = "ADC_GAIN_1_4";
		zephyr,reference = "ADC_REF_INTERNAL";
		zephyr,acquisition-time = <ADC_ACQ_TIME_DEFAULT>;
		zephyr,input-positive = <NRF_SAADC_AIN0>;
		zephyr,input-negative = <NRF_SAADC_GND>;
		zephyr,resolution = <14>;
	};

	channel@1 {
		reg = <1>;
		zephyr,gain = "ADC_GAIN_1_4";
		zephyr,reference = "ADC_REF_VDD_1_4";
		zephyr,input-positive = <NRF_SAADC_AIN1>;
		zephyr,input-negative = <NRF_SAADC_GND>;
		zephyr,resolution = <14>;
	};
};
```

## Kconfig 設定

**prj.conf**
```
CONFIG_ADC=y
CONFIG_ADC_NRFX_SAADC=y
CONFIG_ADC_ASYNC=y
CONFIG_ADC_SEQUENCER=y
CONFIG_NRF_SAADC_OVERSAMPLE_256=y
```

## 程式碼範例

```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/adc.h>
#include <zephyr/logging/log.h>

LOG_MODULE_REGISTER(adc_demo, LOG_LEVEL_INF);

#define ADC_DEVICE_NODE	DT_NODELABEL(adc)
static const struct device *adc_dev = DEVICE_DT_GET(ADC_DEVICE_NODE);

// ADC 緩衝區（存放取樣值）
static int16_t m_sample_buffer[2];

int adc_sample_init(void)
{
	if (!device_is_ready(adc_dev)) {
		LOG_ERR("ADC 裝置未就緒");
		return -ENODEV;
	}
	LOG_INF("ADC 初始化成功");
	return 0;
}

// 同步讀取多通道
int adc_read_channels(void)
{
	// 設定序列：讀通道 0 和 1
	const struct adc_sequence sequence = {
		.channels = BIT(0) | BIT(1),
		.buffer = m_sample_buffer,
		.buffer_size = sizeof(m_sample_buffer),
		.resolution = 14,
	};

	int ret = adc_read(adc_dev, &sequence);
	if (ret) {
		LOG_ERR("ADC 讀取失敗: %d", ret);
		return ret;
	}

	// 轉換為 mV（假設 VDD=3.3V）
	int ch0_mv = (m_sample_buffer[0] * 3300) >> 14;
	int ch1_mv = (m_sample_buffer[1] * 3300) >> 14;

	LOG_INF("CH0: %d mV, CH1: %d mV", ch0_mv, ch1_mv);
	return 0;
}

// 非同步讀取（中斷驅動）
static void adc_callback(const struct device *dev,
			 const struct adc_sequence *sequence,
			 uint16_t num_samples, void *user_data)
{
	LOG_INF("非同步讀取完成: CH0=%d, CH1=%d",
		m_sample_buffer[0], m_sample_buffer[1]);
}

int adc_read_async(void)
{
	const struct adc_sequence sequence = {
		.channels = BIT(0) | BIT(1),
		.buffer = m_sample_buffer,
		.buffer_size = sizeof(m_sample_buffer),
		.resolution = 14,
		.callback = adc_callback,
	};

	return adc_read_async(adc_dev, &sequence, NULL);
}

// 主程式
void main(void)
{
	int ret = adc_sample_init();
	if (ret < 0) {
		LOG_ERR("初始化失敗");
		return;
	}

	while (1) {
		adc_read_channels();
		k_sleep(K_SECONDS(1));
	}
}
```

## 常見問題

**Q1: 如何選擇參考電壓？**
- `ADC_REF_INTERNAL`：內部 0.6V，用於低電壓訊號
- `ADC_REF_VDD_1_4`：VDD/4，用於 0～VDD 範圍
- 通常選 VDD_1_4 以最大化動態範圍

**Q2: 過取樣對精度的影響？**
過取樣 2^n 倍可增加 n/2 bit 有效位數（例如過取樣 256 倍 ≈ 增加 4 bit），但增加功耗與採樣時間。

**Q3: 14-bit 轉 mV 的計算方式？**
```c
mV = (adc_value * V_ref_mV) >> 14
```
其中 V_ref_mV 依據參考電壓設定（例如 VDD=3300mV）。

## API 快速參考

| 函式 | 說明 |
|------|------|
| `adc_read()` | 同步讀取，阻塞直到完成 |
| `adc_read_async()` | 非同步讀取，完成後呼叫 callback |
| `adc_channel_setup()` | 設定單一通道參數 |
| `adc_sequence` | 描述讀取序列（通道掩碼、緩衝、解析度） |

**參考文件**：[Zephyr ADC 驅動](https://docs.zephyrproject.org/latest/hardware/peripherals/adc.html)
