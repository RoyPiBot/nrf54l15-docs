---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: wireless
difficulty: intermediate
keywords: [BLE, GATT, Server, Client, Characteristic, Connection, Service, Read, Write, Notify, nRF54L15]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-03-31
---

# BLE 連線：建立 GATT Server/Client、特徵值讀寫

## 1. 概述

### 1.1 功能描述

BLE（Bluetooth Low Energy）連線是嵌入式無線通訊中最核心的功能之一。在 BLE 協定中，裝置間的資料交換主要透過 **GATT（Generic Attribute Profile）** 機制完成。GATT 定義了兩個角色：

- **GATT Server**：持有資料的一方，將資料組織為 Service → Characteristic 的階層結構，供 Client 讀寫。
- **GATT Client**：發起連線並存取 Server 上資料的一方，可執行讀取（Read）、寫入（Write）、訂閱通知（Notify/Indicate）等操作。

nRF54L15 內建的 BLE 5.4 無線電支援高效能的 GATT 通訊，搭配 Zephyr 的 BLE Host Stack，可以快速建立雙向資料通道。

### 1.2 為什麼重要

- **感測器資料傳輸**：溫溼度、加速度等感測器資料透過 GATT Notify 即時推送至手機或閘道器
- **遠端控制**：手機 App 透過 GATT Write 控制嵌入式裝置的 LED、馬達、繼電器
- **韌體更新（DFU）**：Nordic 的 SMP（Simple Management Protocol）也是建立在 GATT 之上
- **雙向互動**：Request/Response 模式讓裝置間可以建立類似 API 的通訊介面

### 1.3 適用場景

| 場景 | GATT 角色 | 說明 |
|------|-----------|------|
| 感測器節點 | Server（Peripheral） | 暴露感測器資料供手機讀取或訂閱 |
| 手機 App 控制 | Server（Peripheral） | 接收手機的寫入指令執行動作 |
| 閘道器 | Client（Central） | 連線多個 Peripheral 蒐集資料 |
| 裝置對裝置 | 兩端皆可 | 雙向資料交換 |

---

## 2. 硬體概念

### 2.1 nRF54L15 BLE 無線電架構

```
┌─────────────────────────────────────────────────┐
│                  nRF54L15 SoC                   │
│                                                 │
│  ┌──────────┐    ┌──────────────────────────┐   │
│  │  CPU     │    │   BLE Radio (2.4 GHz)    │   │
│  │  Arm     │◄──►│                          │   │
│  │  Cortex  │    │  ┌────────────────────┐  │   │
│  │  M33     │    │  │  Link Layer (LL)   │  │   │
│  └──────────┘    │  │  - 連線管理        │  │   │
│       │          │  │  - 頻道跳頻        │  │   │
│       ▼          │  │  - 封包收發        │  │   │
│  ┌──────────┐    │  └────────────────────┘  │   │
│  │  BLE     │    │                          │   │
│  │  Host    │    │  ┌────────────────────┐  │   │
│  │  Stack   │    │  │  PHY Layer         │  │   │
│  │ (Zephyr) │    │  │  - 1M / 2M / Coded│  │   │
│  │          │    │  └────────────────────┘  │   │
│  │ ┌──────┐ │    └──────────────────────────┘   │
│  │ │ GATT │ │                                   │
│  │ │ ATT  │ │    ┌──────────────────────────┐   │
│  │ │ L2CAP│ │    │       Antenna            │   │
│  │ │ GAP  │ │    └──────────────────────────┘   │
│  │ └──────┘ │                                   │
│  └──────────┘                                   │
└─────────────────────────────────────────────────┘
```

### 2.2 GATT 資料模型

GATT 的資料以階層式結構組織：

```
Profile
 └── Service (由 UUID 識別)
      ├── Characteristic 1
      │    ├── Value（實際資料）
      │    ├── Properties（Read/Write/Notify/Indicate）
      │    └── Descriptor（如 CCCD: Client Characteristic Configuration Descriptor）
      └── Characteristic 2
           ├── Value
           ├── Properties
           └── Descriptor
```

- **Service UUID**：16-bit（SIG 定義）或 128-bit（自定義）
- **Characteristic UUID**：同上
- **ATT Handle**：每個 Attribute 在 Server 中的唯一數字索引，Client 透過 Handle 定位資料
- **CCCD**：Client 用來啟用/停用 Notify 或 Indicate 的描述符

### 2.3 連線參數

| 參數 | 範圍 | 說明 |
|------|------|------|
| Connection Interval | 7.5ms ~ 4s | 兩次連線事件的間隔，越短越即時但越耗電 |
| Slave Latency | 0 ~ 499 | Peripheral 可跳過的連線事件數 |
| Supervision Timeout | 100ms ~ 32s | 超過此時間無通訊則判定斷線 |
| MTU | 23 ~ 247 | 單次 ATT 封包最大承載資料長度 |
| Data Length | 27 ~ 251 | Link Layer 層的封包長度 |

---

## 3. 軟體架構

### 3.1 Zephyr BLE Host Stack 架構

Zephyr 的 BLE 軟體堆疊分層如下：

```
┌────────────────────────────────────┐
│         應用層 (Application)       │
│  - bt_gatt_service_register()      │
│  - bt_gatt_read() / bt_gatt_write()│
│  - bt_gatt_notify()                │
├────────────────────────────────────┤
│         GATT Layer                 │
│  - Service / Characteristic 管理   │
│  - ATT Request/Response 處理       │
├────────────────────────────────────┤
│         ATT (Attribute Protocol)   │
│  - Read/Write Request 編碼解碼     │
│  - Handle-Value Notification       │
├────────────────────────────────────┤
│         L2CAP                      │
│  - 邏輯通道多工                    │
│  - MTU 協商                        │
├────────────────────────────────────┤
│         HCI (Host-Controller I/F)  │
├────────────────────────────────────┤
│         Link Layer Controller      │
│  - nRF54L15 硬體無線電驅動         │
└────────────────────────────────────┘
```

### 3.2 關鍵 Zephyr API

**連線管理：**
- `bt_enable()` — 初始化 BLE Stack
- `bt_le_adv_start()` — 開始廣播（Peripheral 端）
- `bt_conn_le_create()` — 發起連線（Central 端）
- `bt_conn_cb_register()` — 註冊連線/斷線回呼

**GATT Server 端：**
- `BT_GATT_SERVICE_DEFINE()` — 巨集宣告 GATT Service
- `BT_GATT_CHARACTERISTIC()` — 巨集宣告 Characteristic
- `bt_gatt_notify()` — 主動推送通知

**GATT Client 端：**
- `bt_gatt_discover()` — 探索遠端 Service/Characteristic
- `bt_gatt_read()` — 讀取遠端 Characteristic
- `bt_gatt_write()` — 寫入遠端 Characteristic
- `bt_gatt_subscribe()` — 訂閱通知

---

## 4. Devicetree 設定

nRF54L15 的 BLE 功能主要由軟體驅動，Devicetree 通常不需要額外設定 BLE 相關節點。但若使用外部天線或特殊 GPIO 控制 RF Switch，則需要自訂 overlay。

### 4.1 基本 Overlay（`boards/nrf54l15dk_nrf54l15_cpuapp.overlay`）

```dts
/* BLE 連線範例的 Devicetree Overlay
 * 此範例啟用 UART console 用於除錯輸出
 * 以及 GPIO LED 用於連線狀態指示
 */

/ {
    aliases {
        /* 連線狀態指示 LED */
        led0 = &led0;
        led1 = &led1;
    };

    leds {
        compatible = "gpio-leds";
        led0: led_0 {
            /* DK 板上 LED1 — 用於顯示廣播中 */
            gpios = <&gpio0 4 GPIO_ACTIVE_HIGH>;
            label = "BLE Advertising LED";
        };
        led1: led_1 {
            /* DK 板上 LED2 — 用於顯示已連線 */
            gpios = <&gpio0 5 GPIO_ACTIVE_HIGH>;
            label = "BLE Connected LED";
        };
    };
};

/* UART0 用於 console 除錯訊息 */
&uart0 {
    status = "okay";
    current-speed = <115200>;
};
```

### 4.2 屬性說明

| 屬性 | 說明 |
|------|------|
| `gpios` | GPIO 控制器引用、腳位編號、旗標（高/低電位有效） |
| `compatible` | 驅動程式匹配字串，`"gpio-leds"` 為 Zephyr 內建 LED 驅動 |
| `status` | `"okay"` 啟用該節點，`"disabled"` 停用 |
| `current-speed` | UART 鮑率，除錯用 115200 即可 |

---

## 5. Kconfig 設定

### 5.1 GATT Server 端 `prj.conf`

```kconfig
# ============================
# BLE 基礎設定
# ============================
# 啟用 Bluetooth 支援
CONFIG_BT=y
# 啟用 BLE Peripheral 角色（可被連線）
CONFIG_BT_PERIPHERAL=y
# 設定裝置名稱（廣播時顯示）
CONFIG_BT_DEVICE_NAME="nRF54L15 GATT Server"
# 裝置名稱最大長度
CONFIG_BT_DEVICE_NAME_MAX=30

# ============================
# GATT 設定
# ============================
# 啟用 GATT 動態資料庫（允許執行時期新增 Service）
CONFIG_BT_GATT_DYNAMIC_DB=y
# ATT 最大 MTU（增大可提升吞吐量）
CONFIG_BT_L2CAP_TX_MTU=247
# 接收端 MTU
CONFIG_BT_BUF_ACL_RX_SIZE=251
# 傳送端 MTU
CONFIG_BT_BUF_ACL_TX_SIZE=251

# ============================
# 連線設定
# ============================
# 最大同時連線數
CONFIG_BT_MAX_CONN=3
# 最大已配對裝置數
CONFIG_BT_MAX_PAIRED=3

# ============================
# 安全性設定
# ============================
# 啟用 SMP（Security Manager Protocol）
CONFIG_BT_SMP=y
# 配對支援
CONFIG_BT_BONDABLE=y
# 支援 SC (Secure Connections)
CONFIG_BT_SMP_SC_ONLY=n

# ============================
# 系統設定
# ============================
# 主堆疊大小
CONFIG_MAIN_STACK_SIZE=2048
# 系統工作佇列堆疊大小
CONFIG_SYSTEM_WORKQUEUE_STACK_SIZE=2048
# 啟用 Log 系統
CONFIG_LOG=y
CONFIG_BT_LOG_LEVEL_INF=y

# ============================
# GPIO（LED 狀態指示）
# ============================
CONFIG_GPIO=y
```

### 5.2 GATT Client 端額外設定

```kconfig
# 在 Server 設定基礎上新增：

# 啟用 Central 角色（可主動連線）
CONFIG_BT_CENTRAL=y

# 啟用 GATT Client 功能
CONFIG_BT_GATT_CLIENT=y

# 啟用掃描功能
CONFIG_BT_SCAN=y

# GATT Discovery 所需的最大 Attribute 數量
CONFIG_BT_GATT_DM=y
```

### 5.3 各設定項說明

| Kconfig 選項 | 預設值 | 說明 |
|-------------|--------|------|
| `CONFIG_BT` | n | 啟用 Bluetooth 子系統 |
| `CONFIG_BT_PERIPHERAL` | n | 啟用 Peripheral（被連線）角色 |
| `CONFIG_BT_CENTRAL` | n | 啟用 Central（主動連線）角色 |
| `CONFIG_BT_GATT_CLIENT` | n | 啟用 GATT Client 操作（discover/read/write） |
| `CONFIG_BT_GATT_DYNAMIC_DB` | n | 允許執行時動態註冊 Service |
| `CONFIG_BT_L2CAP_TX_MTU` | 23 | L2CAP 傳送 MTU，加大可減少分段 |
| `CONFIG_BT_MAX_CONN` | 1 | 同時連線數上限 |
| `CONFIG_BT_SMP` | n | 啟用配對與加密 |
| `CONFIG_BT_BONDABLE` | n | 啟用後配對資訊可持久保存 |

---

## 6. 完整程式碼範例

### 6.1 GATT Server（Peripheral 端）

以下範例實作一個自定義 Service，包含兩個 Characteristic：
- **Sensor Data（Read + Notify）**：模擬感測器數值，支援讀取與通知
- **Control Point（Write）**：接收 Client 的控制指令

```c
/**
 * @file main.c
 * @brief nRF54L15 BLE GATT Server 完整範例
 *
 * 自定義 Service 包含：
 * - Sensor Data Characteristic（可讀取、可通知）
 * - Control Point Characteristic（可寫入）
 *
 * 目標板：nrf54l15dk/nrf54l15/cpuapp
 */

#include <zephyr/kernel.h>
#include <zephyr/sys/printk.h>
#include <zephyr/sys/byteorder.h>
#include <zephyr/logging/log.h>

#include <zephyr/bluetooth/bluetooth.h>
#include <zephyr/bluetooth/hci.h>
#include <zephyr/bluetooth/conn.h>
#include <zephyr/bluetooth/uuid.h>
#include <zephyr/bluetooth/gatt.h>

#include <zephyr/drivers/gpio.h>

/* 註冊 Log 模組 */
LOG_MODULE_REGISTER(ble_gatt_server, LOG_LEVEL_INF);

/* ============================
 * 自定義 Service UUID 定義
 * ============================ */

/* 自定義 Service UUID: 12345678-1234-5678-1234-56789abcdef0 */
#define BT_UUID_CUSTOM_SERVICE_VAL \
    BT_UUID_128_ENCODE(0x12345678, 0x1234, 0x5678, 0x1234, 0x56789abcdef0)
#define BT_UUID_CUSTOM_SERVICE BT_UUID_DECLARE_128(BT_UUID_CUSTOM_SERVICE_VAL)

/* Sensor Data Characteristic UUID: 12345678-1234-5678-1234-56789abcdef1 */
#define BT_UUID_SENSOR_DATA_VAL \
    BT_UUID_128_ENCODE(0x12345678, 0x1234, 0x5678, 0x1234, 0x56789abcdef1)
#define BT_UUID_SENSOR_DATA BT_UUID_DECLARE_128(BT_UUID_SENSOR_DATA_VAL)

/* Control Point Characteristic UUID: 12345678-1234-5678-1234-56789abcdef2 */
#define BT_UUID_CONTROL_POINT_VAL \
    BT_UUID_128_ENCODE(0x12345678, 0x1234, 0x5678, 0x1234, 0x56789abcdef2)
#define BT_UUID_CONTROL_POINT BT_UUID_DECLARE_128(BT_UUID_CONTROL_POINT_VAL)

/* ============================
 * LED 狀態指示定義
 * ============================ */

/* DK 板 LED 節點 */
static const struct gpio_dt_spec led_adv = GPIO_DT_SPEC_GET_OR(DT_ALIAS(led0), gpios, {0});
static const struct gpio_dt_spec led_conn = GPIO_DT_SPEC_GET_OR(DT_ALIAS(led1), gpios, {0});

/* ============================
 * 全域變數
 * ============================ */

/* 目前的 BLE 連線參考 */
static struct bt_conn *current_conn;

/* 模擬感測器數值（2 bytes: 溫度 * 100） */
static int16_t sensor_value = 2500; /* 25.00°C */

/* 通知啟用旗標 */
static bool notify_enabled;

/* ============================
 * GATT 回呼函式
 * ============================ */

/**
 * @brief 讀取 Sensor Data Characteristic 的回呼
 *
 * 當 Client 發送 Read Request 時，此函式被呼叫。
 * 回傳目前的感測器數值。
 */
static ssize_t read_sensor_data(struct bt_conn *conn,
                                const struct bt_gatt_attr *attr,
                                void *buf, uint16_t len, uint16_t offset)
{
    /* 取得 Characteristic 的數值指標 */
    const int16_t *value = attr->user_data;

    LOG_INF("Client 讀取感測器數值: %d (0x%04x)", *value, *value);

    /* 使用 bt_gatt_attr_read 安全地複製資料到回應緩衝區 */
    return bt_gatt_attr_read(conn, attr, buf, len, offset,
                             value, sizeof(*value));
}

/**
 * @brief 寫入 Control Point Characteristic 的回呼
 *
 * 當 Client 發送 Write Request 時，此函式被呼叫。
 * 解析收到的控制指令並執行對應動作。
 */
static ssize_t write_control_point(struct bt_conn *conn,
                                   const struct bt_gatt_attr *attr,
                                   const void *buf, uint16_t len,
                                   uint16_t offset, uint8_t flags)
{
    const uint8_t *data = buf;

    /* 檢查資料長度 */
    if (len < 1) {
        LOG_WRN("寫入資料長度不足");
        return BT_GATT_ERR(BT_ATT_ERR_INVALID_ATTRIBUTE_LEN);
    }

    /* 不支援偏移寫入 */
    if (offset != 0) {
        LOG_WRN("不支援偏移寫入");
        return BT_GATT_ERR(BT_ATT_ERR_INVALID_OFFSET);
    }

    /* 解析控制指令 */
    uint8_t command = data[0];
    LOG_INF("收到控制指令: 0x%02x", command);

    switch (command) {
    case 0x01:
        LOG_INF("指令: 重置感測器數值");
        sensor_value = 2500;
        break;
    case 0x02:
        LOG_INF("指令: 增加感測器數值");
        sensor_value += 100;
        break;
    case 0x03:
        LOG_INF("指令: 減少感測器數值");
        sensor_value -= 100;
        break;
    default:
        LOG_WRN("未知指令: 0x%02x", command);
        return BT_GATT_ERR(BT_ATT_ERR_NOT_SUPPORTED);
    }

    return len;
}

/**
 * @brief CCCD（Client Characteristic Configuration Descriptor）變更回呼
 *
 * 當 Client 啟用或停用 Notification 時，此函式被呼叫。
 */
static void sensor_ccc_cfg_changed(const struct bt_gatt_attr *attr,
                                   uint16_t value)
{
    notify_enabled = (value == BT_GATT_CCC_NOTIFY);
    LOG_INF("Sensor 通知 %s", notify_enabled ? "已啟用" : "已停用");
}

/* ============================
 * GATT Service 定義
 * ============================ */

/*
 * 使用 BT_GATT_SERVICE_DEFINE 巨集靜態定義 GATT Service。
 * 此巨集會在編譯時期將 Service 註冊到 GATT 資料庫。
 *
 * Service 結構：
 *   Custom Service (UUID: 12345678-...def0)
 *     ├── Sensor Data (UUID: ...def1)  [Read, Notify]
 *     │    └── CCCD (自動管理)
 *     └── Control Point (UUID: ...def2) [Write]
 */
BT_GATT_SERVICE_DEFINE(custom_svc,
    /* 主要 Service 宣告 */
    BT_GATT_PRIMARY_SERVICE(BT_UUID_CUSTOM_SERVICE),

    /* Sensor Data Characteristic */
    BT_GATT_CHARACTERISTIC(BT_UUID_SENSOR_DATA,
                           BT_GATT_CHRC_READ | BT_GATT_CHRC_NOTIFY,
                           BT_GATT_PERM_READ,
                           read_sensor_data, NULL, &sensor_value),

    /* Sensor Data 的 CCCD（用於啟用/停用 Notify） */
    BT_GATT_CCC(sensor_ccc_cfg_changed,
                BT_GATT_PERM_READ | BT_GATT_PERM_WRITE),

    /* Control Point Characteristic */
    BT_GATT_CHARACTERISTIC(BT_UUID_CONTROL_POINT,
                           BT_GATT_CHRC_WRITE,
                           BT_GATT_PERM_WRITE,
                           NULL, write_control_point, NULL),
);

/* ============================
 * 廣播設定
 * ============================ */

/* 廣播資料：包含旗標和 Service UUID */
static const struct bt_data ad[] = {
    /* BLE 旗標：General Discoverable + 不支援 BR/EDR */
    BT_DATA_BYTES(BT_DATA_FLAGS,
                  (BT_LE_AD_GENERAL | BT_LE_AD_NO_BREDR)),
    /* 廣播自定義 Service UUID，讓 Client 可透過 UUID 過濾掃描結果 */
    BT_DATA_BYTES(BT_DATA_UUID128_ALL, BT_UUID_CUSTOM_SERVICE_VAL),
};

/* 掃描回應資料：包含裝置名稱 */
static const struct bt_data sd[] = {
    BT_DATA(BT_DATA_NAME_COMPLETE, CONFIG_BT_DEVICE_NAME,
            sizeof(CONFIG_BT_DEVICE_NAME) - 1),
};

/* ============================
 * 連線管理回呼
 * ============================ */

/**
 * @brief 連線建立回呼
 */
static void connected(struct bt_conn *conn, uint8_t err)
{
    char addr[BT_ADDR_LE_STR_LEN];

    bt_addr_le_to_str(bt_conn_get_dst(conn), addr, sizeof(addr));

    if (err) {
        LOG_ERR("連線失敗 (addr: %s, err: %d)", addr, err);
        return;
    }

    LOG_INF("已連線: %s", addr);
    current_conn = bt_conn_ref(conn);

    /* 點亮連線 LED，熄滅廣播 LED */
    if (led_conn.port) {
        gpio_pin_set_dt(&led_conn, 1);
    }
    if (led_adv.port) {
        gpio_pin_set_dt(&led_adv, 0);
    }
}

/**
 * @brief 連線斷開回呼
 */
static void disconnected(struct bt_conn *conn, uint8_t reason)
{
    char addr[BT_ADDR_LE_STR_LEN];

    bt_addr_le_to_str(bt_conn_get_dst(conn), addr, sizeof(addr));
    LOG_INF("已斷線: %s (reason: 0x%02x)", addr, reason);

    if (current_conn) {
        bt_conn_unref(current_conn);
        current_conn = NULL;
    }

    notify_enabled = false;

    /* 熄滅連線 LED */
    if (led_conn.port) {
        gpio_pin_set_dt(&led_conn, 0);
    }

    /* 重新開始廣播 */
    int ret = bt_le_adv_start(BT_LE_ADV_CONN, ad, ARRAY_SIZE(ad),
                               sd, ARRAY_SIZE(sd));
    if (ret) {
        LOG_ERR("重新廣播失敗 (err: %d)", ret);
    } else {
        LOG_INF("重新開始廣播");
        if (led_adv.port) {
            gpio_pin_set_dt(&led_adv, 1);
        }
    }
}

/**
 * @brief 連線參數更新回呼
 */
static void le_param_updated(struct bt_conn *conn, uint16_t interval,
                             uint16_t latency, uint16_t timeout)
{
    LOG_INF("連線參數更新: interval=%d, latency=%d, timeout=%d",
            interval, latency, timeout);
}

/* 註冊連線回呼結構 */
BT_CONN_CB_DEFINE(conn_callbacks) = {
    .connected = connected,
    .disconnected = disconnected,
    .le_param_updated = le_param_updated,
};

/* ============================
 * LED 初始化
 * ============================ */

static int init_leds(void)
{
    int ret;

    if (led_adv.port) {
        ret = gpio_pin_configure_dt(&led_adv, GPIO_OUTPUT_INACTIVE);
        if (ret < 0) {
            LOG_ERR("LED0 設定失敗: %d", ret);
            return ret;
        }
    }

    if (led_conn.port) {
        ret = gpio_pin_configure_dt(&led_conn, GPIO_OUTPUT_INACTIVE);
        if (ret < 0) {
            LOG_ERR("LED1 設定失敗: %d", ret);
            return ret;
        }
    }

    return 0;
}

/* ============================
 * 主程式
 * ============================ */

int main(void)
{
    int err;

    LOG_INF("=== nRF54L15 BLE GATT Server 啟動 ===");

    /* 初始化 LED */
    init_leds();

    /* 初始化 BLE Stack */
    err = bt_enable(NULL);
    if (err) {
        LOG_ERR("BLE 初始化失敗 (err: %d)", err);
        return -1;
    }
    LOG_INF("BLE 初始化完成");

    /* 開始廣播 */
    err = bt_le_adv_start(BT_LE_ADV_CONN, ad, ARRAY_SIZE(ad),
                           sd, ARRAY_SIZE(sd));
    if (err) {
        LOG_ERR("廣播啟動失敗 (err: %d)", err);
        return -1;
    }
    LOG_INF("開始 BLE 廣播，等待連線...");

    if (led_adv.port) {
        gpio_pin_set_dt(&led_adv, 1);
    }

    /* 主迴圈：模擬感測器數值變化並發送通知 */
    while (1) {
        k_sleep(K_SECONDS(2));

        /* 模擬感測器數值微幅變動 */
        sensor_value += (k_cycle_get_32() % 11) - 5; /* -5 ~ +5 */

        /* 若有連線且通知已啟用，發送 Notification */
        if (current_conn && notify_enabled) {
            err = bt_gatt_notify(current_conn, &custom_svc.attrs[1],
                                 &sensor_value, sizeof(sensor_value));
            if (err) {
                LOG_WRN("通知發送失敗 (err: %d)", err);
            } else {
                LOG_INF("已發送通知: sensor_value=%d", sensor_value);
            }
        }
    }

    return 0;
}
```

### 6.2 GATT Client（Central 端）

以下範例實作一個 Central 裝置，掃描、連線到上述 Server，並讀寫其 Characteristic。

```c
/**
 * @file main.c
 * @brief nRF54L15 BLE GATT Client 完整範例
 *
 * 掃描並連線到 GATT Server，執行：
 * - 探索（Discover）自定義 Service
 * - 讀取 Sensor Data Characteristic
 * - 訂閱 Sensor Data 的 Notification
 * - 寫入 Control Point Characteristic
 *
 * 目標板：nrf54l15dk/nrf54l15/cpuapp
 */

#include <zephyr/kernel.h>
#include <zephyr/sys/printk.h>
#include <zephyr/sys/byteorder.h>
#include <zephyr/logging/log.h>

#include <zephyr/bluetooth/bluetooth.h>
#include <zephyr/bluetooth/hci.h>
#include <zephyr/bluetooth/conn.h>
#include <zephyr/bluetooth/uuid.h>
#include <zephyr/bluetooth/gatt.h>

LOG_MODULE_REGISTER(ble_gatt_client, LOG_LEVEL_INF);

/* 與 Server 端相同的 UUID 定義 */
#define BT_UUID_CUSTOM_SERVICE_VAL \
    BT_UUID_128_ENCODE(0x12345678, 0x1234, 0x5678, 0x1234, 0x56789abcdef0)
#define BT_UUID_CUSTOM_SERVICE BT_UUID_DECLARE_128(BT_UUID_CUSTOM_SERVICE_VAL)

#define BT_UUID_SENSOR_DATA_VAL \
    BT_UUID_128_ENCODE(0x12345678, 0x1234, 0x5678, 0x1234, 0x56789abcdef1)
#define BT_UUID_SENSOR_DATA BT_UUID_DECLARE_128(BT_UUID_SENSOR_DATA_VAL)

#define BT_UUID_CONTROL_POINT_VAL \
    BT_UUID_128_ENCODE(0x12345678, 0x1234, 0x5678, 0x1234, 0x56789abcdef2)
#define BT_UUID_CONTROL_POINT BT_UUID_DECLARE_128(BT_UUID_CONTROL_POINT_VAL)

/* ============================
 * 全域變數
 * ============================ */

static struct bt_conn *default_conn;

/* 記錄探索到的 Attribute Handle */
static uint16_t sensor_data_handle;
static uint16_t sensor_ccc_handle;
static uint16_t control_point_handle;

/* 探索參數 */
static struct bt_gatt_discover_params discover_params;
static struct bt_gatt_subscribe_params subscribe_params;

/* 同步信號 */
static K_SEM_DEFINE(sem_connected, 0, 1);
static K_SEM_DEFINE(sem_discovered, 0, 1);

/* ============================
 * GATT Notification 回呼
 * ============================ */

/**
 * @brief 收到 Notification 時的回呼
 *
 * 當 Server 發送 Sensor Data 的通知時觸發。
 */
static uint8_t notify_cb(struct bt_conn *conn,
                         struct bt_gatt_subscribe_params *params,
                         const void *data, uint16_t length)
{
    if (!data) {
        LOG_INF("通知訂閱已取消");
        params->value_handle = 0U;
        return BT_GATT_ITER_STOP;
    }

    if (length == sizeof(int16_t)) {
        int16_t value = sys_get_le16(data);
        LOG_INF("收到通知 — 感測器數值: %d.%02d°C",
                value / 100, (value % 100 < 0) ? -(value % 100) : value % 100);
    } else {
        LOG_WRN("通知資料長度異常: %d", length);
    }

    return BT_GATT_ITER_CONTINUE;
}

/* ============================
 * GATT Discovery 回呼
 * ============================ */

/**
 * @brief Service/Characteristic 探索回呼
 *
 * 逐步探索 Server 的 Service → Characteristic → Descriptor。
 * 探索流程：
 *   1. 探索 Primary Service（取得 Service 的 Handle 範圍）
 *   2. 探索 Characteristic（取得各 Characteristic 的 Value Handle）
 *   3. 探索 Descriptor（取得 CCCD Handle，用於啟用通知）
 */
static uint8_t discover_cb(struct bt_conn *conn,
                           const struct bt_gatt_attr *attr,
                           struct bt_gatt_discover_params *params)
{
    if (!attr) {
        LOG_INF("探索完成");
        memset(params, 0, sizeof(*params));
        k_sem_give(&sem_discovered);
        return BT_GATT_ITER_STOP;
    }

    LOG_INF("發現 Attribute — handle: %u", attr->handle);

    /* 階段 1：找到 Primary Service 後，繼續探索其 Characteristic */
    if (params->type == BT_GATT_DISCOVER_PRIMARY) {
        struct bt_gatt_service_val *svc = attr->user_data;
        LOG_INF("找到 Service — 起始 handle: %u, 結束 handle: %u",
                attr->handle, svc->end_handle);

        /* 切換為探索 Characteristic */
        params->uuid = NULL;
        params->start_handle = attr->handle + 1;
        params->end_handle = svc->end_handle;
        params->type = BT_GATT_DISCOVER_CHARACTERISTIC;

        int err = bt_gatt_discover(conn, params);
        if (err) {
            LOG_ERR("Characteristic 探索失敗 (err: %d)", err);
        }
        return BT_GATT_ITER_STOP;
    }

    /* 階段 2：找到 Characteristic */
    if (params->type == BT_GATT_DISCOVER_CHARACTERISTIC) {
        struct bt_gatt_chrc *chrc = attr->user_data;

        /* 比對 Sensor Data UUID */
        if (!bt_uuid_cmp(chrc->uuid, BT_UUID_SENSOR_DATA)) {
            sensor_data_handle = chrc->value_handle;
            LOG_INF("找到 Sensor Data — value handle: %u", sensor_data_handle);
        }
        /* 比對 Control Point UUID */
        else if (!bt_uuid_cmp(chrc->uuid, BT_UUID_CONTROL_POINT)) {
            control_point_handle = chrc->value_handle;
            LOG_INF("找到 Control Point — value handle: %u", control_point_handle);

            /* 找到最後一個 Characteristic 後，探索 Descriptor（CCCD） */
            params->uuid = BT_UUID_GATT_CCC;
            params->start_handle = sensor_data_handle + 1;
            params->end_handle = control_point_handle - 1;
            params->type = BT_GATT_DISCOVER_DESCRIPTOR;

            int err = bt_gatt_discover(conn, params);
            if (err) {
                LOG_ERR("Descriptor 探索失敗 (err: %d)", err);
            }
            return BT_GATT_ITER_STOP;
        }

        return BT_GATT_ITER_CONTINUE;
    }

    /* 階段 3：找到 CCCD Descriptor */
    if (params->type == BT_GATT_DISCOVER_DESCRIPTOR) {
        sensor_ccc_handle = attr->handle;
        LOG_INF("找到 CCCD — handle: %u", sensor_ccc_handle);

        k_sem_give(&sem_discovered);
        return BT_GATT_ITER_STOP;
    }

    return BT_GATT_ITER_STOP;
}

/* ============================
 * GATT Read 回呼
 * ============================ */

/**
 * @brief 讀取完成回呼
 */
static uint8_t read_cb(struct bt_conn *conn, uint8_t err,
                       struct bt_gatt_read_params *params,
                       const void *data, uint16_t length)
{
    if (err) {
        LOG_ERR("讀取失敗 (err: 0x%02x)", err);
        return BT_GATT_ITER_STOP;
    }

    if (data && length == sizeof(int16_t)) {
        int16_t value = sys_get_le16(data);
        LOG_INF("讀取成功 — 感測器數值: %d.%02d°C",
                value / 100, (value % 100 < 0) ? -(value % 100) : value % 100);
    }

    return BT_GATT_ITER_STOP;
}

/* ============================
 * GATT Write 回呼
 * ============================ */

/**
 * @brief 寫入完成回呼
 */
static void write_cb(struct bt_conn *conn, uint8_t err,
                     struct bt_gatt_write_params *params)
{
    if (err) {
        LOG_ERR("寫入失敗 (err: 0x%02x)", err);
    } else {
        LOG_INF("寫入成功");
    }
}

/* ============================
 * 掃描與連線
 * ============================ */

/**
 * @brief 掃描結果回呼
 *
 * 過濾包含自定義 Service UUID 的廣播封包，找到後停止掃描並連線。
 */
static void scan_cb(const bt_addr_le_t *addr, int8_t rssi,
                    uint8_t type, struct net_buf_simple *ad_buf)
{
    char addr_str[BT_ADDR_LE_STR_LEN];
    int err;

    /* 只處理可連線的廣播 */
    if (type != BT_GAP_ADV_TYPE_ADV_IND &&
        type != BT_GAP_ADV_TYPE_ADV_DIRECT_IND) {
        return;
    }

    /* 解析廣播資料，尋找自定義 Service UUID */
    struct net_buf_simple_state state;
    bool found = false;

    net_buf_simple_save(ad_buf, &state);

    while (ad_buf->len > 1) {
        uint8_t len = net_buf_simple_pull_u8(ad_buf);
        if (len == 0 || len > ad_buf->len) {
            break;
        }
        uint8_t ad_type = net_buf_simple_pull_u8(ad_buf);
        len--;

        if (ad_type == BT_DATA_UUID128_ALL && len == 16) {
            /* 比對 128-bit UUID */
            struct bt_uuid_128 uuid;
            uuid.uuid.type = BT_UUID_TYPE_128;
            memcpy(uuid.val, ad_buf->data, 16);

            if (!bt_uuid_cmp(&uuid.uuid, BT_UUID_CUSTOM_SERVICE)) {
                found = true;
                break;
            }
        }

        net_buf_simple_pull(ad_buf, len);
    }

    net_buf_simple_restore(ad_buf, &state);

    if (!found) {
        return;
    }

    bt_addr_le_to_str(addr, addr_str, sizeof(addr_str));
    LOG_INF("找到目標裝置: %s (RSSI: %d)", addr_str, rssi);

    /* 停止掃描 */
    err = bt_le_scan_stop();
    if (err) {
        LOG_ERR("停止掃描失敗 (err: %d)", err);
        return;
    }

    /* 發起連線 */
    struct bt_conn_le_create_param create_param =
        BT_CONN_LE_CREATE_PARAM_INIT(BT_CONN_LE_OPT_NONE,
                                      BT_GAP_SCAN_FAST_INTERVAL,
                                      BT_GAP_SCAN_FAST_WINDOW);
    struct bt_le_conn_param conn_param =
        BT_LE_CONN_PARAM_INIT(24, 40, 0, 400);

    err = bt_conn_le_create(addr, &create_param, &conn_param, &default_conn);
    if (err) {
        LOG_ERR("連線失敗 (err: %d)", err);
    }
}

/**
 * @brief 連線建立回呼
 */
static void connected(struct bt_conn *conn, uint8_t err)
{
    char addr[BT_ADDR_LE_STR_LEN];

    bt_addr_le_to_str(bt_conn_get_dst(conn), addr, sizeof(addr));

    if (err) {
        LOG_ERR("連線失敗: %s (err: %d)", addr, err);
        if (default_conn) {
            bt_conn_unref(default_conn);
            default_conn = NULL;
        }
        return;
    }

    LOG_INF("已連線: %s", addr);
    k_sem_give(&sem_connected);
}

/**
 * @brief 連線斷開回呼
 */
static void disconnected(struct bt_conn *conn, uint8_t reason)
{
    char addr[BT_ADDR_LE_STR_LEN];

    bt_addr_le_to_str(bt_conn_get_dst(conn), addr, sizeof(addr));
    LOG_INF("已斷線: %s (reason: 0x%02x)", addr, reason);

    if (default_conn) {
        bt_conn_unref(default_conn);
        default_conn = NULL;
    }
}

BT_CONN_CB_DEFINE(conn_callbacks) = {
    .connected = connected,
    .disconnected = disconnected,
};

/* ============================
 * 主程式
 * ============================ */

int main(void)
{
    int err;

    LOG_INF("=== nRF54L15 BLE GATT Client 啟動 ===");

    /* 初始化 BLE Stack */
    err = bt_enable(NULL);
    if (err) {
        LOG_ERR("BLE 初始化失敗 (err: %d)", err);
        return -1;
    }
    LOG_INF("BLE 初始化完成");

    /* 開始掃描 */
    struct bt_le_scan_param scan_param = {
        .type = BT_LE_SCAN_TYPE_ACTIVE,
        .options = BT_LE_SCAN_OPT_NONE,
        .interval = BT_GAP_SCAN_FAST_INTERVAL,
        .window = BT_GAP_SCAN_FAST_WINDOW,
    };

    err = bt_le_scan_start(&scan_param, scan_cb);
    if (err) {
        LOG_ERR("掃描啟動失敗 (err: %d)", err);
        return -1;
    }
    LOG_INF("開始掃描...");

    /* 等待連線建立 */
    k_sem_take(&sem_connected, K_FOREVER);
    LOG_INF("連線已建立，開始探索 Service...");

    /* 短暫等待讓連線穩定 */
    k_sleep(K_MSEC(500));

    /* 探索自定義 Service */
    discover_params.uuid = BT_UUID_CUSTOM_SERVICE;
    discover_params.func = discover_cb;
    discover_params.start_handle = BT_ATT_FIRST_ATTRIBUTE_HANDLE;
    discover_params.end_handle = BT_ATT_LAST_ATTRIBUTE_HANDLE;
    discover_params.type = BT_GATT_DISCOVER_PRIMARY;

    err = bt_gatt_discover(default_conn, &discover_params);
    if (err) {
        LOG_ERR("Service 探索啟動失敗 (err: %d)", err);
        return -1;
    }

    /* 等待探索完成 */
    k_sem_take(&sem_discovered, K_FOREVER);
    LOG_INF("Service 探索完成");

    /* 步驟 1：讀取 Sensor Data */
    if (sensor_data_handle) {
        static struct bt_gatt_read_params read_params;
        read_params.func = read_cb;
        read_params.handle_count = 1;
        read_params.single.handle = sensor_data_handle;
        read_params.single.offset = 0;

        err = bt_gatt_read(default_conn, &read_params);
        if (err) {
            LOG_ERR("讀取請求失敗 (err: %d)", err);
        }
        k_sleep(K_SECONDS(1));
    }

    /* 步驟 2：訂閱 Sensor Data 的通知 */
    if (sensor_ccc_handle && sensor_data_handle) {
        subscribe_params.notify = notify_cb;
        subscribe_params.value_handle = sensor_data_handle;
        subscribe_params.ccc_handle = sensor_ccc_handle;
        subscribe_params.value = BT_GATT_CCC_NOTIFY;

        err = bt_gatt_subscribe(default_conn, &subscribe_params);
        if (err) {
            LOG_ERR("訂閱失敗 (err: %d)", err);
        } else {
            LOG_INF("已訂閱 Sensor Data 通知");
        }
    }

    /* 步驟 3：寫入 Control Point */
    if (control_point_handle) {
        static struct bt_gatt_write_params write_params;
        uint8_t cmd = 0x02; /* 增加感測器數值 */

        write_params.func = write_cb;
        write_params.handle = control_point_handle;
        write_params.offset = 0;
        write_params.data = &cmd;
        write_params.length = sizeof(cmd);

        k_sleep(K_SECONDS(1));

        err = bt_gatt_write(default_conn, &write_params);
        if (err) {
            LOG_ERR("寫入請求失敗 (err: %d)", err);
        }
    }

    /* 持續接收通知 */
    LOG_INF("持續接收通知中...");
    while (1) {
        k_sleep(K_SECONDS(5));

        if (!default_conn) {
            LOG_INF("連線已斷開，重新掃描...");
            err = bt_le_scan_start(&scan_param, scan_cb);
            if (err) {
                LOG_ERR("重新掃描失敗 (err: %d)", err);
            }
        }
    }

    return 0;
}
```

---

## 7. 進階用法

### 7.1 MTU 交換（提升吞吐量）

連線建立後，預設 ATT MTU 為 23 bytes（有效 payload 僅 20 bytes）。透過 MTU Exchange 可提升至 247 bytes，大幅減少分段傳輸：

```c
/* 在 connected 回呼中發起 MTU Exchange */
static void connected(struct bt_conn *conn, uint8_t err)
{
    if (!err) {
        /* 設定交換參數 */
        static struct bt_gatt_exchange_params exchange_params;
        exchange_params.func = mtu_exchange_cb;

        int ret = bt_gatt_exchange_mtu(conn, &exchange_params);
        if (ret) {
            LOG_ERR("MTU 交換失敗 (err: %d)", ret);
        }
    }
}

/* MTU 交換完成回呼 */
static void mtu_exchange_cb(struct bt_conn *conn, uint8_t err,
                            struct bt_gatt_exchange_params *params)
{
    if (err) {
        LOG_ERR("MTU 交換錯誤: 0x%02x", err);
    } else {
        uint16_t mtu = bt_gatt_get_mtu(conn);
        LOG_INF("MTU 交換成功: %u bytes", mtu);
    }
}
```

### 7.2 連線參數協商（省電優化）

Peripheral 可以在資料不頻繁時要求較大的連線間隔來省電：

```c
/* 請求較長的連線間隔以省電 */
static void request_low_power_params(struct bt_conn *conn)
{
    /* 連線間隔 500ms ~ 1000ms，Slave Latency 4（可跳過 4 次連線事件） */
    struct bt_le_conn_param param = BT_LE_CONN_PARAM_INIT(400, 800, 4, 600);

    int err = bt_conn_le_param_update(conn, &param);
    if (err) {
        LOG_ERR("連線參數更新請求失敗 (err: %d)", err);
    }
}

/* 需要快速傳輸時切換回高效能參數 */
static void request_high_throughput_params(struct bt_conn *conn)
{
    /* 連線間隔 7.5ms ~ 15ms，無 Slave Latency */
    struct bt_le_conn_param param = BT_LE_CONN_PARAM_INIT(6, 12, 0, 400);

    int err = bt_conn_le_param_update(conn, &param);
    if (err) {
        LOG_ERR("連線參數更新請求失敗 (err: %d)", err);
    }
}
```

### 7.3 多重 Notification（批次推送）

當有多個 Characteristic 需要同時通知時，可使用 `bt_gatt_notify_cb` 或在同一連線事件中連續呼叫：

```c
/* 定義多組感測器資料 */
static int16_t temperature = 2500;
static uint16_t humidity = 6500;
static uint16_t pressure = 10132;

/* 批次發送多個通知 */
static void send_all_notifications(struct bt_conn *conn)
{
    /* 溫度通知 */
    bt_gatt_notify(conn, &sensor_svc.attrs[1],
                   &temperature, sizeof(temperature));

    /* 濕度通知 */
    bt_gatt_notify(conn, &sensor_svc.attrs[4],
                   &humidity, sizeof(humidity));

    /* 氣壓通知 */
    bt_gatt_notify(conn, &sensor_svc.attrs[7],
                   &pressure, sizeof(pressure));
}
```

### 7.4 安全連線（配對加密）

若 Characteristic 需要加密保護，可在 Permission 中設定：

```c
/* 需要加密才能讀取的 Characteristic */
BT_GATT_CHARACTERISTIC(BT_UUID_SENSOR_DATA,
                       BT_GATT_CHRC_READ | BT_GATT_CHRC_NOTIFY,
                       BT_GATT_PERM_READ_ENCRYPT,  /* 需要加密連線 */
                       read_sensor_data, NULL, &sensor_value),

/* 需要認證（MITM 保護）才能寫入的 Characteristic */
BT_GATT_CHARACTERISTIC(BT_UUID_CONTROL_POINT,
                       BT_GATT_CHRC_WRITE,
                       BT_GATT_PERM_WRITE_AUTHEN,  /* 需要認證 */
                       NULL, write_control_point, NULL),
```

對應的 Kconfig 設定：

```kconfig
CONFIG_BT_SMP=y
CONFIG_BT_BONDABLE=y
CONFIG_BT_SMP_SC_ONLY=y     # 僅允許 Secure Connections
CONFIG_BT_FIXED_PASSKEY=y    # 使用固定 Passkey（開發用）
```

---

## 8. 常見問題與除錯

### 8.1 FAQ

**Q1：連線後馬上斷線（reason: 0x3e CONN_TIMEOUT）**

原因：連線參數不相容，或 RF 信號太弱。

解決：
- 確認 Supervision Timeout ≥ (1 + Slave Latency) × Connection Interval × 2
- 拉近裝置距離測試
- 檢查天線連接

**Q2：GATT Discover 回呼沒有收到任何 Attribute**

原因：Service 可能未正確註冊，或 UUID 不匹配。

解決：
```bash
# 使用 nRF Connect App 確認 Server 的 Service 確實存在
# 檢查 UUID 的 byte order（Zephyr 使用 little-endian 編碼）
# 確認 BT_GATT_SERVICE_DEFINE 放在全域範圍（非函式內）
```

**Q3：bt_gatt_notify() 回傳 -EINVAL (-22)**

原因：CCCD 未被 Client 啟用，或傳入的 Attribute 指標不正確。

解決：
- 確認 `notify_enabled` 旗標在 `ccc_cfg_changed` 回呼中正確設定
- 確認傳入的 `attr` 指標指向 Characteristic Value（非 Declaration）
- `&svc.attrs[1]` 指向第一個 Characteristic 的 Value Attribute（index 0 是 Service Declaration）

**Q4：寫入大於 20 bytes 的資料失敗**

原因：預設 ATT MTU 為 23（header 3 + payload 20）。

解決：
- 在連線後執行 MTU Exchange
- 設定 `CONFIG_BT_L2CAP_TX_MTU=247` 及對應的 buffer 大小
- 或使用 `bt_gatt_write_without_response()` 搭配 Long Write

**Q5：多個 Client 連線時通知混亂**

原因：每個連線的 CCCD 狀態獨立管理。

解決：
- 使用 `bt_gatt_notify(NULL, ...)` 對所有已訂閱的連線發送通知
- 或迭代連線清單，逐一發送

### 8.2 除錯技巧

**啟用詳細 BLE Log：**

```kconfig
# prj.conf 中加入
CONFIG_LOG=y
CONFIG_BT_LOG_LEVEL_DBG=y          # BLE 總體 debug log
CONFIG_BT_ATT_LOG_LEVEL_DBG=y      # ATT 層 debug log
CONFIG_BT_GATT_LOG_LEVEL_DBG=y     # GATT 層 debug log
CONFIG_BT_HCI_DRIVER_LOG_LEVEL_DBG=y # HCI 層 debug log
```

**使用 nRF Sniffer 抓取空中封包：**

1. 準備另一片 nRF DK，燒錄 nRF Sniffer 韌體
2. 安裝 Wireshark + nRF Sniffer 外掛
3. 篩選 BLE ATT/GATT 封包分析

**使用 nRF Connect for Mobile 測試 Server：**

1. 下載 nRF Connect App（iOS/Android）
2. 掃描並連線到你的裝置
3. 瀏覽 Service → Characteristic
4. 手動執行 Read / Write / Subscribe 測試

### 8.3 常見錯誤碼

| 錯誤碼 | 名稱 | 說明 |
|--------|------|------|
| 0x02 | `BT_ATT_ERR_READ_NOT_PERMITTED` | Characteristic 不支援 Read |
| 0x03 | `BT_ATT_ERR_WRITE_NOT_PERMITTED` | Characteristic 不支援 Write |
| 0x05 | `BT_ATT_ERR_AUTHENTICATION` | 需要配對認證 |
| 0x06 | `BT_ATT_ERR_NOT_SUPPORTED` | Server 不支援此操作 |
| 0x07 | `BT_ATT_ERR_INVALID_OFFSET` | 偏移量無效 |
| 0x0D | `BT_ATT_ERR_INVALID_ATTRIBUTE_LEN` | 資料長度無效 |
| 0x0E | `BT_ATT_ERR_ENCRYPTION_KEY_SIZE` | 加密金鑰長度不足 |

---

## 9. API 快速參考

### 9.1 連線管理 API

| 函式 | 簽名 | 說明 |
|------|------|------|
| `bt_enable` | `int bt_enable(bt_ready_cb_t cb)` | 初始化 BLE Stack |
| `bt_le_adv_start` | `int bt_le_adv_start(const struct bt_le_adv_param *param, const struct bt_data *ad, size_t ad_len, const struct bt_data *sd, size_t sd_len)` | 開始廣播 |
| `bt_le_adv_stop` | `int bt_le_adv_stop(void)` | 停止廣播 |
| `bt_le_scan_start` | `int bt_le_scan_start(const struct bt_le_scan_param *param, bt_le_scan_cb_t cb)` | 開始掃描 |
| `bt_le_scan_stop` | `int bt_le_scan_stop(void)` | 停止掃描 |
| `bt_conn_le_create` | `int bt_conn_le_create(const bt_addr_le_t *peer, const struct bt_conn_le_create_param *create_param, const struct bt_le_conn_param *conn_param, struct bt_conn **conn)` | 發起 BLE 連線 |
| `bt_conn_disconnect` | `int bt_conn_disconnect(struct bt_conn *conn, uint8_t reason)` | 斷開連線 |
| `bt_conn_le_param_update` | `int bt_conn_le_param_update(struct bt_conn *conn, const struct bt_le_conn_param *param)` | 更新連線參數 |

### 9.2 GATT Server API

| 函式/巨集 | 簽名 | 說明 |
|-----------|------|------|
| `BT_GATT_SERVICE_DEFINE` | `BT_GATT_SERVICE_DEFINE(name, attrs...)` | 靜態定義 GATT Service |
| `BT_GATT_PRIMARY_SERVICE` | `BT_GATT_PRIMARY_SERVICE(uuid)` | 定義 Primary Service |
| `BT_GATT_CHARACTERISTIC` | `BT_GATT_CHARACTERISTIC(uuid, props, perm, read, write, user_data)` | 定義 Characteristic |
| `BT_GATT_CCC` | `BT_GATT_CCC(cfg_changed, perm)` | 定義 CCCD |
| `bt_gatt_notify` | `int bt_gatt_notify(struct bt_conn *conn, const struct bt_gatt_attr *attr, const void *data, uint16_t len)` | 發送通知 |
| `bt_gatt_indicate` | `int bt_gatt_indicate(struct bt_conn *conn, struct bt_gatt_indicate_params *params)` | 發送指示（需確認） |
| `bt_gatt_attr_read` | `ssize_t bt_gatt_attr_read(struct bt_conn *conn, const struct bt_gatt_attr *attr, void *buf, uint16_t buf_len, uint16_t offset, const void *value, uint16_t value_len)` | 輔助讀取函式 |

### 9.3 GATT Client API

| 函式 | 簽名 | 說明 |
|------|------|------|
| `bt_gatt_discover` | `int bt_gatt_discover(struct bt_conn *conn, struct bt_gatt_discover_params *params)` | 探索遠端 Service/Characteristic |
| `bt_gatt_read` | `int bt_gatt_read(struct bt_conn *conn, struct bt_gatt_read_params *params)` | 讀取遠端 Characteristic |
| `bt_gatt_write` | `int bt_gatt_write(struct bt_conn *conn, struct bt_gatt_write_params *params)` | 寫入遠端 Characteristic（有回應） |
| `bt_gatt_write_without_response` | `int bt_gatt_write_without_response(struct bt_conn *conn, uint16_t handle, const void *data, uint16_t length, bool sign)` | 寫入遠端 Characteristic（無回應） |
| `bt_gatt_subscribe` | `int bt_gatt_subscribe(struct bt_conn *conn, struct bt_gatt_subscribe_params *params)` | 訂閱通知/指示 |
| `bt_gatt_unsubscribe` | `int bt_gatt_unsubscribe(struct bt_conn *conn, struct bt_gatt_subscribe_params *params)` | 取消訂閱 |
| `bt_gatt_exchange_mtu` | `int bt_gatt_exchange_mtu(struct bt_conn *conn, struct bt_gatt_exchange_params *params)` | 協商 MTU 大小 |

---

## 10. 延伸閱讀

### Nordic 官方文件

- [nRF Connect SDK — Bluetooth LE fundamentals](https://docs.nordicsemi.com/bundle/ncs-latest/page/nrf/protocols/bt/index.html)
- [nRF Connect SDK — BLE Peripheral samples](https://docs.nordicsemi.com/bundle/ncs-latest/page/nrf/samples/bluetooth.html)
- [nRF54L15 Product Specification](https://docs.nordicsemi.com/bundle/ps_nrf54L15/page/keyfeatures_html5.html)

### Zephyr 官方文件

- [Zephyr — Bluetooth Host API](https://docs.zephyrproject.org/latest/connectivity/bluetooth/api/index.html)
- [Zephyr — GATT Server API](https://docs.zephyrproject.org/latest/connectivity/bluetooth/api/gatt.html)
- [Zephyr — BLE Peripheral sample](https://docs.zephyrproject.org/latest/samples/bluetooth/peripheral/README.html)
- [Zephyr — BLE Central sample](https://docs.zephyrproject.org/latest/samples/bluetooth/central/README.html)

### 實用工具

- [nRF Connect for Mobile](https://www.nordicsemi.com/Products/Development-tools/nRF-Connect-for-mobile) — iOS/Android BLE 測試工具
- [nRF Sniffer for Bluetooth LE](https://www.nordicsemi.com/Products/Development-tools/nRF-Sniffer-for-Bluetooth-LE) — Wireshark 封包擷取
- [Bluetooth SIG — GATT Specification](https://www.bluetooth.com/specifications/specs/core-specification/) — 官方協定規格

### 建置與燒錄指令

```bash
# 建置 GATT Server
west build -b nrf54l15dk/nrf54l15/cpuapp -d build_server -- -DOVERLAY_CONFIG=prj.conf

# 建置 GATT Client
west build -b nrf54l15dk/nrf54l15/cpuapp -d build_client -- -DOVERLAY_CONFIG=prj_client.conf

# 燒錄到 DK 板
west flash -d build_server

# 監看 Log 輸出
west espressif monitor   # 或使用
minicom -D /dev/ttyACM0 -b 115200
```
