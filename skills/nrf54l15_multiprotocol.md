---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [wireless, build-system]
difficulty: advanced
keywords: [BLE, Thread, 多協定, 時分多工, 共存, 射頻調度]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-02
---

# 多協定共存：BLE+Thread 同時運行、時分多工

## 概述
nRF54L15 支援同一 SoC 上運行多個無線協議（BLE + Thread）。透過**時分多工**（TDM）機制，系統在兩個協議間動態調度射頻資源，避免信號碰撞，實現低功耗多協議應用。

## Devicetree + Kconfig

**prj.conf** — 啟用多協議棧：
```ini
# BLE 和 Thread 棧配置
CONFIG_BT=y
CONFIG_BT_PERIPHERAL=y
CONFIG_BT_OBSERVER=y
CONFIG_OPENTHREAD_ENABLED=y

# 多協議共存
CONFIG_BT_CTLR_COEX=y
CONFIG_BT_CTLR_COEX_THREAD=y

# 時分多工參數
CONFIG_BT_CTLR_COEX_PRIORITY_MODE=y
CONFIG_BT_CTLR_COEX_TIMEOUT_MS=1000

# Zephyr 通用配置
CONFIG_MAIN_STACK_SIZE=2048
CONFIG_DYNAMIC_MEMORY_SIZE=65536
CONFIG_BT_HCI_VS=y
CONFIG_SERIAL=y
CONFIG_UART_CONSOLE=y
```

**overlay-multiproto.overlay** — 射頻前端配置：
```dts
&radio {
    status = "okay";
};

&hfxo {
    status = "okay";
};

&uart0 {
    current-speed = <115200>;
};
```

## 程式碼範例

**main.c** — BLE + Thread 共存示例：
```c
#include <zephyr/kernel.h>
#include <zephyr/bluetooth/bluetooth.h>
#include <openthread/instance.h>
#include <openthread/thread.h>
#include <openthread/udp.h>

// BLE 廣告資料
static const struct bt_data ad[] = {
    BT_DATA_BYTES(BT_DATA_FLAGS, (BT_LE_AD_GENERAL | BT_LE_AD_NO_BREDR)),
};

// BLE 回調：連線事件
static void on_le_connect(struct bt_conn *conn, uint8_t err) {
    if (err) {
        printk("BLE 連線失敗: %u\n", err);
    } else {
        printk("BLE 已連線\n");
    }
}

static struct bt_conn_cb conn_callbacks = {
    .connected = on_le_connect,
};

// Thread 狀態變更回調
static void on_thread_state_changed(uint32_t flags, void *context) {
    if (flags & OT_CHANGED_THREAD_ROLE) {
        otDeviceRole role = otThreadGetDeviceRole((otInstance *)context);
        const char *role_str[] = {"禁用", "分離", "子節點", "路由器", "主導者"};
        printk("Thread 角色變更: %s\n", role_str[role]);
    }
}

// 初始化 BLE
static int ble_init(void) {
    int err = bt_enable(NULL);
    if (err) {
        printk("BLE 初始化失敗 %d\n", err);
        return err;
    }

    bt_conn_cb_register(&conn_callbacks);

    err = bt_le_adv_start(BT_LE_ADV_CONN_NAME, ad, ARRAY_SIZE(ad), NULL, 0);
    if (err) {
        printk("BLE 廣告啟動失敗 %d\n", err);
    }
    return err;
}

// 初始化 Thread
static int thread_init(void) {
    otInstance *instance = otInstanceInitSingle();
    if (!instance) {
        printk("Thread 實例初始化失敗\n");
        return -EINVAL;
    }

    otSetStateChangedCallback(instance, on_thread_state_changed, instance);
    otThreadSetEnabled(instance, true);
    printk("Thread 已啟用\n");
    return 0;
}

void main(void) {
    printk("=== nRF54L15 多協議應用 ===\n");

    if (ble_init() != 0) {
        printk("警告: BLE 初始化失敗\n");
    }

    if (thread_init() != 0) {
        printk("警告: Thread 初始化失敗\n");
    }

    printk("BLE 和 Thread 共存運行中...\n");

    while (1) {
        k_sleep(K_SECONDS(5));
        printk("系統穩定運行...\n");
    }
}
```

## 常見問題

**Q1: BLE 和 Thread 如何共享射頻資源？**
- nRF54L15 使用**時分多工**（TDM）調度器，根據優先級和時間片輪流分配射頻時間給 BLE 和 Thread。BLE 通常優先級更高（避免連線斷開）。

**Q2: 如何調整協議優先級？**
- 修改 `CONFIG_BT_CTLR_COEX_PRIORITY_MODE` 和 `CONFIG_BT_CTLR_COEX_TIMEOUT_MS` 參數，或呼叫 `otPlatRadioCoexRequestHandle()` 動態調整。

**Q3: 功耗影響多大？**
- 多協議同時運行會增加 15-30% 功耗。可透過調整廣告/掃描間隔和 Thread 輪詢頻率優化。

## API 快速參考

| 函式 | 功能 |
|------|------|
| `bt_enable(cb)` | 初始化 BLE 棧 |
| `bt_le_adv_start(param, ad, ad_len, sd, sd_len)` | 啟動 BLE 廣告 |
| `otInstanceInitSingle()` | 初始化 Thread 實例 |
| `otThreadSetEnabled(instance, enabled)` | 啟用/禁用 Thread |
| `otPlatRadioCoexRequestHandle(gnt_cnt, req_cnt)` | 射頻共存授權控制（高級） |
