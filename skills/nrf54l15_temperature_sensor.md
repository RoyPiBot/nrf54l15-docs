---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [peripheral, power]
difficulty: intermediate
keywords: [temperature sensor, thermal monitoring, overheat protection, ADC, calibration]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-02
---

# nRF54L15 片上溫度感測器

## 概述
nRF54L15 內建溫度感測器，可於 -40°C ~ +125°C 範圍內讀取晶片溫度。溫度值透過內部 ADC 取樣，支援校準、低功耗監控及過溫保護。

## Devicetree + Kconfig

### nrf54l15dk_nrf54l15_cpuapp.overlay
```dtsi
/ {
    zephyr,user {
        io-channels = <&adc 0>;
    };
};

&adc {
    status = "okay";
    #address-cells = <1>;
    #size-cells = <0>;

    channel@0 {
        reg = <0>;
        zephyr,gain = "ADC_GAIN_1";
        zephyr,reference = "ADC_REF_INTERNAL";
        zephyr,acquisition-time = <ADC_ACQ_TIME_DEFAULT>;
        zephyr,input-positive = <NRF_SAADC_AIN0>;
    };
};
```

### prj.conf
```
CONFIG_ADC=y
CONFIG_SENSOR=y
CONFIG_HWINFO=y
CONFIG_TEMP_NRF5=y
CONFIG_LOG=y
```

## 程式碼範例

```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/adc.h>
#include <zephyr/sys/util.h>
#include <zephyr/logging/log.h>

LOG_MODULE_REGISTER(temp_sensor);

#define VREF_MV 600
#define SLOPE_MV_PER_C 2.3f
#define CALIBRATION_OFFSET 270

static const struct adc_dt_spec adc_channel = ADC_DT_SPEC_GET_SCALE(
    DT_NODELABEL(adc), 0, 0, VREF_MV, 12);

// 讀取並轉換溫度值
float read_temperature(void) {
    int32_t val_raw;
    int ret;

    if (!adc_is_ready_dt(&adc_channel)) {
        LOG_ERR("ADC 未就緒");
        return -999.0f;
    }

    // 設定序列並讀取
    struct adc_sequence sequence = {
        .buffer = &val_raw,
        .buffer_size = sizeof(val_raw),
        .channels = BIT(adc_channel.channel_id),
        .oversampling = 4,
        .calibrate = true,
    };

    ret = adc_read_dt(&adc_channel, &sequence);
    if (ret < 0) {
        LOG_ERR("ADC 讀取失敗: %d", ret);
        return -999.0f;
    }

    // ADC 轉換為電壓 (mV)
    int32_t val_mv = val_raw;
    adc_raw_to_millivolts_dt(&adc_channel, &val_mv);

    // 電壓轉換為溫度 (°C)
    // 公式: T = (Vin - Offset) / SLOPE
    float temp_c = (val_mv - CALIBRATION_OFFSET) / SLOPE_MV_PER_C;

    return temp_c;
}

// 主程式
int main(void) {
    LOG_INF("nRF54L15 溫度感測器初始化");

    while (1) {
        float temp = read_temperature();

        if (temp < -100.0f) {
            LOG_WRN("溫度感測異常");
        } else {
            LOG_INF("晶片溫度: %.1f°C", temp);

            // 過溫保護：超過 100°C 時發警告
            if (temp > 100.0f) {
                LOG_ERR("過溫警告！溫度: %.1f°C", temp);
                // 可觸發降速或關機邏輯
            }
        }

        k_sleep(K_SECONDS(2));
    }

    return 0;
}
```

## 常見問題

**Q1: 溫度值不準確**
A: 需校準 `CALIBRATION_OFFSET` 值。讀取已知溫度環境下的 ADC 原始值，反推偏移量，或使用廠商提供的校準資料。

**Q2: 過溫保護如何實現**
A: 監控溫度值，超過閾值時設定軟體標誌或觸發硬體保護（如時脈降速、功率限制）。可搭配中斷機制即時響應。

**Q3: 低功耗下如何監控溫度**
A: 配置定期喚醒（RTC 計時器）讀取一次溫度，或使用 PPI 連接 TEMP 事件至 ADC 自動採樣，減少 CPU 喚醒次數。

## API 快速參考

| 函式 | 說明 |
|------|------|
| `adc_read_dt(&adc_channel, &sequence)` | 同步讀取 ADC 數值 |
| `adc_raw_to_millivolts_dt(&channel, &val)` | 原始值轉換為毫伏 |
| `adc_is_ready_dt(&adc_channel)` | 檢查 ADC 是否就緒 |
| `adc_channel_setup_dt(&adc_channel)` | 配置 ADC 通道 |

---
**參考資源**: nRF54L15 PS v0.3 §4.22 (Temperature Sensor), nRF Connect SDK ADC Driver Documentation
