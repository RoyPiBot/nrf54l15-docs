---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [peripheral]
difficulty: [intermediate]
keywords: [PDM, microphone, audio, FFT, I2S, PCM, digital]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-02
---

# PDM 數位麥克風：音訊擷取、濾波、FFT 分析

## 概述

PDM（脈衝密度調變）是一種高效的數位音訊擷取技術，內建 Delta-Sigma 調變。nRF54L15 PDM 控制器透過 I2S 介面驅動 MEMS 麥克風，提供即時音訊捕捉。本指南涵蓋基本初始化、低通濾波及頻域分析（FFT）。

## Devicetree + Kconfig

### nrf54l15dk.overlay
```dts
&i2s0 {
    status = "okay";
    pinctrl-0 = <&i2s0_default>;
    pinctrl-names = "default";
};

&pinctrl {
    i2s0_default: i2s0_default {
        group1 {
            psels = <NRF_PSEL(I2S_SCK_M, 0, 15)>,
                    <NRF_PSEL(I2S_SDOUT, 0, 16)>;
        };
    };
};

&pwm0 {
    status = "okay";
    pinctrl-0 = <&pwm0_default>;
    pinctrl-names = "default";
};

&pwm0_default {
    group1 {
        psels = <NRF_PSEL(PWM_OUT0, 0, 13)>;
    };
};
```

### prj.conf
```
CONFIG_I2S=y
CONFIG_PDM=y
CONFIG_DMA=y
CONFIG_HEAP_MEM_POOL_SIZE=32768
CONFIG_LOG=y
```

## 程式碼範例

```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/i2s.h>
#include <zephyr/logging/log.h>
#include <string.h>
#include <math.h>

LOG_MODULE_REGISTER(pdm_mic);

#define SAMPLE_RATE 16000
#define FRAME_SIZE  256
#define BUF_BLOCKS  2

const struct device *i2s_dev;
int32_t audio_buf[FRAME_SIZE];

// 簡易低通濾波（一階 IIR）
int32_t iir_filter(int32_t sample, int32_t *last) {
    int32_t filtered = (sample >> 1) + (*last >> 1);
    *last = sample;
    return filtered;
}

// 簡易功率估算（代替完整 FFT）
uint32_t estimate_power(int32_t *samples, int count) {
    uint64_t sum = 0;
    for (int i = 0; i < count; i++) {
        int32_t s = samples[i];
        sum += (uint64_t)s * s;
    }
    return (uint32_t)(sum / count);
}

void pdm_init(void) {
    i2s_dev = DEVICE_DT_GET(DT_NODELABEL(i2s0));
    if (!device_is_ready(i2s_dev)) {
        LOG_ERR("I2S 裝置未就緒");
        return;
    }

    struct i2s_config cfg = {
        .frame_clk_freq = SAMPLE_RATE,
        .block_size = FRAME_SIZE * 4,
        .channels = 1,
        .format = I2S_FMT_DATA_FORMAT_I2S |
                  I2S_FMT_FRAME_MASTER |
                  I2S_FMT_BIT_CLK_MASTER,
        .options = I2S_OPT_FRAME_CLK_MASTER |
                   I2S_OPT_BIT_CLK_MASTER,
        .mem_slab = NULL,
        .timeout = K_FOREVER,
    };

    i2s_configure(i2s_dev, I2S_DIR_RX, &cfg);
    i2s_trigger(i2s_dev, I2S_DIR_RX, I2S_TRIGGER_START);
    LOG_INF("PDM 初始化完成，採樣率 %d Hz", SAMPLE_RATE);
}

void pdm_capture(void) {
    size_t bytes = i2s_read(i2s_dev, audio_buf, FRAME_SIZE * 4);
    if (bytes != FRAME_SIZE * 4) {
        LOG_WRN("讀取不完整：%u/%u", bytes, FRAME_SIZE * 4);
        return;
    }

    // 應用低通濾波
    int32_t last = 0;
    for (int i = 0; i < FRAME_SIZE; i++) {
        audio_buf[i] = iir_filter(audio_buf[i], &last);
    }

    // 估算功率（頻域指示）
    uint32_t power = estimate_power(audio_buf, FRAME_SIZE);
    LOG_INF("音訊功率：%u", power);
}

int main(void) {
    pdm_init();
    while (1) {
        pdm_capture();
        k_sleep(K_MSEC(100));
    }
    return 0;
}
```

## 常見問題

**Q1: PDM 內部如何運作？**
A: PDM 使用高速時鐘（1.6 MHz）和 Delta-Sigma 調變編碼音訊。內建濾波器將高頻 PDM 信號轉換成低採樣率 PCM（通常 16-48 kHz），每幀 16-32 位元。

**Q2: 如何改進音訊品質？**
A: (1) 增加 DMA 緩衝區減少遺漏；(2) 應用多階 IIR 或 FIR 濾波去除噪聲；(3) 若需高精度 FFT，移植 CMSIS-DSP 或 Kiss FFT。

**Q3: 能同時進行錄製和播放嗎？**
A: 可以。I2S 同時支援 RX 和 TX。分別配置 `I2S_DIR_RX` 和 `I2S_DIR_TX`，確保足夠的堆疊和 DMA 通道。

## API 快速參考

| 函式 | 說明 |
|------|------|
| `i2s_configure(dev, dir, cfg)` | 配置 I2S/PDM（採樣率、格式、通道） |
| `i2s_read(dev, buf, size)` | 阻塞讀取音訊資料到緩衝區 |
| `i2s_write(dev, buf, size)` | 寫入播放資料 |
| `i2s_trigger(dev, dir, cmd)` | 開始/停止 I2S 運作（START/STOP/DROP） |
| `device_is_ready(dev)` | 檢查裝置初始化狀態 |
