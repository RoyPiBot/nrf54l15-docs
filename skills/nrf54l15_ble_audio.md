---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [wireless, peripheral]
difficulty: intermediate
keywords: [LE Audio, LC3, Auracast, 廣播音訊, 編解碼, 低延遲]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-02
---

# LE Audio 音訊：LC3 編解碼、廣播音訊、Auracast

## 概述
LE Audio 是藍牙 5.2+ 新規範，採用高效率 LC3 編解碼器，提供低延遲、低功耗的音訊傳輸。支援廣播模式 (Auracast 無線擴播) 和單播模式，適用於助聽器、無線耳機、聽力保護等應用。nRF54L15 內建硬體音訊支援，可實現實時音訊處理。

## Devicetree + Kconfig

**prj.conf** (基本配置)
```conf
# LE Audio 核心
CONFIG_BT=y
CONFIG_BT_CTLR_LE_AUDIO_CENTRAL=y
CONFIG_BT_CTLR_LE_AUDIO_PERIPHERAL=y
CONFIG_BT_BAP=y

# LC3 編解碼
CONFIG_BT_BAP_CODEC_LC3=y
CONFIG_LC3=y

# Auracast 廣播
CONFIG_BT_BAP_BROADCAST_SOURCE=y
CONFIG_BT_BAP_BROADCAST_SINK=y

# 音訊介面
CONFIG_AUDIO=y
CONFIG_I2S=y
```

**overlay (.overlay)**
```devicetree
&i2s0 {
  status = "okay";
  pinctrl-0 = <&i2s0_default>;
  pinctrl-names = "default";
};
```

## 程式碼範例

```c
#include <zephyr/kernel.h>
#include <zephyr/bluetooth/bluetooth.h>
#include <zephyr/bluetooth/audio/audio.h>
#include <zephyr/audio/audio_codec_lc3.h>

/* LE Audio 配置結構 */
static struct bt_bap_lc3_preset le_audio_cfg = {
  .codec_cfg = {
    .id = BT_CODEC_LC3_ID,
    .cid = 0x0004, /* LC3 廠商ID */
    .vid = 0x0000,
    .data = BT_CODEC_DATA(BT_CODEC_LC3_FREQ_16KHZ,
                          BT_CODEC_LC3_DURATION_10MS),
    .meta = BT_CODEC_DATA(BT_AUDIO_METADATA_TYPE_STREAM_CONTEXT,
                          BT_AUDIO_CONTEXT_MEDIA),
  },
};

/* Auracast 廣播流初始化 */
static int setup_broadcast_source(void) {
  int ret;

  /* 建立 BAP 廣播源 */
  struct bt_bap_broadcast_source *broadcast_source;

  ret = bt_bap_broadcast_source_create(&le_audio_cfg,
                                       &broadcast_source);
  if (ret) {
    printk("廣播源建立失敗: %d\n", ret);
    return ret;
  }

  /* 啟動廣播 */
  ret = bt_bap_broadcast_source_start(broadcast_source);
  if (ret) {
    printk("廣播啟動失敗: %d\n", ret);
    return ret;
  }

  printk("LE Audio Auracast 廣播已啟動 (LC3 16kHz 10ms)\n");
  return 0;
}

/* 主程式 */
int main(void) {
  int ret;

  printk("nRF54L15 LE Audio 示例\n");

  /* 初始化藍牙 */
  ret = bt_enable(NULL);
  if (ret) {
    printk("藍牙初始化失敗: %d\n", ret);
    return ret;
  }

  printk("藍牙已啟用\n");

  /* 設置廣播音訊 */
  ret = setup_broadcast_source();
  if (ret) {
    return ret;
  }

  /* 主迴圈 */
  while (1) {
    k_sleep(K_SECONDS(1));
  }

  return 0;
}
```

## 常見問題

**Q: LC3 vs SBC — 為什麼要用 LC3？**
A: LC3 在 16kHz 10ms 配置下延遲僅 27.5ms，SBC 無法達到；功耗更低，適合助聽器等電池設備。

**Q: Auracast 和單播 LE Audio 有何區別？**
A: Auracast 是廣播模式，一個源對多個接收器，無建立連接成本；單播需要點對點連接，適合私密場景。

**Q: 如何優化音訊延遲和功耗？**
A: 調整 `BT_CODEC_LC3_DURATION_*`（7.5ms vs 10ms），降低取樣率，使用硬體卸載 I2S 和編解碼。

## API 快速參考

| 函式 | 功能 |
|------|------|
| `bt_bap_broadcast_source_create(cfg, src)` | 建立廣播源 |
| `bt_bap_broadcast_source_start(src)` | 啟動廣播傳輸 |
| `bt_audio_codec_lc3_freq_to_hz(freq)` | 將頻率常數轉為 Hz |
| `bt_bap_unicast_client_setup(client)` | 設置單播用戶端 |
| `bt_bap_stream_start(stream)` | 啟動單播音訊流 |
