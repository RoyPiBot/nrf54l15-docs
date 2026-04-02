---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [peripheral]
difficulty: [intermediate]
keywords: [I2S, audio, DAC, ADC, audio streaming, digital audio]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-02
---

# I2S 音訊介面：DAC/ADC 連接、音訊串流

## 概述

I2S（Inter-IC Sound）是用於 DAC/ADC 連接的串列音訊協議。nRF54L15 整合 I2S 控制器，支援多通道音訊串流、可配置的取樣率（8kHz～48kHz）和位寬（8/16/24/32 位）。適用於麥克風陣列、揚聲器驅動和實時音訊處理。

## Devicetree + Kconfig

**nrf54l15dk.overlay**
```devicetree
&i2s0 {
  status = "okay";
  pinctrl-0 = <&i2s0_default>;
  pinctrl-names = "default";
};

&pinctrl {
  i2s0_default: i2s0_default {
    group1 {
      psels = <NRF_PSEL(I2S_SCK_M, 0, 28)>,
              <NRF_PSEL(I2S_LRCK_M, 0, 29)>,
              <NRF_PSEL(I2S_SDOUT, 0, 30)>,
              <NRF_PSEL(I2S_SDIN, 0, 31)>;
    };
  };
};
```

**prj.conf**
```
CONFIG_I2S=y
CONFIG_I2S_LOG_LEVEL_DBG=y
```

## 程式碼範例

```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/i2s.h>
#include <zephyr/logging/log.h>

LOG_MODULE_REGISTER(i2s_example);

// I2S 裝置和記憶體配置
const struct device *i2s_dev = DEVICE_DT_GET(DT_NODELABEL(i2s0));
static int16_t tx_buffer[512];
static int16_t rx_buffer[512];

// I2S 配置結構
static const struct i2s_config i2s_cfg = {
  .word_size = 16,                    // 16 位元音訊
  .channels = 2,                      // 立體聲（左右通道）
  .format = I2S_FMT_DATA_FORMAT_I2S,
  .options = I2S_OPT_FRAME_CLK_MASTER | I2S_OPT_BIT_CLK_MASTER,
  .frame_clk_freq = 16000,            // 16 kHz 取樣率
  .mem_slab = NULL,                   // 使用外部 buffer
  .timeout = K_FOREVER,
};

void main(void) {
  if (!device_is_ready(i2s_dev)) {
    LOG_ERR("I2S 裝置未就緒");
    return;
  }

  // 配置 I2S
  if (i2s_configure(i2s_dev, I2S_DIR_RX, &i2s_cfg) != 0) {
    LOG_ERR("I2S 接收配置失敗");
    return;
  }

  if (i2s_configure(i2s_dev, I2S_DIR_TX, &i2s_cfg) != 0) {
    LOG_ERR("I2S 發送配置失敗");
    return;
  }

  // 初始化音訊 buffer（發送靜音資料）
  memset(tx_buffer, 0, sizeof(tx_buffer));

  // 啟動 I2S 接收和發送
  if (i2s_trigger(i2s_dev, I2S_DIR_RX, I2S_TRIGGER_START) != 0) {
    LOG_ERR("I2S 接收啟動失敗");
    return;
  }

  if (i2s_trigger(i2s_dev, I2S_DIR_TX, I2S_TRIGGER_START) != 0) {
    LOG_ERR("I2S 發送啟動失敗");
    return;
  }

  LOG_INF("I2S 音訊串流已啟動（16kHz，16位，立體聲）");

  // 循環接收和發送音訊
  while (1) {
    // 讀取接收 buffer
    int ret = i2s_read(i2s_dev, (void *)rx_buffer, sizeof(rx_buffer));
    if (ret != 0) {
      LOG_ERR("I2S 讀取失敗: %d", ret);
    } else {
      LOG_DBG("收到 %d 位元組的音訊資料", ret);
    }

    // 傳送音訊 buffer（可做信號處理）
    ret = i2s_write(i2s_dev, (void *)tx_buffer, sizeof(tx_buffer));
    if (ret != 0) {
      LOG_ERR("I2S 寫入失敗: %d", ret);
    }

    k_msleep(32);  // 32ms 緩衝區長度
  }
}
```

## 常見問題

**Q1: 如何改變取樣率（sample rate）？**
修改 `i2s_cfg.frame_clk_freq`，支援 8/16/32/48 kHz。重新呼叫 `i2s_configure()` 即可應用。

**Q2: 單邊接收或發送（非全雙工）？**
只呼叫需要的方向，如 `i2s_configure(dev, I2S_DIR_RX, ...)` 且只啟動接收。可節省功耗。

**Q3: 如何選擇 clock master（內部時脈 vs 外部）？**
`I2S_OPT_FRAME_CLK_MASTER` 設定 nRF54L15 為主控；移除此選項改用 `I2S_OPT_FRAME_CLK_SLAVE` 使用外部時脈。

## API 快速參考

| 函式 | 功能 |
|------|------|
| `i2s_configure(dev, dir, cfg)` | 配置接收（RX）或發送（TX）方向的參數 |
| `i2s_trigger(dev, dir, trigger)` | 啟動（START）或停止（STOP）I2S 串流 |
| `i2s_read(dev, buf, size)` | 從接收 buffer 讀取音訊資料 |
| `i2s_write(dev, buf, size)` | 向發送 buffer 寫入音訊資料 |
| `i2s_buf_write_requeue(dev, buf)` | 重新放入已使用 buffer（用於 DMA 模式） |
