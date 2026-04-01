---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [wireless]
difficulty: [advanced]
keywords: [BLE 6.0, Channel Sounding, 精確測距, 室內定位, RTT, AoA, 室內導航]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-02
---

# BLE 6.0 通道探測：精確測距與室內定位

## 概述

BLE 6.0 Channel Sounding 透過測量訊號多路徑特性計算精確距離（±1-2cm），結合 AoA/AoD 實現室內定位與資產追蹤。利用通道脈衝響應（CIR）分析發射器與接收器間的精確距離，應用於防丟器、精確尋路、無線定位等。

## Devicetree + Kconfig

**prj.conf**
```
CONFIG_BT=y
CONFIG_BT_BROADCASTER=y
CONFIG_BT_OBSERVER=y
CONFIG_BT_EXT_ADV=y
CONFIG_BT_CTLR_CHAN_SOUNDING=y
CONFIG_BT_CTLR_CS_RNG=y
CONFIG_BT_RX_USER_HEAP_SIZE=2048
```

**nrf54l15dk_nrf54l15_cpuapp.overlay**
```devicetree
/ {
    chosen {
        nordic,cs-antenna = &antenna0;
    };

    antennas {
        antenna0: antenna_0 {
            sw-sel-gpios = <&gpio0 5 GPIO_ACTIVE_HIGH>;
        };
    };
};
```

## 程式碼範例

```c
#include <zephyr/kernel.h>
#include <zephyr/bluetooth/bluetooth.h>
#include <zephyr/sys/printk.h>
#include <zephyr/logging/log.h>

LOG_MODULE_REGISTER(cs_rng, LOG_LEVEL_INF);

// 全域變數
static struct bt_conn *conn_device;
static uint8_t peer_addr[BT_ADDR_SIZE];

// 通道探測完成回調
static void cs_rng_done(struct bt_conn *conn,
                        struct bt_le_cs_rng_result *result)
{
    if (!result) {
        LOG_WRN("通道探測失敗或被中止");
        return;
    }

    // 計算距離 (cm) 與 RSSI
    int16_t distance_cm = result->range_cm;
    int16_t rssi = result->rssi;

    LOG_INF("通道探測完成");
    LOG_INF("  距離: %d cm (±5cm)", distance_cm);
    LOG_INF("  RSSI: %d dBm", rssi);
    LOG_INF("  主要路徑延遲: %d ns", result->pd_m);
}

// 啟動通道探測
int start_channel_sounding(void)
{
    if (!conn_device) {
        LOG_ERR("無可用連線");
        return -1;
    }

    struct bt_le_cs_rng_params params = {
        .phy = BT_LE_CS_PHY_2M,
        .rng_mode = BT_LE_CS_RNG_MODE_INITIATOR,
        .tx_power = 0,  // dBm
        .max_ranging_delay_ms = 500,
    };

    int err = bt_le_cs_start(conn_device, &params, cs_rng_done);
    if (err) {
        LOG_ERR("啟動通道探測失敗: %d", err);
        return err;
    }

    LOG_INF("通道探測已啟動");
    return 0;
}

// 連線回調
static void connected(struct bt_conn *conn, uint8_t err)
{
    if (err) {
        LOG_ERR("連線失敗: %d", err);
        return;
    }

    conn_device = bt_conn_ref(conn);
    LOG_INF("已連線");
    k_sleep(K_MSEC(100));
    start_channel_sounding();
}

static void disconnected(struct bt_conn *conn, uint8_t reason)
{
    if (conn_device) {
        bt_conn_unref(conn_device);
        conn_device = NULL;
    }
    LOG_INF("已斷線");
}

static struct bt_conn_cb conn_callbacks = {
    .connected = connected,
    .disconnected = disconnected,
};

// 主程式
int main(void)
{
    int err = bt_enable(NULL);
    if (err) {
        LOG_ERR("藍牙啟用失敗: %d", err);
        return err;
    }

    LOG_INF("藍牙已啟用");
    bt_conn_cb_register(&conn_callbacks);

    // 掃描與連線邏輯（此處簡化）
    LOG_INF("系統就緒，等待連線...");

    while (1) {
        k_sleep(K_SECONDS(1));
    }

    return 0;
}
```

## 常見問題

**Q: 通道探測對頻寬與 PHY 有什麼要求？**
A: 需支援 2M PHY，通常使用 LE Coded PHY 以增加範圍；更寬頻寬（如 LE 160MHz）精度更高。

**Q: 如何在實際室內定位中使用測距結果？**
A: 收集至少 3 個信標的距離，使用三邊測量或加權三角測量計算座標；加入 IMU/ZUPT 可提升動態定位精度。

**Q: 為何有時測距結果飄移？**
A: 多路徑干擾、環境變化、天線角度影響；建議多次取樣加濾波（卡爾曼或移動平均）。

## API 快速參考

| 函式 | 說明 |
|------|------|
| `bt_le_cs_start()` | 發起通道探測測距 |
| `bt_le_cs_abort()` | 中止進行中的探測 |
| `bt_le_cs_rng_result` | 測距結果（距離、RSSI、延遲） |
| `bt_le_cs_rng_params` | 探測參數（PHY、功率、模式） |
| `bt_le_cs_phy_e` | 物理層枚舉（1M, 2M, Coded） |
