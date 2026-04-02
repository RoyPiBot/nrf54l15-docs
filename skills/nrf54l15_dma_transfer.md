---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [peripheral]
difficulty: [intermediate]
keywords: [DMA, EasyDMA, 鏈式傳輸, 直接記憶體存取]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-02
---

# DMA 直接記憶體存取：EasyDMA 設定、鏈式傳輸

## 概述
EasyDMA 是 nRF54L15 內建的低功耗直接記憶體存取引擎，支援無 CPU 介入的高效記憶體傳輸。透過鏈式傳輸可實現複雜的多階段資料流。

## Devicetree + Kconfig

### sample.overlay
```devicetree
/ {
  aliases {
    dma0 = &dma00;
  };
};

&dma00 {
  status = "okay";
};
```

### prj.conf
```
CONFIG_DMA=y
CONFIG_NRF_DPPIC=y
CONFIG_DMA_NRF_DPPIC=y
CONFIG_DPPI_CHANNELS=16
```

## 程式碼範例

```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/dma.h>
#include <zephyr/logging/log.h>

LOG_MODULE_REGISTER(dma_demo);

// 測試用緩衝區（必須 SRAM）
static uint32_t src_buf[64] = {0};
static uint32_t dst_buf[64] = {0};

static void dma_callback(const struct device *dev, void *user_data,
                        uint32_t channel, int status) {
  if (status == 0) {
    LOG_INF("DMA 傳輸完成（通道 %u）", channel);
  } else {
    LOG_ERR("DMA 傳輸錯誤: %d", status);
  }
}

int main(void) {
  const struct device *dma_dev = DEVICE_DT_GET_ONE(nordic_dma);

  if (!device_is_ready(dma_dev)) {
    LOG_ERR("DMA 裝置未就緒");
    return -ENODEV;
  }

  // 初始化來源資料
  for (int i = 0; i < 64; i++) {
    src_buf[i] = 0xDEADBEEF + i;
  }

  // 設定 DMA 主配置
  struct dma_config dma_cfg = {
    .channel_direction = MEMORY_TO_MEMORY,
    .source_data_size = 4,     // 4 bytes (32-bit)
    .dest_data_size = 4,
    .source_burst_length = 16, // 每次傳 16*4=64 bytes
    .dest_burst_length = 16,
    .block_count = 1,
    .dma_callback = dma_callback,
  };

  // 設定傳輸區塊
  struct dma_block_config blk_cfg = {
    .source_address = (uint32_t)src_buf,
    .dest_address = (uint32_t)dst_buf,
    .block_size = 256,         // 64*4=256 bytes
  };

  dma_cfg.head_block = &blk_cfg;

  // 配置 DMA 通道 0
  int ret = dma_configure(dma_dev, 0, &dma_cfg);
  if (ret < 0) {
    LOG_ERR("DMA 配置失敗: %d", ret);
    return ret;
  }

  // 啟動傳輸
  ret = dma_start(dma_dev, 0);
  if (ret < 0) {
    LOG_ERR("DMA 啟動失敗: %d", ret);
    return ret;
  }

  LOG_INF("DMA 傳輸已啟動");
  k_sleep(K_MSEC(100));
  return 0;
}
```

## 常見問題

**Q: EasyDMA 最大傳輸大小？**
A: 單次傳輸最大 65535 bytes，透過鏈式配置（next_block 指針）可突破限制。

**Q: 如何實現鏈式傳輸？**
A: 在 dma_block_config 中設定 next_block，指向下一個區塊結構，DMA 自動銜接。

**Q: 傳輸期間 CPU 消耗？**
A: DMA 獨立運作，CPU 可睡眠，透過中斷喚醒，極低功耗。

## API 快速參考

```c
// 配置 DMA 傳輸
int dma_configure(const struct device *dev, uint32_t channel,
                  struct dma_config *config);

// 啟動傳輸
int dma_start(const struct device *dev, uint32_t channel);

// 停止傳輸
int dma_stop(const struct device *dev, uint32_t channel);

// 查詢狀態
int dma_get_status(const struct device *dev, uint32_t channel,
                   struct dma_status *stat);

// 重新啟動（保持配置）
int dma_reload(const struct device *dev, uint32_t channel,
               uint32_t src, uint32_t dst, size_t size);
```

---
*nRF54L15 EasyDMA 系列文檔 | 鏈式傳輸技術*
