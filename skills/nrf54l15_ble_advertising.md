---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: wireless
difficulty: beginner
keywords:
  - BLE
  - Bluetooth LE
  - advertising
  - scan response
  - advertising interval
  - GAP
  - broadcaster
  - peripheral
  - bt_le_adv_start
  - bt_data
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-03-31
---

# BLE 廣播：設定廣播封包、掃描回應、廣播間隔

## 1. 概述

### 什麼是 BLE 廣播？

BLE（Bluetooth Low Energy）廣播是藍牙低功耗裝置向周圍環境「主動喊話」的機制。裝置透過在三個固定的廣播頻道（Channel 37、38、39）上週期性地發送短小的資料封包，讓周圍的掃描裝置（如手機、閘道器）發現自己。

廣播是 BLE 通訊的第一步 — 在建立連線之前，外圍裝置（Peripheral）必須先廣播自己的存在。即使不建立連線，廣播本身也可以攜帶有用的資訊（如 iBeacon、感測器數據），這種模式稱為 Broadcaster。

### 為什麼重要？

- **裝置發現**：手機 App 必須先掃描到廣播才能連線
- **無連線資料傳輸**：溫度感測器可以直接在廣播封包中發送數據，不需要建立連線
- **功耗控制**：廣播間隔直接影響電池壽命 — 間隔越長越省電
- **使用者體驗**：廣播間隔影響裝置被發現的速度

### 適用場景

| 場景 | 說明 |
|------|------|
| BLE Peripheral | 廣播後等待中央裝置連線（如心率帶、智慧手錶） |
| Beacon | 持續廣播定位資訊或 URL（iBeacon、Eddystone） |
| 感測器廣播 | 在廣播封包中攜帶感測數據，無需連線 |
| 配對模式 | 裝置進入可被發現狀態，等待使用者配對 |

### BLE 廣播類型

| 類型 | PDU Type | 可連線 | 可掃描 | 說明 |
|------|----------|--------|--------|------|
| ADV_IND | 0x00 | ✅ | ✅ | 通用可連線廣播（最常用） |
| ADV_DIRECT_IND | 0x01 | ✅ | ❌ | 定向可連線廣播（指定目標） |
| ADV_SCAN_IND | 0x02 | ❌ | ✅ | 可掃描但不可連線 |
| ADV_NONCONN_IND | 0x03 | ❌ | ❌ | 不可連線不可掃描（純 Beacon） |

---

## 2. 硬體概念

### nRF54L15 無線電架構

nRF54L15 內建 2.4 GHz 無線電收發器，支援 Bluetooth 5.4。廣播相關的硬體元件：

```
┌─────────────────────────────────────────────┐
│                 nRF54L15 SoC                │
│                                             │
│  ┌──────────┐    ┌──────────────────────┐   │
│  │  ARM     │    │   2.4 GHz Radio      │   │
│  │ Cortex-  │◄──►│  ┌────────────────┐  │   │
│  │  M33     │    │  │ BLE Controller │  │   │
│  │ (128MHz) │    │  │  (Link Layer)  │  │   │
│  └──────────┘    │  └────────────────┘  │   │
│       │          │         │             │   │
│       │          │  ┌──────┴───────┐     │   │
│       │          │  │  PA / LNA    │     │   │
│       │          │  │  (功率放大)   │     │   │
│       │          │  └──────┬───────┘     │   │
│       │          └─────────┼─────────────┘   │
│       │                    │                 │
└───────┼────────────────────┼─────────────────┘
        │                    │
   SWD/UART              天線接口
```

### 廣播頻道

BLE 使用 40 個頻道（0-39），其中三個專用於廣播：

| 頻道 | 頻率 | 用途 |
|------|------|------|
| Channel 37 | 2402 MHz | 廣播頻道 |
| Channel 38 | 2426 MHz | 廣播頻道 |
| Channel 39 | 2480 MHz | 廣播頻道 |
| Channel 0-36 | 2404-2478 MHz | 資料頻道 |

這三個廣播頻道刻意分散在 2.4 GHz 頻段中，以避開 Wi-Fi 頻道 1、6、11 的干擾。

### 廣播封包結構

一個 BLE 廣播封包的最大有效負載為 **31 bytes**（Legacy Advertising）：

```
┌──────────────┬──────────────┬───────────────────────┐
│  Preamble    │ Access Addr  │      PDU              │
│  (1 byte)    │ (4 bytes)    │                       │
└──────────────┴──────────────┤                       │
                              │ ┌────────┬──────────┐ │
                              │ │ Header │ Payload  │ │
                              │ │(2 byte)│(≤37 byte)│ │
                              │ └────────┴──────────┘ │
                              └───────────────────────┘

Payload 細節：
┌─────────────────┬──────────────────────────────┐
│  AdvA (6 bytes)  │  AdvData (0-31 bytes)        │
│  廣播者地址      │  廣播資料                     │
└─────────────────┴──────────────────────────────┘

AdvData 由多個 AD Structure 組成：
┌────────┬──────┬────────────┐┌────────┬──────┬────────────┐
│ Length │ Type │   Data     ││ Length │ Type │   Data     │
│(1 byte)│(1 b) │(Length-1 b)││(1 byte)│(1 b) │(Length-1 b)│
└────────┴──────┴────────────┘└────────┴──────┴────────────┘
```

### 廣播間隔與功耗

廣播間隔（Advertising Interval）是兩次廣播事件之間的時間。BLE 規格要求：
- **最小間隔**：20 ms
- **最大間隔**：10,485.76 ms（約 10.24 秒）
- **單位**：0.625 ms（即設定值 × 0.625 ms）
- **隨機延遲**：每次廣播會自動加上 0-10 ms 的隨機延遲，避免多裝置碰撞

功耗與間隔的關係（nRF54L15 典型值，0 dBm TX power）：

| 廣播間隔 | 大約電流消耗 | 適用場景 |
|----------|------------|----------|
| 20 ms | ~1.5 mA | 需要極速發現（僅短時間使用） |
| 100 ms | ~0.3 mA | 快速發現（配對模式） |
| 1000 ms (1s) | ~0.03 mA | 一般 Beacon |
| 10000 ms (10s) | ~0.003 mA | 超低功耗 Beacon |

---

## 3. 軟體架構

### Zephyr BLE 協定堆疊

nRF Connect SDK 使用 Zephyr 的 Bluetooth 子系統，架構如下：

```
┌───────────────────────────────────────┐
│           應用層 (Application)         │
│    bt_le_adv_start(), bt_data, ...    │
├───────────────────────────────────────┤
│           GAP (Generic Access Profile) │
│      廣播管理、連線管理、安全管理        │
├───────────────────────────────────────┤
│        Host (Zephyr Bluetooth Host)    │
│    HCI 命令組裝、事件處理               │
├───────────────────────────────────────┤
│        HCI (Host Controller Interface) │
├───────────────────────────────────────┤
│     Controller (SoftDevice Controller  │
│      或 Zephyr Controller)             │
│    Link Layer 狀態機、排程              │
├───────────────────────────────────────┤
│         Radio Hardware (2.4 GHz)       │
└───────────────────────────────────────┘
```

### 關鍵資料結構

#### `struct bt_data` — 廣播資料元素

每個 `bt_data` 代表廣播封包中的一個 AD Structure：

```c
struct bt_data {
    uint8_t type;         /* AD Type（如 BT_DATA_FLAGS, BT_DATA_NAME_COMPLETE） */
    uint8_t data_len;     /* 資料長度（bytes） */
    const uint8_t *data;  /* 指向資料內容的指標 */
};
```

#### `struct bt_le_adv_param` — 廣播參數

控制廣播行為的參數結構：

```c
struct bt_le_adv_param {
    uint8_t  id;            /* 本地身份（通常為 BT_ID_DEFAULT = 0） */
    uint8_t  sid;           /* 廣播集合 ID（Extended Advertising 用） */
    uint8_t  secondary_max_skip; /* 次要廣播最大跳過次數 */
    uint32_t options;       /* 選項旗標（BT_LE_ADV_OPT_*） */
    uint32_t interval_min;  /* 最小廣播間隔（單位：0.625 ms） */
    uint32_t interval_max;  /* 最大廣播間隔（單位：0.625 ms） */
    const bt_addr_le_t *peer; /* 定向廣播目標地址（NULL = 非定向） */
};
```

### 常用廣播選項旗標（`options` 欄位）

| 旗標 | 值 | 說明 |
|------|-----|------|
| `BT_LE_ADV_OPT_CONNECTABLE` | 0x01 | 允許其他裝置連線 |
| `BT_LE_ADV_OPT_SCANNABLE` | 0x02 | 允許掃描請求（Scan Request） |
| `BT_LE_ADV_OPT_USE_NAME` | 0x04 | 自動將裝置名稱加入廣播資料 |
| `BT_LE_ADV_OPT_USE_IDENTITY` | 0x08 | 使用身份地址（非隨機地址） |
| `BT_LE_ADV_OPT_NO_2M` | 0x10 | 不使用 2M PHY |
| `BT_LE_ADV_OPT_CODED` | 0x20 | 使用 Coded PHY（長距離模式） |
| `BT_LE_ADV_OPT_ONE_TIME` | 0x80 | 連線後自動停止廣播 |

### 廣播間隔巨集

Zephyr 提供方便的巨集將毫秒轉換為 BLE 單位：

```c
/* 將毫秒轉為 0.625 ms 單位 */
#define BT_GAP_ADV_FAST_INT_MIN_1   0x0030  /* 30 ms */
#define BT_GAP_ADV_FAST_INT_MIN_2   0x0060  /* 60 ms */
#define BT_GAP_ADV_FAST_INT_MAX_2   0x00A0  /* 100 ms */
#define BT_GAP_ADV_SLOW_INT_MIN     0x0640  /* 1000 ms */
#define BT_GAP_ADV_SLOW_INT_MAX     0x0780  /* 1200 ms */
```

---

## 4. Devicetree 設定

BLE 廣播本身不需要額外的 Devicetree 設定，因為無線電硬體已在 nRF54L15 的 SoC dtsi 中定義。但如果你需要自訂 GPIO（如 LED 指示廣播狀態），可以使用 overlay：

### 檔案：`boards/nrf54l15dk_nrf54l15_cpuapp.overlay`

```dts
/*
 * nRF54L15 BLE 廣播範例 — Devicetree Overlay
 *
 * 本 overlay 定義一顆 LED 用於指示廣播狀態。
 * nRF54L15DK 開發板已內建 4 顆 LED（LED0-LED3），
 * 這裡我們使用 LED0 作為廣播狀態指示。
 */

/ {
    /* 定義別名方便程式碼引用 */
    aliases {
        adv-led = &led0;  /* 廣播狀態指示 LED */
    };
};

/*
 * 如果你使用外接天線而非板載天線，
 * 可能需要設定天線切換 GPIO：
 *
 * &radio {
 *     ant-sel-gpios = <&gpio0 4 GPIO_ACTIVE_HIGH>;
 * };
 */
```

### 各屬性說明

| 屬性 | 說明 |
|------|------|
| `aliases` | 為 Devicetree 節點建立短名稱，方便 C 程式碼用 `DT_ALIAS()` 取得 |
| `adv-led` | 自定義別名，指向 `led0` 節點 |
| `ant-sel-gpios` | 天線選擇 GPIO（僅外接天線需要） |

> **注意**：nRF54L15DK 的 BLE 無線電在 SoC 層級 dtsi 中已完整定義，一般應用不需要修改無線電相關的 Devicetree 設定。

---

## 5. Kconfig 設定

### 檔案：`prj.conf`

```kconfig
#------------------------------------------------------
# 基本藍牙功能
#------------------------------------------------------
# 啟用藍牙子系統（必要）
CONFIG_BT=y

# 啟用 BLE Peripheral 角色支援（包含廣播功能）
# 如果只需要純廣播（Broadcaster），可以不啟用此項
CONFIG_BT_PERIPHERAL=y

# 設定裝置名稱（會出現在廣播封包中）
CONFIG_BT_DEVICE_NAME="nRF54L15-Adv-Demo"

# 設定裝置名稱最大長度
CONFIG_BT_DEVICE_NAME_MAX=32

# 啟用裝置名稱可動態修改（透過 bt_set_name()）
CONFIG_BT_DEVICE_NAME_DYNAMIC=y

#------------------------------------------------------
# 廣播相關設定
#------------------------------------------------------
# 最大同時廣播集合數量（Extended Advertising 用）
# Legacy Advertising 只需要 1
CONFIG_BT_EXT_ADV_MAX_ADV_SET=1

# 廣播資料最大長度（Legacy 最大 31 bytes）
# Extended Advertising 可設更大值
# CONFIG_BT_CTLR_ADV_DATA_LEN_MAX=31

#------------------------------------------------------
# 連線相關（如果廣播可連線才需要）
#------------------------------------------------------
# 最大同時連線數
CONFIG_BT_MAX_CONN=1

# 最大配對裝置數
CONFIG_BT_MAX_PAIRED=2

#------------------------------------------------------
# 安全性（可選）
#------------------------------------------------------
# 啟用 Security Manager Protocol
# CONFIG_BT_SMP=y

# 啟用隱私模式（使用 RPA 隨機地址）
# CONFIG_BT_PRIVACY=y

#------------------------------------------------------
# 除錯（開發時啟用，量產關閉）
#------------------------------------------------------
# 啟用 Bluetooth 日誌
CONFIG_BT_LOG_LEVEL_DBG=y

# 啟用控制台日誌
CONFIG_LOG=y
CONFIG_LOG_DEFAULT_LEVEL=3

# 啟用 UART 控制台（用於看 printk / LOG 輸出）
CONFIG_CONSOLE=y
CONFIG_UART_CONSOLE=y

#------------------------------------------------------
# GPIO（LED 控制用）
#------------------------------------------------------
CONFIG_GPIO=y
```

### 各設定項詳細說明

| 設定項 | 預設值 | 說明 |
|--------|--------|------|
| `CONFIG_BT` | n | 啟用 Bluetooth 子系統，這是使用任何 BLE 功能的前提 |
| `CONFIG_BT_PERIPHERAL` | n | 啟用 Peripheral 角色，包含可連線廣播所需的 GAP 功能 |
| `CONFIG_BT_BROADCASTER` | n | 僅啟用 Broadcaster 角色（不可連線廣播），比 PERIPHERAL 省 ROM |
| `CONFIG_BT_DEVICE_NAME` | "Zephyr" | 預設裝置名稱，會被包含在廣播封包的 Complete Local Name 欄位 |
| `CONFIG_BT_DEVICE_NAME_DYNAMIC` | n | 允許在執行階段用 `bt_set_name()` 修改裝置名稱 |
| `CONFIG_BT_EXT_ADV_MAX_ADV_SET` | 1 | Extended Advertising 集合數量上限 |
| `CONFIG_BT_MAX_CONN` | 1 | 最大同時 BLE 連線數，影響 RAM 使用量 |
| `CONFIG_BT_LOG_LEVEL_DBG` | n | 啟用 Bluetooth 子系統的 Debug 級別日誌 |

### 僅廣播（Broadcaster）最小設定

如果你的裝置只需要廣播、不需要接受連線，可以用更精簡的設定：

```kconfig
CONFIG_BT=y
CONFIG_BT_BROADCASTER=y
CONFIG_BT_DEVICE_NAME="nRF54L15-Beacon"
CONFIG_LOG=y
```

這比 `CONFIG_BT_PERIPHERAL=y` 節省約 2-4 KB ROM。

---

## 6. 完整程式碼範例

### 檔案：`src/main.c`

```c
/*
 * BLE 廣播完整範例 — nRF54L15
 *
 * 功能：
 * 1. 啟動 BLE 並設定可連線可掃描廣播
 * 2. 廣播封包包含：Flags、裝置名稱、自訂服務 UUID
 * 3. 掃描回應包含：完整裝置名稱、製造商特定資料
 * 4. 使用 LED 指示廣播狀態
 * 5. 按鈕切換廣播開關
 *
 * 目標板：nrf54l15dk/nrf54l15/cpuapp
 */

#include <zephyr/kernel.h>           /* Zephyr 核心 API（k_sleep 等） */
#include <zephyr/sys/printk.h>       /* printk 除錯輸出 */
#include <zephyr/logging/log.h>      /* 結構化日誌系統 */
#include <zephyr/bluetooth/bluetooth.h> /* BLE 核心 API（bt_enable 等） */
#include <zephyr/bluetooth/hci.h>    /* HCI 定義（地址類型等） */
#include <zephyr/bluetooth/gap.h>    /* GAP 定義（廣播間隔常數等） */
#include <zephyr/drivers/gpio.h>     /* GPIO API（LED 控制） */

/* 註冊日誌模組，名稱為 "ble_adv"，預設日誌等級 INFO */
LOG_MODULE_REGISTER(ble_adv, LOG_LEVEL_INF);

/*============================================================
 * 常數定義
 *============================================================*/

/*
 * 廣播間隔設定
 * BLE 規格中，間隔的單位是 0.625 ms
 * BT_GAP_ADV_FAST_INT_MIN_2 = 0x0060 = 96 → 96 × 0.625 = 60 ms
 * BT_GAP_ADV_FAST_INT_MAX_2 = 0x00A0 = 160 → 160 × 0.625 = 100 ms
 *
 * 控制器會在 [min, max] 範圍內選擇實際間隔，
 * 並額外加上 0-10 ms 隨機延遲。
 */
#define ADV_INT_MIN BT_GAP_ADV_FAST_INT_MIN_2  /* 最小間隔 60 ms */
#define ADV_INT_MAX BT_GAP_ADV_FAST_INT_MAX_2  /* 最大間隔 100 ms */

/*
 * 自訂 128-bit UUID（用於辨識你的服務）
 * 格式：以位元組反序排列（little-endian）
 * 原始 UUID：12345678-1234-5678-1234-56789abcdef0
 */
#define CUSTOM_SERVICE_UUID \
    BT_UUID_128_ENCODE(0x12345678, 0x1234, 0x5678, 0x1234, 0x56789abcdef0)

/*
 * 製造商 ID（Bluetooth SIG 分配）
 * 0xFFFF 是測試/開發用途
 * 正式產品需要向 Bluetooth SIG 申請自己的 Company ID
 */
#define MANUFACTURER_ID_LOW  0xFF
#define MANUFACTURER_ID_HIGH 0xFF

/*============================================================
 * LED 設定（使用 Devicetree）
 *============================================================*/

/* 從 Devicetree 取得 LED0 節點（nRF54L15DK 開發板內建） */
static const struct gpio_dt_spec adv_led =
    GPIO_DT_SPEC_GET(DT_ALIAS(led0), gpios);

/*============================================================
 * 廣播資料定義
 *============================================================*/

/*
 * 廣播封包資料（Advertising Data）
 * 最大 31 bytes，包含最重要的資訊
 *
 * 這裡我們放入：
 * 1. Flags（3 bytes）— BLE 規格要求，告知掃描者裝置的能力
 * 2. 不完整 128-bit UUID 列表 — 告知掃描者裝置提供的服務
 */
static const struct bt_data ad[] = {
    /*
     * AD Structure #1: Flags
     *
     * BT_DATA_FLAGS 的值組合：
     * - BT_LE_AD_GENERAL: 一般可發現模式（General Discoverable）
     *   表示裝置持續廣播，隨時可被發現
     * - BT_LE_AD_NO_BREDR: 不支援經典藍牙（BR/EDR）
     *   nRF54L15 僅支援 BLE，必須設定此旗標
     *
     * 佔用 3 bytes：Length(1) + Type(1) + Value(1)
     */
    BT_DATA_BYTES(BT_DATA_FLAGS,
                  (BT_LE_AD_GENERAL | BT_LE_AD_NO_BREDR)),

    /*
     * AD Structure #2: 不完整的 128-bit 服務 UUID 列表
     *
     * BT_DATA_UUID128_SOME 表示「這裡列出部分 UUID，可能還有更多」
     * 如果你只有一個服務，也可以用 BT_DATA_UUID128_ALL
     *
     * 佔用 18 bytes：Length(1) + Type(1) + UUID(16)
     *
     * 中央裝置（如手機）可以根據 UUID 過濾掃描結果，
     * 只顯示提供特定服務的裝置
     */
    BT_DATA_BYTES(BT_DATA_UUID128_SOME, CUSTOM_SERVICE_UUID),
};

/*
 * 掃描回應資料（Scan Response Data）
 * 也是最大 31 bytes，當掃描者主動發送 Scan Request 時回傳
 *
 * 掃描回應相當於「第二頁」資訊 — 掃描者需要主動詢問才會收到
 * 適合放置次要資訊，如完整裝置名稱、製造商資料等
 */

/*
 * 製造商特定資料（Manufacturer Specific Data）
 * 前 2 bytes 是公司 ID（little-endian），後面是自訂資料
 * 這裡我們放入一個版本號和溫度感測器讀數（範例值）
 */
static const uint8_t mfg_data[] = {
    MANUFACTURER_ID_LOW,    /* 公司 ID 低位元組 */
    MANUFACTURER_ID_HIGH,   /* 公司 ID 高位元組 */
    0x01,                   /* 自訂：資料格式版本 */
    0x00,                   /* 自訂：溫度整數部分（會動態更新） */
    0x00,                   /* 自訂：溫度小數部分（會動態更新） */
};

static struct bt_data sd[] = {
    /*
     * AD Structure #1: 完整裝置名稱
     *
     * BT_DATA_NAME_COMPLETE 包含完整裝置名稱
     * 名稱來自 CONFIG_BT_DEVICE_NAME（在 prj.conf 中設定）
     *
     * 注意：BT_DATA 巨集需要傳入資料指標和長度
     * 而裝置名稱可用 CONFIG_BT_DEVICE_NAME 取得
     */
    BT_DATA(BT_DATA_NAME_COMPLETE,
            CONFIG_BT_DEVICE_NAME,
            sizeof(CONFIG_BT_DEVICE_NAME) - 1),  /* -1 排除結尾的 \0 */

    /*
     * AD Structure #2: 製造商特定資料
     *
     * BT_DATA 巨集：(type, data_pointer, data_length)
     * 製造商特定資料沒有固定格式，完全自訂
     */
    BT_DATA(BT_DATA_MANUFACTURER_DATA,
            mfg_data,
            sizeof(mfg_data)),
};

/*============================================================
 * 廣播參數定義
 *============================================================*/

/*
 * 定義廣播參數
 *
 * BT_LE_ADV_PARAM_INIT() 巨集參數說明：
 *   參數 1: options — 廣播選項旗標組合
 *     BT_LE_ADV_OPT_CONNECTABLE — 允許連線
 *     BT_LE_ADV_OPT_SCANNABLE   — 允許掃描回應
 *     BT_LE_ADV_OPT_ONE_TIME    — 建立連線後自動停止廣播
 *   參數 2: interval_min — 最小廣播間隔
 *   參數 3: interval_max — 最大廣播間隔
 *   參數 4: peer — 定向廣播目標（NULL = 非定向）
 */
static struct bt_le_adv_param adv_param = BT_LE_ADV_PARAM_INIT(
    (BT_LE_ADV_OPT_CONNECTABLE |
     BT_LE_ADV_OPT_SCANNABLE |
     BT_LE_ADV_OPT_ONE_TIME),
    ADV_INT_MIN,    /* 60 ms */
    ADV_INT_MAX,    /* 100 ms */
    NULL            /* 非定向廣播 */
);

/*============================================================
 * 狀態追蹤
 *============================================================*/

static bool is_advertising;  /* 目前是否正在廣播 */
static bool is_connected;    /* 目前是否已連線 */

/*============================================================
 * 回呼函式
 *============================================================*/

/*
 * BLE 連線建立回呼
 *
 * 當中央裝置成功連線到本裝置時被呼叫
 * 參數：
 *   conn — 連線物件指標
 *   err  — 錯誤碼（0 = 成功）
 */
static void connected_cb(struct bt_conn *conn, uint8_t err)
{
    if (err) {
        LOG_ERR("連線失敗 (err %u)", err);
        /* 連線失敗，重新開始廣播 */
        is_advertising = false;
        return;
    }

    LOG_INF("已連線");
    is_connected = true;
    is_advertising = false;  /* ONE_TIME 模式下廣播已自動停止 */

    /* 關閉廣播指示 LED */
    gpio_pin_set_dt(&adv_led, 0);
}

/*
 * BLE 連線斷開回呼
 *
 * 當連線斷開時被呼叫
 * 參數：
 *   conn   — 連線物件指標
 *   reason — 斷開原因（HCI 錯誤碼）
 */
static void disconnected_cb(struct bt_conn *conn, uint8_t reason)
{
    LOG_INF("已斷線 (reason 0x%02x)", reason);
    is_connected = false;

    /* 斷線後自動重新開始廣播 */
    int ret = bt_le_adv_start(&adv_param, ad, ARRAY_SIZE(ad),
                               sd, ARRAY_SIZE(sd));
    if (ret) {
        LOG_ERR("重新啟動廣播失敗 (err %d)", ret);
        return;
    }

    LOG_INF("已重新開始廣播");
    is_advertising = true;
    gpio_pin_set_dt(&adv_led, 1);  /* 點亮 LED 表示正在廣播 */
}

/*
 * 連線回呼結構
 * 註冊到 Bluetooth 子系統，當連線事件發生時自動呼叫
 */
BT_CONN_CB_DEFINE(conn_callbacks) = {
    .connected = connected_cb,
    .disconnected = disconnected_cb,
};

/*============================================================
 * LED 初始化
 *============================================================*/

/*
 * 初始化廣播狀態指示 LED
 * 回傳：0 = 成功，負值 = 錯誤
 */
static int init_led(void)
{
    int ret;

    /* 檢查 LED GPIO 裝置是否就緒 */
    if (!gpio_is_ready_dt(&adv_led)) {
        LOG_ERR("LED GPIO 裝置未就緒");
        return -ENODEV;
    }

    /* 設定 LED GPIO 為輸出，初始狀態為關閉 */
    ret = gpio_pin_configure_dt(&adv_led, GPIO_OUTPUT_INACTIVE);
    if (ret < 0) {
        LOG_ERR("LED GPIO 設定失敗 (err %d)", ret);
        return ret;
    }

    return 0;
}

/*============================================================
 * 廣播控制函式
 *============================================================*/

/*
 * 啟動 BLE 廣播
 *
 * 此函式會：
 * 1. 呼叫 bt_le_adv_start() 開始廣播
 * 2. 點亮 LED 指示廣播中
 *
 * 回傳：0 = 成功，負值 = 錯誤
 */
static int start_advertising(void)
{
    int ret;

    /*
     * bt_le_adv_start() — 開始 BLE 廣播
     *
     * 參數說明：
     *   param    — 廣播參數（間隔、選項等）
     *   ad       — 廣播資料陣列
     *   ad_len   — 廣播資料元素數量
     *   sd       — 掃描回應資料陣列（NULL = 無掃描回應）
     *   sd_len   — 掃描回應資料元素數量
     *
     * 回傳值：
     *   0       — 成功
     *   -EALREADY — 已在廣播中
     *   -ENOMEM   — 記憶體不足
     *   -EINVAL   — 無效參數
     *   -ENODEV   — 藍牙未啟用
     */
    ret = bt_le_adv_start(&adv_param, ad, ARRAY_SIZE(ad),
                           sd, ARRAY_SIZE(sd));
    if (ret) {
        LOG_ERR("啟動廣播失敗 (err %d)", ret);
        return ret;
    }

    is_advertising = true;
    gpio_pin_set_dt(&adv_led, 1);  /* 點亮 LED */
    LOG_INF("BLE 廣播已啟動");
    LOG_INF("  間隔: %d ~ %d ms",
            (ADV_INT_MIN * 625) / 1000,
            (ADV_INT_MAX * 625) / 1000);

    return 0;
}

/*
 * 停止 BLE 廣播
 *
 * 回傳：0 = 成功，負值 = 錯誤
 */
static int stop_advertising(void)
{
    int ret;

    /*
     * bt_le_adv_stop() — 停止 BLE 廣播
     *
     * 回傳值：
     *   0       — 成功
     *   -EALREADY — 已經停止
     *   -ENODEV   — 藍牙未啟用
     */
    ret = bt_le_adv_stop();
    if (ret) {
        LOG_ERR("停止廣播失敗 (err %d)", ret);
        return ret;
    }

    is_advertising = false;
    gpio_pin_set_dt(&adv_led, 0);  /* 關閉 LED */
    LOG_INF("BLE 廣播已停止");

    return 0;
}

/*============================================================
 * 主函式
 *============================================================*/

int main(void)
{
    int ret;

    LOG_INF("=== nRF54L15 BLE 廣播範例 ===");

    /* 步驟 1：初始化 LED */
    ret = init_led();
    if (ret) {
        LOG_ERR("LED 初始化失敗");
        /* LED 初始化失敗不影響核心功能，繼續執行 */
    }

    /*
     * 步驟 2：啟用 Bluetooth 子系統
     *
     * bt_enable() 初始化整個 BLE 協定堆疊：
     *   - HCI 傳輸層
     *   - Link Layer Controller
     *   - Host 層（GAP、GATT 等）
     *
     * 參數：回呼函式（NULL = 同步等待完成）
     * 回傳：0 = 成功
     *
     * 注意：此函式必須在使用任何其他 bt_* API 之前呼叫
     */
    ret = bt_enable(NULL);
    if (ret) {
        LOG_ERR("Bluetooth 初始化失敗 (err %d)", ret);
        return ret;
    }
    LOG_INF("Bluetooth 已初始化");

    /* 步驟 3：啟動廣播 */
    ret = start_advertising();
    if (ret) {
        LOG_ERR("無法啟動廣播");
        return ret;
    }

    /*
     * 步驟 4：主迴圈
     *
     * 在真實應用中，主迴圈可以：
     * - 更新廣播資料（如感測器數值）
     * - 處理按鈕輸入
     * - 進入低功耗睡眠
     *
     * 這裡我們簡單地每 5 秒印出一次狀態
     */
    while (1) {
        k_sleep(K_SECONDS(5));

        if (is_advertising) {
            LOG_INF("狀態：廣播中...");
        } else if (is_connected) {
            LOG_INF("狀態：已連線");
        } else {
            LOG_INF("狀態：閒置");
        }
    }

    return 0;
}
```

### 檔案：`CMakeLists.txt`

```cmake
# SPDX-License-Identifier: Apache-2.0
# BLE 廣播範例專案

cmake_minimum_required(VERSION 3.20.0)

# 載入 Zephyr 建置系統
find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})

# 定義專案名稱
project(ble_advertising_demo)

# 加入原始碼檔案
target_sources(app PRIVATE src/main.c)
```

### 編譯與燒錄

```bash
# 方法 1：使用 west 命令列工具
# 從專案根目錄執行

# 編譯（指定目標板）
west build -b nrf54l15dk/nrf54l15/cpuapp

# 燒錄到開發板
west flash

# 查看序列埠輸出（使用 JLink RTT 或 UART）
# 方法 A：使用 JLink RTT Viewer
JLinkRTTViewer

# 方法 B：使用 minicom 連接 UART
minicom -D /dev/ttyACM0 -b 115200

# 方法 2：清除後重新編譯（修改 Kconfig 後建議使用）
west build -b nrf54l15dk/nrf54l15/cpuapp --pristine
west flash
```

---

## 7. 進階用法

### 7.1 動態更新廣播資料

廣播資料可以在執行期間動態更新，例如定期更新感測器讀數：

```c
#include <zephyr/bluetooth/bluetooth.h>
#include <zephyr/logging/log.h>

LOG_MODULE_DECLARE(ble_adv, LOG_LEVEL_INF);

/*
 * 動態更新廣播資料
 *
 * 步驟：
 * 1. 修改資料緩衝區內容
 * 2. 呼叫 bt_le_adv_update_data() 通知協定堆疊
 *
 * 注意：bt_le_adv_update_data() 會在下一次廣播事件生效
 */
static uint8_t mfg_data_dynamic[] = {
    0xFF, 0xFF,  /* 公司 ID */
    0x01,        /* 格式版本 */
    0x00,        /* 溫度整數 */
    0x00,        /* 溫度小數 */
    0x00,        /* 電池電壓（×10） */
};

static const struct bt_data ad_dynamic[] = {
    BT_DATA_BYTES(BT_DATA_FLAGS,
                  (BT_LE_AD_GENERAL | BT_LE_AD_NO_BREDR)),
    BT_DATA(BT_DATA_MANUFACTURER_DATA,
            mfg_data_dynamic, sizeof(mfg_data_dynamic)),
};

/*
 * 更新廣播中的感測器資料
 *
 * 參數：
 *   temperature — 溫度值（×100，如 2350 = 23.50°C）
 *   battery_mv  — 電池電壓（mV）
 */
int update_adv_sensor_data(int16_t temperature, uint16_t battery_mv)
{
    /* 更新製造商資料緩衝區 */
    mfg_data_dynamic[3] = (uint8_t)(temperature / 100);      /* 整數部分 */
    mfg_data_dynamic[4] = (uint8_t)(temperature % 100);      /* 小數部分 */
    mfg_data_dynamic[5] = (uint8_t)(battery_mv / 100);       /* 電壓（×10） */

    /*
     * bt_le_adv_update_data() — 更新廣播資料
     *
     * 參數與 bt_le_adv_start() 的 ad/sd 部分相同
     * 只能在廣播進行中呼叫
     *
     * 回傳：0 = 成功
     */
    int ret = bt_le_adv_update_data(ad_dynamic, ARRAY_SIZE(ad_dynamic),
                                     NULL, 0);
    if (ret) {
        LOG_ERR("更新廣播資料失敗 (err %d)", ret);
    }

    return ret;
}
```

### 7.2 Extended Advertising（擴展廣播）

nRF54L15 支援 Bluetooth 5.x 的 Extended Advertising，突破 31 bytes 限制：

```c
#include <zephyr/bluetooth/bluetooth.h>
#include <zephyr/logging/log.h>

LOG_MODULE_DECLARE(ble_adv, LOG_LEVEL_INF);

/*
 * Extended Advertising 範例
 *
 * 優勢：
 * - 廣播資料可超過 31 bytes（最高 1650 bytes）
 * - 支援多個廣播集合同時運行
 * - 支援 Coded PHY（長距離模式）
 * - 支援週期性廣播
 *
 * 需要的額外 Kconfig：
 *   CONFIG_BT_EXT_ADV=y
 *   CONFIG_BT_EXT_ADV_MAX_ADV_SET=2
 *   CONFIG_BT_CTLR_ADV_DATA_LEN_MAX=251
 */

/* Extended Advertising 的大型資料負載 */
static const uint8_t ext_adv_payload[] = {
    /* 這裡可以放超過 31 bytes 的資料 */
    0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08,
    0x09, 0x0A, 0x0B, 0x0C, 0x0D, 0x0E, 0x0F, 0x10,
    0x11, 0x12, 0x13, 0x14, 0x15, 0x16, 0x17, 0x18,
    0x19, 0x1A, 0x1B, 0x1C, 0x1D, 0x1E, 0x1F, 0x20,
    0x21, 0x22, 0x23, 0x24,  /* 已超過 31 bytes！ */
};

static const struct bt_data ext_ad[] = {
    BT_DATA_BYTES(BT_DATA_FLAGS,
                  (BT_LE_AD_GENERAL | BT_LE_AD_NO_BREDR)),
    BT_DATA(BT_DATA_MANUFACTURER_DATA,
            ext_adv_payload, sizeof(ext_adv_payload)),
};

static struct bt_le_ext_adv *ext_adv_set;

/*
 * Extended Advertising 回呼：廣播已發送
 */
static void ext_adv_sent_cb(struct bt_le_ext_adv *adv,
                             struct bt_le_ext_adv_sent_info *info)
{
    LOG_INF("Extended Advertising 事件已發送 (%u)", info->num_sent);
}

/*
 * Extended Advertising 回呼：收到連線
 */
static void ext_adv_connected_cb(struct bt_le_ext_adv *adv,
                                  struct bt_le_ext_adv_connected_info *info)
{
    LOG_INF("透過 Extended Advertising 收到連線");
}

/* 回呼結構 */
static const struct bt_le_ext_adv_cb ext_adv_callbacks = {
    .sent = ext_adv_sent_cb,
    .connected = ext_adv_connected_cb,
};

/*
 * 啟動 Extended Advertising
 */
int start_extended_advertising(void)
{
    int ret;

    /* 建立 Extended Advertising 參數 */
    struct bt_le_adv_param ext_adv_param = BT_LE_ADV_PARAM_INIT(
        (BT_LE_ADV_OPT_CONNECTABLE | BT_LE_ADV_OPT_EXT_ADV),
        BT_GAP_ADV_FAST_INT_MIN_2,
        BT_GAP_ADV_FAST_INT_MAX_2,
        NULL
    );

    /*
     * bt_le_ext_adv_create() — 建立 Extended Advertising 集合
     *
     * 參數：
     *   param — 廣播參數
     *   cb    — 事件回呼結構
     *   adv   — 輸出：建立的廣播集合指標
     */
    ret = bt_le_ext_adv_create(&ext_adv_param, &ext_adv_callbacks,
                                &ext_adv_set);
    if (ret) {
        LOG_ERR("建立 Extended Advertising 失敗 (err %d)", ret);
        return ret;
    }

    /*
     * bt_le_ext_adv_set_data() — 設定 Extended Advertising 資料
     *
     * 參數：
     *   adv    — 廣播集合指標
     *   ad     — 廣播資料
     *   ad_len — 廣播資料數量
     *   sd     — 掃描回應資料
     *   sd_len — 掃描回應資料數量
     */
    ret = bt_le_ext_adv_set_data(ext_adv_set,
                                  ext_ad, ARRAY_SIZE(ext_ad),
                                  NULL, 0);
    if (ret) {
        LOG_ERR("設定 Extended Advertising 資料失敗 (err %d)", ret);
        return ret;
    }

    /*
     * bt_le_ext_adv_start() — 開始 Extended Advertising
     *
     * BT_LE_EXT_ADV_START_DEFAULT 使用預設參數：
     *   timeout = 0（無限廣播）
     *   num_events = 0（無限事件）
     */
    ret = bt_le_ext_adv_start(ext_adv_set,
                               BT_LE_EXT_ADV_START_DEFAULT);
    if (ret) {
        LOG_ERR("啟動 Extended Advertising 失敗 (err %d)", ret);
        return ret;
    }

    LOG_INF("Extended Advertising 已啟動");
    return 0;
}
```

### 7.3 調整 TX Power（發射功率）

```c
#include <zephyr/bluetooth/hci.h>
#include <zephyr/bluetooth/hci_vs.h>

/*
 * 設定廣播發射功率
 *
 * nRF54L15 支援的 TX Power 等級：
 *   -40, -20, -16, -12, -8, -4, 0, +2, +3, +4, +7, +8 dBm
 *
 * 較低功率 → 省電但範圍小
 * 較高功率 → 範圍大但耗電
 *
 * 參數：
 *   tx_power_dbm — 目標功率（dBm）
 *                  控制器會選擇最接近的支援值
 */
int set_adv_tx_power(int8_t tx_power_dbm)
{
    struct bt_hci_cp_vs_write_tx_power_level *cp;
    struct bt_hci_rp_vs_write_tx_power_level *rp;
    struct net_buf *buf;
    struct net_buf *rsp = NULL;
    int ret;

    /* 分配 HCI 命令緩衝區 */
    buf = bt_hci_cmd_create(BT_HCI_OP_VS_WRITE_TX_POWER_LEVEL,
                             sizeof(*cp));
    if (!buf) {
        return -ENOMEM;
    }

    cp = net_buf_add(buf, sizeof(*cp));
    cp->handle_type = BT_HCI_VS_LL_HANDLE_TYPE_ADV;  /* 廣播用 */
    cp->handle = 0;                                    /* 廣播 handle */
    cp->tx_power_level = tx_power_dbm;

    /* 發送 Vendor Specific HCI 命令 */
    ret = bt_hci_cmd_send_sync(BT_HCI_OP_VS_WRITE_TX_POWER_LEVEL,
                                buf, &rsp);
    if (ret) {
        return ret;
    }

    /* 讀取控制器實際設定的功率值 */
    rp = (void *)rsp->data;
    LOG_INF("TX Power 已設定為 %d dBm（要求 %d dBm）",
            rp->selected_tx_power, tx_power_dbm);

    net_buf_unref(rsp);
    return 0;
}
```

### 7.4 使用隱私模式（Resolvable Private Address）

```c
/*
 * 使用 RPA（Resolvable Private Address）增強隱私
 *
 * 問題：固定的 BLE 地址可被追蹤
 * 解決：使用 RPA，地址每 15 分鐘自動更換
 *
 * 需要的 Kconfig：
 *   CONFIG_BT_PRIVACY=y
 *   CONFIG_BT_RPA_TIMEOUT=900   # RPA 更換間隔（秒），預設 900
 *
 * 啟用後，只有已配對的裝置能辨識你的裝置
 * 未配對的掃描者每次看到的地址都不同
 */

/* 使用隱私模式的廣播參數 — 不要設 USE_IDENTITY */
static struct bt_le_adv_param privacy_adv_param = BT_LE_ADV_PARAM_INIT(
    (BT_LE_ADV_OPT_CONNECTABLE | BT_LE_ADV_OPT_SCANNABLE),
    BT_GAP_ADV_FAST_INT_MIN_2,
    BT_GAP_ADV_FAST_INT_MAX_2,
    NULL
);
```

### 7.5 廣播間隔動態調整策略

在實際產品中，常見的策略是快速廣播一段時間後切換到慢速廣播以省電：

```c
#include <zephyr/kernel.h>
#include <zephyr/bluetooth/bluetooth.h>
#include <zephyr/logging/log.h>

LOG_MODULE_DECLARE(ble_adv, LOG_LEVEL_INF);

/* 快速廣播參數：30 ms 間隔，維持 30 秒 */
#define FAST_ADV_INT_MIN    0x0030  /* 30 ms */
#define FAST_ADV_INT_MAX    0x0060  /* 60 ms */
#define FAST_ADV_DURATION_SEC 30

/* 慢速廣播參數：1000 ms 間隔，無限期 */
#define SLOW_ADV_INT_MIN    BT_GAP_ADV_SLOW_INT_MIN  /* 1000 ms */
#define SLOW_ADV_INT_MAX    BT_GAP_ADV_SLOW_INT_MAX  /* 1200 ms */

static struct bt_le_adv_param fast_adv_param = BT_LE_ADV_PARAM_INIT(
    (BT_LE_ADV_OPT_CONNECTABLE | BT_LE_ADV_OPT_SCANNABLE),
    FAST_ADV_INT_MIN, FAST_ADV_INT_MAX, NULL
);

static struct bt_le_adv_param slow_adv_param = BT_LE_ADV_PARAM_INIT(
    (BT_LE_ADV_OPT_CONNECTABLE | BT_LE_ADV_OPT_SCANNABLE),
    SLOW_ADV_INT_MIN, SLOW_ADV_INT_MAX, NULL
);

/* 計時器回呼：切換到慢速廣播 */
static void switch_to_slow_adv(struct k_work *work)
{
    int ret;

    /* 先停止快速廣播 */
    bt_le_adv_stop();

    /* 啟動慢速廣播 */
    ret = bt_le_adv_start(&slow_adv_param, ad, ARRAY_SIZE(ad),
                           sd, ARRAY_SIZE(sd));
    if (ret) {
        LOG_ERR("切換慢速廣播失敗 (err %d)", ret);
        return;
    }

    LOG_INF("已切換到慢速廣播 (1000-1200 ms 間隔)");
}

K_WORK_DEFINE(slow_adv_work, switch_to_slow_adv);

/* 延遲工作項：30 秒後觸發 */
static void slow_adv_timer_handler(struct k_timer *timer)
{
    k_work_submit(&slow_adv_work);
}

K_TIMER_DEFINE(slow_adv_timer, slow_adv_timer_handler, NULL);

/*
 * 啟動雙速廣播策略
 * 先快速廣播 30 秒，然後自動切換到慢速
 */
int start_dual_speed_advertising(void)
{
    int ret;

    /* 啟動快速廣播 */
    ret = bt_le_adv_start(&fast_adv_param, ad, ARRAY_SIZE(ad),
                           sd, ARRAY_SIZE(sd));
    if (ret) {
        LOG_ERR("啟動快速廣播失敗 (err %d)", ret);
        return ret;
    }

    LOG_INF("快速廣播已啟動 (30-60 ms 間隔)，%d 秒後切換慢速",
            FAST_ADV_DURATION_SEC);

    /* 設定計時器，30 秒後切換 */
    k_timer_start(&slow_adv_timer, K_SECONDS(FAST_ADV_DURATION_SEC),
                  K_NO_WAIT);

    return 0;
}
```

---

## 8. 常見問題與除錯

### FAQ

#### Q1: 廣播啟動後手機掃描不到裝置？

**排查步驟：**

1. **確認廣播確實在運行**
   ```c
   /* 檢查 bt_le_adv_start() 回傳值 */
   int ret = bt_le_adv_start(...);
   printk("adv_start 回傳: %d\n", ret);  /* 應為 0 */
   ```

2. **確認手機 BLE 功能開啟**
   - iOS：設定 → 藍牙 → 開啟
   - Android：設定 → 藍牙 → 開啟，並確認位置權限已授予

3. **使用 nRF Connect App 掃描**
   - Nordic 的 nRF Connect 手機 App 是最佳除錯工具
   - 可以看到完整的廣播資料、RSSI、裝置名稱

4. **確認沒有使用身份地址濾除**
   ```
   # 如果啟用了 CONFIG_BT_PRIVACY=y，
   # 手機可能因為 RPA 地址變化而顯示為不同裝置
   ```

5. **檢查 TX Power 是否太低**
   - 預設 0 dBm 在大多數環境足夠
   - 如果距離遠，嘗試提高到 +4 或 +8 dBm

#### Q2: `bt_le_adv_start()` 回傳 `-ENOMEM`？

**原因**：Bluetooth Controller 的緩衝區不足

**解決方案**：在 `prj.conf` 中增加緩衝區：
```kconfig
CONFIG_BT_BUF_ACL_RX_SIZE=73
CONFIG_BT_BUF_ACL_TX_SIZE=73
CONFIG_BT_BUF_ACL_TX_COUNT=7
CONFIG_BT_BUF_CMD_TX_SIZE=65
```

#### Q3: 廣播資料超過 31 bytes 怎麼辦？

**解決方案 1**：精簡廣播資料
- 把次要資訊移到掃描回應（Scan Response）
- 使用短名稱（`BT_DATA_NAME_SHORTENED`）代替完整名稱
- 減少 UUID 數量

**解決方案 2**：使用 Extended Advertising（見 7.2 節）
```kconfig
CONFIG_BT_EXT_ADV=y
CONFIG_BT_CTLR_ADV_DATA_LEN_MAX=251
```

#### Q4: 如何確認廣播封包的實際內容？

**方法 1**：使用 nRF Connect App（手機）
- 掃描到裝置後點擊「RAW」查看原始資料

**方法 2**：使用 nRF Sniffer（需要額外硬體）
- 用另一塊 nRF DK 板刷入 Sniffer 韌體
- 搭配 Wireshark 完整擷取空中封包

**方法 3**：使用 Zephyr Shell（開發板上）
```kconfig
# 在 prj.conf 中啟用 BT Shell
CONFIG_BT_SHELL=y
CONFIG_SHELL=y
```
```
# 在 shell 中執行：
uart:~$ bt adv-info
```

#### Q5: 連線後廣播不會自動停止？

確認使用了 `BT_LE_ADV_OPT_ONE_TIME` 旗標：
```c
struct bt_le_adv_param param = BT_LE_ADV_PARAM_INIT(
    (BT_LE_ADV_OPT_CONNECTABLE | BT_LE_ADV_OPT_ONE_TIME),
    ...
);
```
沒有此旗標時，控制器會在連線後繼續廣播，允許更多裝置連線（多連線模式）。

#### Q6: 如何同時支援多個連線？

```kconfig
# prj.conf
CONFIG_BT_MAX_CONN=3  # 最多 3 個同時連線

# 不要使用 BT_LE_ADV_OPT_ONE_TIME
# 這樣每次有連線建立後，廣播仍會繼續
```

### 常見錯誤訊息對照

| 錯誤碼 | 意義 | 常見原因 |
|--------|------|----------|
| `-EAGAIN` (-11) | 資源暫時不可用 | BT 子系統尚未就緒，等一下再試 |
| `-EALREADY` (-114) | 已在執行 | 重複呼叫 `bt_le_adv_start()` |
| `-EINVAL` (-22) | 無效參數 | 廣播間隔超出範圍或資料格式錯誤 |
| `-ENOMEM` (-12) | 記憶體不足 | 緩衝區設定太小 |
| `-ENODEV` (-19) | 裝置不存在 | `bt_enable()` 尚未呼叫 |
| `-ENOTCONN` (-128) | 未連線 | 嘗試在非連線狀態執行連線操作 |

### 除錯技巧

**啟用詳細 BLE 日誌：**
```kconfig
# prj.conf — 開發除錯專用
CONFIG_BT_LOG_LEVEL_DBG=y
CONFIG_BT_HCI_DRIVER_LOG_LEVEL_DBG=y
CONFIG_LOG=y
CONFIG_LOG_DEFAULT_LEVEL=4
CONFIG_LOG_BUFFER_SIZE=4096
```

**使用 `bt_addr_le_to_str()` 印出地址：**
```c
#include <zephyr/bluetooth/bluetooth.h>

/* 印出本機 BLE 地址 */
void print_local_address(void)
{
    bt_addr_le_t addr;
    size_t count = 1;
    char addr_str[BT_ADDR_LE_STR_LEN];

    bt_id_get(&addr, &count);
    bt_addr_le_to_str(&addr, addr_str, sizeof(addr_str));
    printk("本機地址: %s\n", addr_str);
}
```

---

## 9. API 快速參考

### 核心廣播 API

| 函式 | 說明 | 回傳 |
|------|------|------|
| `bt_enable(bt_ready_cb_t cb)` | 初始化 BLE 子系統。`cb=NULL` 為同步模式 | 0=成功 |
| `bt_le_adv_start(const struct bt_le_adv_param *param, const struct bt_data *ad, size_t ad_len, const struct bt_data *sd, size_t sd_len)` | 啟動 Legacy Advertising | 0=成功 |
| `bt_le_adv_stop(void)` | 停止廣播 | 0=成功 |
| `bt_le_adv_update_data(const struct bt_data *ad, size_t ad_len, const struct bt_data *sd, size_t sd_len)` | 更新進行中的廣播資料 | 0=成功 |
| `bt_set_name(const char *name)` | 動態修改裝置名稱（需 `CONFIG_BT_DEVICE_NAME_DYNAMIC`） | 0=成功 |
| `bt_get_name(void)` | 取得目前裝置名稱 | `const char*` |

### Extended Advertising API

| 函式 | 說明 | 回傳 |
|------|------|------|
| `bt_le_ext_adv_create(const struct bt_le_adv_param *param, const struct bt_le_ext_adv_cb *cb, struct bt_le_ext_adv **adv)` | 建立 Extended Adv 集合 | 0=成功 |
| `bt_le_ext_adv_start(struct bt_le_ext_adv *adv, const struct bt_le_ext_adv_start_param *param)` | 啟動 Extended Adv | 0=成功 |
| `bt_le_ext_adv_stop(struct bt_le_ext_adv *adv)` | 停止 Extended Adv | 0=成功 |
| `bt_le_ext_adv_set_data(struct bt_le_ext_adv *adv, const struct bt_data *ad, size_t ad_len, const struct bt_data *sd, size_t sd_len)` | 設定/更新 Extended Adv 資料 | 0=成功 |
| `bt_le_ext_adv_delete(struct bt_le_ext_adv *adv)` | 刪除 Extended Adv 集合 | 0=成功 |

### 資料建構巨集

| 巨集 | 說明 | 範例 |
|------|------|------|
| `BT_DATA_BYTES(type, ...)` | 用位元組值建構 `bt_data` | `BT_DATA_BYTES(BT_DATA_FLAGS, 0x06)` |
| `BT_DATA(type, data, len)` | 用指標和長度建構 `bt_data` | `BT_DATA(BT_DATA_NAME_COMPLETE, name, strlen(name))` |
| `BT_LE_ADV_PARAM_INIT(opt, min, max, peer)` | 初始化廣播參數結構 | 見程式碼範例 |
| `BT_LE_ADV_CONN` | 預定義：可連線可掃描廣播 | `bt_le_adv_start(BT_LE_ADV_CONN, ...)` |
| `BT_LE_ADV_NCONN` | 預定義：不可連線不可掃描 | 純 Beacon 模式 |

### 常用 AD Type 常數

| 常數 | 值 | 說明 |
|------|-----|------|
| `BT_DATA_FLAGS` | 0x01 | 廣播旗標（必填） |
| `BT_DATA_UUID16_SOME` | 0x02 | 不完整 16-bit UUID 列表 |
| `BT_DATA_UUID16_ALL` | 0x03 | 完整 16-bit UUID 列表 |
| `BT_DATA_UUID128_SOME` | 0x06 | 不完整 128-bit UUID 列表 |
| `BT_DATA_UUID128_ALL` | 0x07 | 完整 128-bit UUID 列表 |
| `BT_DATA_NAME_SHORTENED` | 0x08 | 縮短裝置名稱 |
| `BT_DATA_NAME_COMPLETE` | 0x09 | 完整裝置名稱 |
| `BT_DATA_TX_POWER` | 0x0A | TX Power Level |
| `BT_DATA_MANUFACTURER_DATA` | 0xFF | 製造商特定資料 |
| `BT_DATA_SVC_DATA16` | 0x16 | 16-bit UUID 服務資料 |

---

## 10. 延伸閱讀

### Nordic 官方文件

- [nRF54L15 Product Specification](https://docs.nordicsemi.com/bundle/ps_nrf54L15/page/keyfeatures_html5.html) — 晶片完整規格
- [nRF Connect SDK: Bluetooth LE Advertising](https://docs.nordicsemi.com/bundle/ncs-latest/page/nrf/protocols/bt/ble/index.html) — SDK 廣播指南
- [Zephyr Bluetooth API Reference](https://docs.zephyrproject.org/latest/connectivity/bluetooth/api/index.html) — Zephyr BLE API 文件

### Zephyr 範例專案

以下範例位於 nRF Connect SDK 安裝目錄中：

| 路徑 | 說明 |
|------|------|
| `zephyr/samples/bluetooth/beacon/` | 簡單 Beacon 範例（不可連線廣播） |
| `zephyr/samples/bluetooth/peripheral/` | BLE Peripheral 範例（可連線廣播 + GATT） |
| `zephyr/samples/bluetooth/peripheral_hr/` | 心率感測器 Peripheral（含服務定義） |
| `nrf/samples/bluetooth/peripheral_lbs/` | Nordic LED Button Service 範例 |
| `zephyr/samples/bluetooth/extended_adv/` | Extended Advertising 範例 |

### Bluetooth 規格文件

- [Bluetooth Core Specification v5.4](https://www.bluetooth.com/specifications/specs/core-specification-5-4/) — 完整藍牙核心規格
- [Bluetooth Assigned Numbers](https://www.bluetooth.com/specifications/assigned-numbers/) — AD Type、UUID、Company ID 等編號查詢
- [GATT Services](https://www.bluetooth.com/specifications/gatt/services/) — 標準 GATT 服務列表

### 相關工具

| 工具 | 說明 |
|------|------|
| [nRF Connect for Mobile](https://www.nordicsemi.com/Products/Development-tools/nRF-Connect-for-mobile) | 手機端 BLE 除錯工具 |
| [nRF Sniffer for Bluetooth LE](https://www.nordicsemi.com/Products/Development-tools/nRF-Sniffer-for-Bluetooth-LE) | BLE 空中封包擷取工具 |
| [Wireshark](https://www.wireshark.org/) | 搭配 nRF Sniffer 分析封包 |
| [nRF Connect for Desktop](https://www.nordicsemi.com/Products/Development-tools/nRF-Connect-for-Desktop) | 桌面端 BLE 工具套件 |
