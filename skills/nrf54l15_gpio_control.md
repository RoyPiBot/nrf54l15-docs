---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: peripheral
difficulty: beginner
keywords: [GPIO, input, output, interrupt, pull-up, pull-down, callback, devicetree, nrf54l15, digital-io]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-03-31
---

# GPIO 基礎操作：輸入輸出、中斷、上下拉電阻設定

## 1. 概述

### 1.1 什麼是 GPIO

GPIO（General-Purpose Input/Output，通用輸入輸出）是微控制器最基本也最重要的周邊介面。每一支 GPIO 腳位都可以被軟體設定為：

- **數位輸出**：控制 LED、繼電器、馬達驅動器等
- **數位輸入**：讀取按鈕、開關、感測器的高低電位
- **中斷來源**：當腳位電位發生變化時，觸發 CPU 中斷，實現事件驅動架構

### 1.2 nRF54L15 的 GPIO 資源

nRF54L15 提供以下 GPIO 資源：

| 項目 | 規格 |
|------|------|
| GPIO Port 0 | 最多 5 支腳位（P0.00–P0.04） |
| GPIO Port 1 | 最多 15 支腳位（P1.00–P1.14） |
| GPIO Port 2 | 最多 11 支腳位（P2.00–P2.10） |
| 輸出驅動模式 | 標準推挽（push-pull）、開漏（open-drain） |
| 內建上/下拉電阻 | 每支腳位皆可個別設定 |
| 中斷觸發模式 | 上升沿、下降沿、雙沿、電位準位 |
| 電壓範圍 | 1.8V–3.6V（視供電設定） |

### 1.3 適用場景

- 點亮 / 閃爍 LED
- 讀取按鈕或機械開關
- 控制外部電路（致能訊號、晶片選擇）
- 產生軟體 PWM（低頻場景）
- 連接簡單的數位感測器（如 DHT11 的 data pin）
- 作為除錯用的觀測點（toggle pin + 邏輯分析儀）

### 1.4 在 nRF54L15 DK 上的實體對應

nRF54L15 DK 開發板上已經連接了以下 GPIO 裝置：

| 裝置 | DK 標記 | GPIO 腳位 | Devicetree 別名 |
|------|---------|-----------|-----------------|
| LED 0 | LED0 | P2.09 | `led0` |
| LED 1 | LED1 | P1.10 | `led1` |
| LED 2 | LED2 | P2.07 | `led2` |
| LED 3 | LED3 | P1.14 | `led3` |
| 按鈕 0 | Button 0 | P1.13 | `sw0` |
| 按鈕 1 | Button 1 | P1.09 | `sw1` |
| 按鈕 2 | Button 2 | P1.08 | `sw2` |
| 按鈕 3 | Button 3 | P0.04 | `sw3` |

> **注意**：LED 為低電位有效（active-low），按鈕按下時接地（active-low，需上拉電阻）。

---

## 2. 硬體概念

### 2.1 GPIO 腳位內部架構

每一支 GPIO 腳位內部的簡化電路如下：

```
              VDD
               │
              ┌┤ 上拉電阻（可選）
              │┤ （~13kΩ）
              └┤
               │
外部腳位 ──────┤──── 輸入緩衝器 ──→ 輸入暫存器（讀取值）
               │
               ├──── 輸出驅動器 ←── 輸出暫存器（寫入值）
               │     ├─ P-MOSFET（高側）
               │     └─ N-MOSFET（低側）
               │
              ┌┤ 下拉電阻（可選）
              │┤ （~13kΩ）
              └┤
               │
              GND
```

### 2.2 輸出驅動模式

| 模式 | 說明 | 典型應用 |
|------|------|----------|
| **Push-Pull（推挽）** | 高側與低側 MOSFET 交替導通，可主動驅動高電位與低電位 | LED 驅動、數位訊號輸出 |
| **Open-Drain（開漏）** | 只有低側 MOSFET，高電位靠外部上拉 | I²C 匯流排、多裝置共用線路 |

### 2.3 上拉與下拉電阻

當腳位設為輸入模式時，如果外部沒有提供確定的電位，腳位會處於「浮空」（floating）狀態，讀取值不確定。此時需要上拉或下拉電阻：

- **上拉電阻（Pull-Up）**：將腳位預設拉到高電位（VDD）。適合 active-low 按鈕（按下接地）
- **下拉電阻（Pull-Down）**：將腳位預設拉到低電位（GND）。適合 active-high 按鈕（按下接 VDD）
- **無（Disabled）**：浮空狀態，外部電路必須提供確定電位

nRF54L15 內建的上/下拉電阻阻值約為 13kΩ（典型值），可透過軟體個別啟用。

### 2.4 中斷機制

nRF54L15 的 GPIO 中斷由 GPIOTE（GPIO Tasks and Events）模組處理。GPIOTE 可以監控指定腳位的電位變化，當條件滿足時產生中斷事件。

支援的觸發條件：

| 觸發模式 | 說明 |
|----------|------|
| **上升沿（Rising Edge）** | 電位從低變高時觸發 |
| **下降沿（Falling Edge）** | 電位從高變低時觸發 |
| **雙沿（Both Edges）** | 電位變化時都觸發 |
| **電位準位高（Level High）** | 電位為高時持續觸發 |
| **電位準位低（Level Low）** | 電位為低時持續觸發 |

### 2.5 電氣規格注意事項

- nRF54L15 的 GPIO 不具備 5V 容忍（**NOT 5V tolerant**）。絕對最大輸入電壓為 VDD + 0.3V
- 每支腳位最大灌電流 / 源電流約為 6mA（高驅動模式可達更高，但需查閱 datasheet）
- 切換頻率受限於 slew rate 設定

---

## 3. 軟體架構

### 3.1 Zephyr GPIO 驅動架構概覽

```
┌─────────────────────────────┐
│       應用程式碼             │  ← 你寫的 main.c
├─────────────────────────────┤
│    Zephyr GPIO API          │  ← gpio.h 提供的標準 API
│    (gpio_pin_configure,     │
│     gpio_pin_set, ...)      │
├─────────────────────────────┤
│    GPIO Driver              │  ← Nordic 提供的驅動實作
│    (gpio_nrfx.c)            │
├─────────────────────────────┤
│    nrfx HAL / MDK           │  ← Nordic 硬體抽象層
├─────────────────────────────┤
│    nRF54L15 硬體暫存器       │  ← 實際的 GPIO 周邊
└─────────────────────────────┘
```

### 3.2 關鍵標頭檔

```c
#include <zephyr/drivers/gpio.h>    /* GPIO API：設定、讀寫、中斷 */
#include <zephyr/kernel.h>          /* 核心 API：延遲、執行緒、信號量 */
#include <zephyr/devicetree.h>      /* Devicetree 巨集 */
```

### 3.3 Devicetree 與 GPIO 的關係

在 Zephyr 中，GPIO 控制器和腳位的定義來自 Devicetree。你不會直接寫暫存器位址，而是：

1. 在 Devicetree（`.dts` / `.overlay`）中宣告腳位
2. 在 C 程式碼中使用 `GPIO_DT_SPEC_GET()` 巨集取得腳位資訊
3. 使用 `gpio_pin_configure_dt()` 等 `_dt` 後綴的 API 操作腳位

這個流程確保了硬體設定與軟體邏輯分離，方便移植。

### 3.4 GPIO 旗標（Flags）組合

Zephyr 使用位元旗標來描述 GPIO 腳位的行為。以下是常用旗標：

| 旗標 | 說明 |
|------|------|
| `GPIO_OUTPUT` | 設為輸出模式 |
| `GPIO_OUTPUT_INIT_HIGH` | 輸出模式，初始值為高 |
| `GPIO_OUTPUT_INIT_LOW` | 輸出模式，初始值為低 |
| `GPIO_OUTPUT_ACTIVE` | 輸出模式，初始值為邏輯有效（考慮 active-low） |
| `GPIO_OUTPUT_INACTIVE` | 輸出模式，初始值為邏輯無效 |
| `GPIO_INPUT` | 設為輸入模式 |
| `GPIO_PULL_UP` | 啟用內建上拉電阻 |
| `GPIO_PULL_DOWN` | 啟用內建下拉電阻 |
| `GPIO_ACTIVE_LOW` | 邏輯反轉：物理低電位 = 邏輯有效 |
| `GPIO_OPEN_DRAIN` | 開漏輸出模式 |
| `GPIO_INT_EDGE_RISING` | 上升沿中斷 |
| `GPIO_INT_EDGE_FALLING` | 下降沿中斷 |
| `GPIO_INT_EDGE_BOTH` | 雙沿中斷 |
| `GPIO_INT_LEVEL_LOW` | 低電位準位中斷 |
| `GPIO_INT_LEVEL_HIGH` | 高電位準位中斷 |

> **重點**：`GPIO_ACTIVE_LOW` 通常在 Devicetree 中設定（`GPIO_ACTIVE_LOW` flag），不需要在 C 程式碼中手動指定。使用 `_dt` API 時會自動套用。

---

## 4. Devicetree 設定

### 4.1 nRF54L15 DK 預設的 GPIO 定義

nRF54L15 DK 的板載 LED 與按鈕已經在 SDK 的板級 `.dts` 中定義好了。以下是簡化版的定義（供理解用）：

```dts
/ {
    aliases {
        led0 = &led0;
        led1 = &led1;
        led2 = &led2;
        led3 = &led3;
        sw0 = &button0;
        sw1 = &button1;
        sw2 = &button2;
        sw3 = &button3;
    };

    leds {
        compatible = "gpio-leds";

        led0: led_0 {
            /* P2.09，低電位有效 */
            gpios = <&gpio2 9 GPIO_ACTIVE_LOW>;
            label = "Green LED 0";
        };

        led1: led_1 {
            gpios = <&gpio1 10 GPIO_ACTIVE_LOW>;
            label = "Green LED 1";
        };

        led2: led_2 {
            gpios = <&gpio2 7 GPIO_ACTIVE_LOW>;
            label = "Green LED 2";
        };

        led3: led_3 {
            gpios = <&gpio1 14 GPIO_ACTIVE_LOW>;
            label = "Green LED 3";
        };
    };

    buttons {
        compatible = "gpio-keys";

        button0: button_0 {
            /* P1.13，低電位有效（按下接地），啟用上拉 */
            gpios = <&gpio1 13 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>;
            label = "Push button 0";
            zephyr,code = <INPUT_KEY_0>;
        };

        button1: button_1 {
            gpios = <&gpio1 9 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>;
            label = "Push button 1";
            zephyr,code = <INPUT_KEY_1>;
        };

        button2: button_2 {
            gpios = <&gpio1 8 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>;
            label = "Push button 2";
            zephyr,code = <INPUT_KEY_2>;
        };

        button3: button_3 {
            gpios = <&gpio0 4 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>;
            label = "Push button 3";
            zephyr,code = <INPUT_KEY_3>;
        };
    };
};
```

### 4.2 自定義 GPIO 腳位 — Overlay 檔案

如果你需要使用板載 LED/按鈕以外的 GPIO 腳位，需要建立一個 `.overlay` 檔案。

**檔案路徑**：`boards/nrf54l15dk_nrf54l15_cpuapp.overlay`

```dts
/*
 * 自定義 GPIO overlay 範例
 * 目標板：nrf54l15dk/nrf54l15/cpuapp
 *
 * 這個範例定義了：
 * 1. 一個自訂的 LED（連接到 P2.00）
 * 2. 一個外部感測器的致能腳位（P1.05）
 * 3. 一個外部中斷輸入腳位（P1.06）
 */

/ {
    /* 自定義的 GPIO 裝置節點 */
    custom_gpio {
        compatible = "gpio-leds";  /* 使用 gpio-leds 作為泛用 GPIO 輸出 */

        /* 外部 LED，連接在 P2.00，高電位有效 */
        ext_led: ext_led_0 {
            gpios = <&gpio2 0 GPIO_ACTIVE_HIGH>;
            label = "External LED";
        };

        /* 感測器致能腳位，P1.05，高電位有效 */
        sensor_en: sensor_enable {
            gpios = <&gpio1 5 GPIO_ACTIVE_HIGH>;
            label = "Sensor Enable";
        };
    };

    /* 外部按鈕或中斷來源 */
    custom_buttons {
        compatible = "gpio-keys";

        /* 外部中斷輸入，P1.06，低電位有效，啟用上拉 */
        ext_irq: ext_interrupt {
            gpios = <&gpio1 6 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>;
            label = "External Interrupt";
        };
    };

    /* 為自定義腳位建立別名（可選但推薦） */
    aliases {
        extled = &ext_led;
        sensoren = &sensor_en;
        extirq = &ext_irq;
    };
};
```

### 4.3 Devicetree 屬性說明

| 屬性 | 說明 |
|------|------|
| `compatible` | 綁定的驅動類型。`gpio-leds` 用於 LED 輸出，`gpio-keys` 用於按鍵輸入 |
| `gpios` | 指定 GPIO 控制器（`&gpio0`/`&gpio1`/`&gpio2`）、腳位編號、旗標 |
| `label` | 人類可讀的標籤名稱（用於除錯與識別） |
| `GPIO_ACTIVE_LOW` | 物理低電位 = 邏輯有效。使用 `gpio_pin_set_dt(spec, 1)` 時實際輸出低電位 |
| `GPIO_ACTIVE_HIGH` | 物理高電位 = 邏輯有效（預設行為） |
| `GPIO_PULL_UP` | 在 Devicetree 層級設定上拉電阻 |
| `GPIO_PULL_DOWN` | 在 Devicetree 層級設定下拉電阻 |

### 4.4 在 C 程式碼中引用 Devicetree 節點

```c
/* 方法一：透過別名取得（推薦） */
#define MY_LED_NODE DT_ALIAS(led0)

/* 方法二：透過路徑取得 */
#define MY_LED_NODE DT_PATH(leds, led_0)

/* 方法三：透過 nodelabel 取得 */
#define MY_LED_NODE DT_NODELABEL(led0)

/* 取得 GPIO spec（包含控制器、腳位、旗標） */
static const struct gpio_dt_spec led = GPIO_DT_SPEC_GET(MY_LED_NODE, gpios);
```

---

## 5. Kconfig 設定

### 5.1 必要的 prj.conf 設定

在你的專案根目錄建立或編輯 `prj.conf`：

```ini
# ==========================================
# GPIO 基礎設定
# ==========================================

# 啟用 GPIO 驅動子系統（必要）
CONFIG_GPIO=y

# ==========================================
# 日誌設定（建議開啟以便除錯）
# ==========================================

# 啟用日誌子系統
CONFIG_LOG=y

# 設定 GPIO 驅動的日誌等級（0=關閉, 1=錯誤, 2=警告, 3=資訊, 4=除錯）
CONFIG_GPIO_LOG_LEVEL_DBG=y

# ==========================================
# 選配設定
# ==========================================

# 啟用 GPIO shell 指令（可在串列終端直接操控 GPIO，開發階段很有用）
# 使用方式：在終端輸入 `gpio` 查看可用指令
CONFIG_GPIO_SHELL=y

# 啟用 shell 子系統（GPIO shell 的前置需求）
CONFIG_SHELL=y
```

### 5.2 Kconfig 選項詳細說明

| 設定項 | 預設值 | 說明 |
|--------|--------|------|
| `CONFIG_GPIO` | `n` | 啟用 GPIO 驅動子系統。**必須設為 y** |
| `CONFIG_GPIO_LOG_LEVEL_DBG` | — | 將 GPIO 驅動日誌設為 DEBUG 等級，方便追蹤問題 |
| `CONFIG_GPIO_SHELL` | `n` | 啟用 GPIO 互動式 shell 指令，開發時非常實用 |
| `CONFIG_GPIO_INIT_PRIORITY` | `40` | GPIO 驅動初始化優先順序，通常不需修改 |
| `CONFIG_GPIO_NRFX` | `y`（自動選取） | Nordic 專用 GPIO 驅動，當目標為 nRF SoC 時自動啟用 |

### 5.3 GPIO Shell 使用範例

啟用 `CONFIG_GPIO_SHELL=y` 後，在串列終端中可以直接操作 GPIO：

```bash
# 列出所有 GPIO 控制器
uart:~$ gpio devices

# 設定 P2.09（LED0）為輸出，初始低電位
uart:~$ gpio conf gpio@50400 9 out

# 設定 P2.09 為高電位（LED 點亮，因為 active-low 實際輸出低）
uart:~$ gpio set gpio@50400 9 1

# 設定 P2.09 為低電位
uart:~$ gpio set gpio@50400 9 0

# 讀取 P1.13（Button 0）的值
uart:~$ gpio get gpio@50300 13
```

---

## 6. 完整程式碼範例

### 6.1 範例一：基本 LED 閃爍（GPIO 輸出）

**檔案：`src/main.c`**

```c
/*
 * nRF54L15 GPIO 輸出範例 — LED 閃爍
 *
 * 功能：讓板載 LED0 以 500ms 間隔閃爍
 * 目標板：nrf54l15dk/nrf54l15/cpuapp
 *
 * 建置指令：
 *   west build -b nrf54l15dk/nrf54l15/cpuapp
 *   west flash
 */

#include <zephyr/kernel.h>          /* k_msleep() 等核心 API */
#include <zephyr/drivers/gpio.h>    /* GPIO API */

/* 閃爍間隔（毫秒） */
#define BLINK_INTERVAL_MS 500

/*
 * 從 Devicetree 取得 LED0 的 GPIO 規格。
 *
 * GPIO_DT_SPEC_GET() 會展開為一個 struct gpio_dt_spec，包含：
 *   - .port：GPIO 控制器裝置指標
 *   - .pin：腳位編號
 *   - .dt_flags：Devicetree 中設定的旗標（如 GPIO_ACTIVE_LOW）
 *
 * DT_ALIAS(led0) 對應 Devicetree 中的 aliases { led0 = &led0; };
 */
static const struct gpio_dt_spec led = GPIO_DT_SPEC_GET(DT_ALIAS(led0), gpios);

int main(void)
{
    int ret;

    /*
     * 步驟 1：檢查 GPIO 裝置是否就緒
     *
     * gpio_is_ready_dt() 會確認：
     *   - GPIO 控制器驅動已初始化
     *   - 裝置可以正常使用
     *
     * 這是防禦性程式設計的好習慣，尤其在系統啟動早期。
     */
    if (!gpio_is_ready_dt(&led)) {
        printk("錯誤：LED GPIO 控制器尚未就緒\n");
        return -ENODEV;
    }

    /*
     * 步驟 2：設定腳位為輸出模式
     *
     * gpio_pin_configure_dt() 參數說明：
     *   - &led：GPIO 規格（包含控制器、腳位、Devicetree 旗標）
     *   - GPIO_OUTPUT_INACTIVE：設為輸出，初始值為邏輯無效
     *     （因為 LED0 是 active-low，邏輯無效 = 物理高電位 = LED 熄滅）
     *
     * 使用 _dt 後綴的 API 會自動合併 Devicetree 中的旗標。
     */
    ret = gpio_pin_configure_dt(&led, GPIO_OUTPUT_INACTIVE);
    if (ret < 0) {
        printk("錯誤：無法設定 LED 腳位，錯誤碼 %d\n", ret);
        return ret;
    }

    printk("LED 閃爍程式啟動\n");

    /*
     * 步驟 3：主迴圈 — 持續切換 LED 狀態
     */
    while (1) {
        /*
         * gpio_pin_toggle_dt() 切換腳位的邏輯電位。
         * 會自動考慮 GPIO_ACTIVE_LOW 旗標。
         *
         * 返回 0 表示成功，負值表示錯誤。
         */
        ret = gpio_pin_toggle_dt(&led);
        if (ret < 0) {
            printk("錯誤：切換 LED 失敗，錯誤碼 %d\n", ret);
            return ret;
        }

        /* 延遲指定時間 */
        k_msleep(BLINK_INTERVAL_MS);
    }

    return 0;  /* 實際上不會執行到這裡 */
}
```

### 6.2 範例二：按鈕輸入讀取（GPIO 輸入 + 上拉電阻）

**檔案：`src/main.c`**

```c
/*
 * nRF54L15 GPIO 輸入範例 — 輪詢按鈕狀態
 *
 * 功能：持續讀取 Button 0 的狀態，並控制 LED0
 *       按下按鈕 → LED 亮，放開按鈕 → LED 滅
 * 目標板：nrf54l15dk/nrf54l15/cpuapp
 */

#include <zephyr/kernel.h>
#include <zephyr/drivers/gpio.h>

/* 輪詢間隔（毫秒） */
#define POLL_INTERVAL_MS 50

/* 從 Devicetree 取得 LED0 和 Button 0 的 GPIO 規格 */
static const struct gpio_dt_spec led = GPIO_DT_SPEC_GET(DT_ALIAS(led0), gpios);
static const struct gpio_dt_spec button = GPIO_DT_SPEC_GET(DT_ALIAS(sw0), gpios);

int main(void)
{
    int ret;

    /* 確認兩個 GPIO 控制器都已就緒 */
    if (!gpio_is_ready_dt(&led)) {
        printk("錯誤：LED GPIO 控制器尚未就緒\n");
        return -ENODEV;
    }
    if (!gpio_is_ready_dt(&button)) {
        printk("錯誤：按鈕 GPIO 控制器尚未就緒\n");
        return -ENODEV;
    }

    /* 設定 LED 為輸出（初始熄滅） */
    ret = gpio_pin_configure_dt(&led, GPIO_OUTPUT_INACTIVE);
    if (ret < 0) {
        printk("錯誤：無法設定 LED 腳位，錯誤碼 %d\n", ret);
        return ret;
    }

    /*
     * 設定按鈕為輸入模式。
     *
     * GPIO_INPUT：設為輸入模式。
     *
     * 注意：上拉電阻已經在 Devicetree 中透過 GPIO_PULL_UP 旗標設定，
     * 使用 gpio_pin_configure_dt() 會自動套用，不需要在這裡重複指定。
     *
     * 如果你需要在程式碼中覆蓋 Devicetree 設定，可以：
     *   gpio_pin_configure_dt(&button, GPIO_INPUT | GPIO_PULL_UP);
     */
    ret = gpio_pin_configure_dt(&button, GPIO_INPUT);
    if (ret < 0) {
        printk("錯誤：無法設定按鈕腳位，錯誤碼 %d\n", ret);
        return ret;
    }

    printk("按鈕輪詢程式啟動\n");

    while (1) {
        /*
         * gpio_pin_get_dt() 讀取腳位的邏輯電位。
         *
         * 因為按鈕在 Devicetree 中標記為 GPIO_ACTIVE_LOW：
         *   - 按鈕按下（物理低電位）→ 返回 1（邏輯有效）
         *   - 按鈕放開（物理高電位，上拉）→ 返回 0（邏輯無效）
         *
         * 返回值：0 或 1 表示邏輯電位，負值表示錯誤。
         */
        int btn_state = gpio_pin_get_dt(&button);
        if (btn_state < 0) {
            printk("錯誤：讀取按鈕失敗，錯誤碼 %d\n", btn_state);
            return btn_state;
        }

        /*
         * gpio_pin_set_dt() 設定腳位的邏輯電位。
         *
         * 第二個參數是邏輯值（0 或 1），會自動考慮 GPIO_ACTIVE_LOW。
         *   - 傳入 1 → LED 點亮（因為 active-low，實際輸出低電位）
         *   - 傳入 0 → LED 熄滅
         */
        gpio_pin_set_dt(&led, btn_state);

        k_msleep(POLL_INTERVAL_MS);
    }

    return 0;
}
```

### 6.3 範例三：GPIO 中斷（按鈕觸發回呼函式）

**檔案：`src/main.c`**

```c
/*
 * nRF54L15 GPIO 中斷範例 — 按鈕中斷觸發 LED 切換
 *
 * 功能：按下 Button 0 時，透過中斷切換 LED0 的開關狀態
 *       使用中斷取代輪詢，更省電且回應更即時
 * 目標板：nrf54l15dk/nrf54l15/cpuapp
 */

#include <zephyr/kernel.h>
#include <zephyr/drivers/gpio.h>

/* 從 Devicetree 取得 LED 和按鈕的 GPIO 規格 */
static const struct gpio_dt_spec led = GPIO_DT_SPEC_GET(DT_ALIAS(led0), gpios);
static const struct gpio_dt_spec button = GPIO_DT_SPEC_GET(DT_ALIAS(sw0), gpios);

/*
 * 宣告 GPIO 中斷回呼資料結構。
 *
 * struct gpio_callback 用於儲存回呼函式的相關資訊。
 * 這個結構體會被傳給 gpio_add_callback()，用來註冊中斷處理函式。
 * 必須宣告為靜態或全域變數，因為 GPIO 驅動會持有它的指標。
 */
static struct gpio_callback button_cb_data;

/*
 * 按鈕中斷回呼函式。
 *
 * 參數說明：
 *   - dev：觸發中斷的 GPIO 控制器裝置指標
 *   - cb：對應的 gpio_callback 結構體指標
 *   - pins：觸發中斷的腳位遮罩（bitmask），
 *           可用 BIT(pin) 來檢查是哪支腳位觸發
 *
 * 注意：這個函式在中斷上下文（ISR）中執行！
 *       不可以呼叫會阻塞的函式（如 k_msleep()、printk() 可能不安全）。
 *       建議只做最少的工作（如設旗標），將複雜邏輯交給工作執行緒。
 */
static void button_pressed_cb(const struct device *dev,
                               struct gpio_callback *cb,
                               uint32_t pins)
{
    /* 切換 LED 狀態 */
    gpio_pin_toggle_dt(&led);
}

int main(void)
{
    int ret;

    /* 確認 GPIO 控制器已就緒 */
    if (!gpio_is_ready_dt(&led) || !gpio_is_ready_dt(&button)) {
        printk("錯誤：GPIO 控制器尚未就緒\n");
        return -ENODEV;
    }

    /* 設定 LED 為輸出（初始熄滅） */
    ret = gpio_pin_configure_dt(&led, GPIO_OUTPUT_INACTIVE);
    if (ret < 0) {
        printk("錯誤：無法設定 LED 腳位，錯誤碼 %d\n", ret);
        return ret;
    }

    /* 設定按鈕為輸入模式 */
    ret = gpio_pin_configure_dt(&button, GPIO_INPUT);
    if (ret < 0) {
        printk("錯誤：無法設定按鈕腳位，錯誤碼 %d\n", ret);
        return ret;
    }

    /*
     * 步驟：設定中斷
     *
     * gpio_pin_interrupt_configure_dt() 參數說明：
     *   - &button：GPIO 規格
     *   - GPIO_INT_EDGE_TO_ACTIVE：當腳位從邏輯無效變為邏輯有效時觸發。
     *     因為按鈕是 GPIO_ACTIVE_LOW，這相當於「下降沿」（按下按鈕）。
     *
     * 其他常用中斷模式：
     *   - GPIO_INT_EDGE_TO_INACTIVE：放開按鈕時觸發
     *   - GPIO_INT_EDGE_BOTH：按下和放開都觸發
     *   - GPIO_INT_EDGE_RISING：物理上升沿（不考慮 active-low）
     *   - GPIO_INT_EDGE_FALLING：物理下降沿
     *   - GPIO_INT_LEVEL_ACTIVE：邏輯有效時持續觸發（準位觸發）
     */
    ret = gpio_pin_interrupt_configure_dt(&button, GPIO_INT_EDGE_TO_ACTIVE);
    if (ret < 0) {
        printk("錯誤：無法設定按鈕中斷，錯誤碼 %d\n", ret);
        return ret;
    }

    /*
     * 初始化回呼結構體並註冊。
     *
     * GPIO_CALLBACK_INIT() 會設定：
     *   - 回呼函式指標（button_pressed_cb）
     *   - 關心的腳位遮罩（BIT(button.pin) 表示只監聽這支腳位）
     *
     * gpio_add_callback() 將回呼註冊到 GPIO 控制器。
     * 同一個控制器可以註冊多個回呼。
     */
    gpio_init_callback(&button_cb_data, button_pressed_cb, BIT(button.pin));

    ret = gpio_add_callback(button.port, &button_cb_data);
    if (ret < 0) {
        printk("錯誤：無法註冊按鈕回呼，錯誤碼 %d\n", ret);
        return ret;
    }

    printk("GPIO 中斷範例啟動 — 按下 Button 0 切換 LED0\n");

    /*
     * 主迴圈保持執行緒存活。
     *
     * 中斷回呼會在背景觸發，主迴圈可以做其他事情。
     * 在這個簡單範例中，主迴圈只是休眠。
     * 在實際應用中，這裡可以放入低功耗模式或其他任務。
     */
    while (1) {
        k_msleep(1000);
    }

    return 0;
}
```

### 6.4 範例四：多按鈕多 LED + 去彈跳（完整應用）

**檔案：`src/main.c`**

```c
/*
 * nRF54L15 完整 GPIO 範例 — 多按鈕、多 LED、軟體去彈跳
 *
 * 功能：
 *   - Button 0 按下 → 切換 LED0
 *   - Button 1 按下 → 切換 LED1
 *   - 使用 k_work_delayable 實現軟體去彈跳（debounce）
 *   - 按鈕按下次數計數並輸出到串列終端
 *
 * 目標板：nrf54l15dk/nrf54l15/cpuapp
 */

#include <zephyr/kernel.h>
#include <zephyr/drivers/gpio.h>

/* 去彈跳延遲時間（毫秒） */
#define DEBOUNCE_DELAY_MS 50

/* LED 定義 */
static const struct gpio_dt_spec led0 = GPIO_DT_SPEC_GET(DT_ALIAS(led0), gpios);
static const struct gpio_dt_spec led1 = GPIO_DT_SPEC_GET(DT_ALIAS(led1), gpios);

/* 按鈕定義 */
static const struct gpio_dt_spec btn0 = GPIO_DT_SPEC_GET(DT_ALIAS(sw0), gpios);
static const struct gpio_dt_spec btn1 = GPIO_DT_SPEC_GET(DT_ALIAS(sw1), gpios);

/* 中斷回呼結構體 */
static struct gpio_callback btn0_cb_data;
static struct gpio_callback btn1_cb_data;

/* 去彈跳用的延遲工作項目 */
static struct k_work_delayable btn0_debounce_work;
static struct k_work_delayable btn1_debounce_work;

/* 按鈕按下次數計數器 */
static volatile uint32_t btn0_count;
static volatile uint32_t btn1_count;

/*
 * 去彈跳工作處理函式 — Button 0
 *
 * 這個函式在系統工作佇列（system workqueue）的執行緒中執行，
 * 不在中斷上下文中，所以可以安全地呼叫 printk() 等函式。
 */
static void btn0_debounce_handler(struct k_work *work)
{
    /* 再次確認按鈕狀態（去彈跳後） */
    int val = gpio_pin_get_dt(&btn0);
    if (val == 1) {
        /* 確認是真正的按下事件 */
        gpio_pin_toggle_dt(&led0);
        btn0_count++;
        printk("Button 0 按下（第 %u 次），LED0 已切換\n", btn0_count);
    }
}

/* 去彈跳工作處理函式 — Button 1 */
static void btn1_debounce_handler(struct k_work *work)
{
    int val = gpio_pin_get_dt(&btn1);
    if (val == 1) {
        gpio_pin_toggle_dt(&led1);
        btn1_count++;
        printk("Button 1 按下（第 %u 次），LED1 已切換\n", btn1_count);
    }
}

/*
 * 按鈕中斷回呼 — Button 0
 *
 * 在中斷上下文中執行，只負責排程延遲工作，
 * 將實際處理延遲到工作佇列執行緒中。
 */
static void btn0_isr(const struct device *dev,
                     struct gpio_callback *cb,
                     uint32_t pins)
{
    /*
     * k_work_reschedule() 會：
     *   - 如果工作尚未排程 → 排程在指定延遲後執行
     *   - 如果工作已排程 → 重新計時（取消舊的，重新排程）
     *
     * 這正好實現了去彈跳：在最後一次邊沿觸發後等待穩定。
     */
    k_work_reschedule(&btn0_debounce_work, K_MSEC(DEBOUNCE_DELAY_MS));
}

/* 按鈕中斷回呼 — Button 1 */
static void btn1_isr(const struct device *dev,
                     struct gpio_callback *cb,
                     uint32_t pins)
{
    k_work_reschedule(&btn1_debounce_work, K_MSEC(DEBOUNCE_DELAY_MS));
}

int main(void)
{
    int ret;

    /* 檢查所有 GPIO 裝置是否就緒 */
    if (!gpio_is_ready_dt(&led0) || !gpio_is_ready_dt(&led1) ||
        !gpio_is_ready_dt(&btn0) || !gpio_is_ready_dt(&btn1)) {
        printk("錯誤：GPIO 控制器尚未就緒\n");
        return -ENODEV;
    }

    /* 設定 LED 為輸出 */
    gpio_pin_configure_dt(&led0, GPIO_OUTPUT_INACTIVE);
    gpio_pin_configure_dt(&led1, GPIO_OUTPUT_INACTIVE);

    /* 設定按鈕為輸入 */
    gpio_pin_configure_dt(&btn0, GPIO_INPUT);
    gpio_pin_configure_dt(&btn1, GPIO_INPUT);

    /* 設定按鈕中斷（按下觸發） */
    gpio_pin_interrupt_configure_dt(&btn0, GPIO_INT_EDGE_TO_ACTIVE);
    gpio_pin_interrupt_configure_dt(&btn1, GPIO_INT_EDGE_TO_ACTIVE);

    /* 初始化中斷回呼 */
    gpio_init_callback(&btn0_cb_data, btn0_isr, BIT(btn0.pin));
    gpio_init_callback(&btn1_cb_data, btn1_isr, BIT(btn1.pin));

    gpio_add_callback(btn0.port, &btn0_cb_data);
    gpio_add_callback(btn1.port, &btn1_cb_data);

    /* 初始化去彈跳延遲工作 */
    k_work_init_delayable(&btn0_debounce_work, btn0_debounce_handler);
    k_work_init_delayable(&btn1_debounce_work, btn1_debounce_handler);

    printk("多按鈕多 LED 範例啟動（含去彈跳）\n");
    printk("  Button 0 → LED0\n");
    printk("  Button 1 → LED1\n");

    while (1) {
        k_msleep(1000);
    }

    return 0;
}
```

### 6.5 CMakeLists.txt

每個範例都需要一個 `CMakeLists.txt`：

```cmake
# SPDX-License-Identifier: Apache-2.0

cmake_minimum_required(VERSION 3.20.0)

# 尋找 Zephyr 套件（必須在 project() 之前）
find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})

# 定義專案名稱
project(gpio_example)

# 加入原始碼檔案
target_sources(app PRIVATE src/main.c)
```

### 6.6 建置與燒錄指令

```bash
# 建置（指定目標板）
west build -b nrf54l15dk/nrf54l15/cpuapp

# 燒錄到開發板
west flash

# 查看串列終端輸出（115200 baud）
# 方式一：使用內建的 serial monitor
west espressif monitor
# 方式二：使用通用工具
minicom -D /dev/ttyACM0 -b 115200
# 方式三：使用 screen
screen /dev/ttyACM0 115200
```

---

## 7. 進階用法

### 7.1 GPIO Port 批次操作

當需要同時操作多支腳位時，可以使用 port 層級的 API，效率更高：

```c
#include <zephyr/drivers/gpio.h>

/*
 * 批次設定同一個 port 上的多支腳位。
 *
 * gpio_port_set_masked() 可以用一次暫存器寫入同時設定多支腳位，
 * 比逐一呼叫 gpio_pin_set() 快很多。
 */
void set_multiple_pins(const struct device *gpio_dev)
{
    /*
     * 設定 pin 0、1、2 為高電位，其他不變。
     *
     * 參數說明：
     *   - gpio_dev：GPIO 控制器
     *   - mask：要操作的腳位遮罩（BIT(0) | BIT(1) | BIT(2) = 0x07）
     *   - value：要設定的值（0x07 = 三支都設為高）
     */
    gpio_port_set_masked(gpio_dev, 0x07, 0x07);

    /* 只設定 pin 0 為低、pin 1 為高，pin 2 不變 */
    gpio_port_set_masked(gpio_dev, BIT(0) | BIT(1), BIT(1));

    /*
     * 讀取整個 port 的值。
     *
     * gpio_port_get() 一次讀取所有腳位的狀態，
     * 返回 32 位元的值，每個位元對應一支腳位。
     */
    gpio_port_value_t port_value;
    gpio_port_get(gpio_dev, &port_value);

    /* 檢查 pin 3 是否為高電位 */
    if (port_value & BIT(3)) {
        printk("Pin 3 為高電位\n");
    }
}
```

### 7.2 GPIO 與低功耗模式

在低功耗應用中，GPIO 的設定對功耗影響很大：

```c
/*
 * 低功耗 GPIO 設定技巧
 */

/* 1. 未使用的腳位設為輸出低電位（避免浮空消耗電流） */
gpio_pin_configure(gpio_dev, unused_pin, GPIO_OUTPUT_INIT_LOW);

/* 2. 輸入腳位務必啟用上拉或下拉（避免浮空） */
gpio_pin_configure_dt(&sensor_pin, GPIO_INPUT | GPIO_PULL_DOWN);

/*
 * 3. 使用電位準位中斷作為喚醒來源
 *
 * nRF54L15 在 System OFF 模式下，可以透過 GPIO DETECT 信號喚醒。
 * 需要在 Devicetree 中設定 sense 屬性。
 */
gpio_pin_interrupt_configure_dt(&wakeup_pin, GPIO_INT_LEVEL_ACTIVE);

/* 4. 進入低功耗前，關閉不需要的 GPIO 中斷 */
gpio_pin_interrupt_configure_dt(&button, GPIO_INT_DISABLE);
```

### 7.3 在中斷中安全地處理複雜邏輯

```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/gpio.h>

/*
 * 使用訊息佇列（message queue）將中斷事件傳遞到工作執行緒。
 * 這是處理複雜中斷邏輯的推薦模式。
 */

/* 定義事件類型 */
struct gpio_event {
    uint8_t pin;        /* 觸發的腳位 */
    int64_t timestamp;  /* 觸發時間戳 */
};

/* 建立訊息佇列：最多儲存 10 個事件 */
K_MSGQ_DEFINE(gpio_event_queue, sizeof(struct gpio_event), 10, 4);

/* 中斷回呼：只負責將事件放入佇列 */
static void gpio_isr(const struct device *dev,
                     struct gpio_callback *cb,
                     uint32_t pins)
{
    struct gpio_event evt = {
        .pin = 0,  /* 可用 find_lsb_set(pins) - 1 取得實際腳位 */
        .timestamp = k_uptime_get(),
    };

    /* k_msgq_put() 可在 ISR 中安全呼叫（不會阻塞） */
    k_msgq_put(&gpio_event_queue, &evt, K_NO_WAIT);
}

/* 工作執行緒：處理佇列中的事件 */
void gpio_event_thread(void *p1, void *p2, void *p3)
{
    struct gpio_event evt;

    while (1) {
        /* 等待事件（會阻塞直到有事件） */
        k_msgq_get(&gpio_event_queue, &evt, K_FOREVER);

        /* 在執行緒中可以安全地做任何事情 */
        printk("GPIO 事件：pin %u，時間 %lld ms\n",
               evt.pin, evt.timestamp);

        /* 這裡可以執行複雜的邏輯，如存取 flash、發送 BLE 封包等 */
    }
}

/* 定義專用執行緒 */
K_THREAD_DEFINE(gpio_evt_tid, 1024,
                gpio_event_thread, NULL, NULL, NULL,
                7, 0, 0);
```

### 7.4 GPIO 同時作為輸入和輸出

某些協定（如 1-Wire）需要同一支腳位在輸入和輸出之間切換：

```c
/* 切換為輸出並拉低 */
gpio_pin_configure_dt(&data_pin, GPIO_OUTPUT_INIT_LOW);
k_busy_wait(500);  /* 維持低電位 500 微秒 */

/* 切換為輸入並讀取 */
gpio_pin_configure_dt(&data_pin, GPIO_INPUT | GPIO_PULL_UP);
k_busy_wait(70);   /* 等待裝置回應 */
int val = gpio_pin_get_dt(&data_pin);
```

### 7.5 開漏輸出模式

I²C 匯流排或多裝置共用線路需要開漏模式：

```c
/*
 * 開漏模式：只能主動拉低，高電位由外部上拉電阻提供。
 * 多個裝置可以安全地共用同一條線路（wired-AND 邏輯）。
 */
gpio_pin_configure_dt(&shared_line,
                      GPIO_OUTPUT_INIT_HIGH | GPIO_OPEN_DRAIN | GPIO_PULL_UP);

/* 拉低（主動驅動） */
gpio_pin_set_dt(&shared_line, 0);

/* 釋放（由外部上拉回高） */
gpio_pin_set_dt(&shared_line, 1);
```

---

## 8. 常見問題與除錯

### 8.1 FAQ

#### Q1：LED 的亮滅邏輯反了怎麼辦？

**原因**：nRF54L15 DK 的 LED 是 active-low（低電位亮），但你可能使用了不含 `_dt` 後綴的 API，沒有自動處理 `GPIO_ACTIVE_LOW` 旗標。

**解決方案**：一律使用 `_dt` 後綴的 API（`gpio_pin_set_dt()`、`gpio_pin_get_dt()`），它們會自動處理邏輯反轉。

```c
/* 錯誤：直接使用物理電位 API */
gpio_pin_set(led.port, led.pin, 1);  /* 物理高電位 → LED 實際上熄滅 */

/* 正確：使用 _dt API，自動處理 active-low */
gpio_pin_set_dt(&led, 1);  /* 邏輯有效 → LED 點亮 */
```

#### Q2：按鈕中斷觸發多次怎麼辦？

**原因**：機械按鈕的彈跳（bounce），一次按壓會產生多次邊沿變化。

**解決方案**：實作軟體去彈跳（參考範例 6.4），或使用硬體去彈跳電容。

#### Q3：`gpio_is_ready_dt()` 返回 false？

**可能原因**：
1. `prj.conf` 中沒有設定 `CONFIG_GPIO=y`
2. Devicetree 中的 GPIO 控制器節點被禁用
3. 使用了不存在的別名

**排查步驟**：
```bash
# 檢查建置後的 Devicetree 輸出
cat build/zephyr/zephyr.dts | grep -A5 "gpio@"

# 確認 Kconfig 設定
cat build/zephyr/.config | grep CONFIG_GPIO
```

#### Q4：中斷回呼函式不被觸發？

**排查清單**：
1. 確認 `gpio_pin_interrupt_configure_dt()` 返回 0
2. 確認 `gpio_add_callback()` 返回 0
3. 確認回呼的腳位遮罩正確：`BIT(button.pin)` 而不是 `BIT(按鈕的 port 編號)`
4. 確認沒有其他程式碼後來呼叫了 `gpio_pin_interrupt_configure_dt(..., GPIO_INT_DISABLE)`
5. 在回呼函式中加入簡單的 LED toggle 確認是否進入回呼

#### Q5：GPIO 不能使用特定腳位？

**原因**：某些腳位可能被其他周邊佔用（如 UART、SPI、I²C）。nRF54L15 使用 pin mux 機制，每支腳位同一時間只能分配給一個功能。

**排查**：
```bash
# 檢查腳位分配衝突
cat build/zephyr/zephyr.dts | grep -B2 -A5 "pinctrl"
```

### 8.2 常見編譯錯誤

| 錯誤訊息 | 原因 | 解決方案 |
|----------|------|----------|
| `undefined reference to __device_dts_ord_XX` | Devicetree 節點不存在或被禁用 | 檢查 `.overlay` 或 `.dts` 中的節點定義 |
| `'GPIO_DT_SPEC_GET' undeclared` | 缺少標頭檔 | 加入 `#include <zephyr/drivers/gpio.h>` |
| `DT_N_ALIAS_led0_P_gpios_IDX_0_EXISTS is not defined` | `led0` 別名不存在 | 確認目標板有定義 `led0` 別名 |
| `GPIO_OUTPUT_INACTIVE undeclared` | SDK 版本太舊 | 更新到 nRF Connect SDK 2.6+ |

### 8.3 除錯技巧

```c
/* 1. 使用 printk 輸出 GPIO 規格資訊 */
printk("LED: port=%s, pin=%d, flags=0x%x\n",
       led.port->name, led.pin, led.dt_flags);

/* 2. 使用 GPIO shell 即時測試腳位 */
/* 在 prj.conf 中啟用 CONFIG_GPIO_SHELL=y */

/* 3. 使用邏輯分析儀觀察實際波形 */
/* 在關鍵位置插入 toggle pin 作為觀測點 */
static const struct gpio_dt_spec debug_pin =
    GPIO_DT_SPEC_GET(DT_ALIAS(led3), gpios);

gpio_pin_configure_dt(&debug_pin, GPIO_OUTPUT_INACTIVE);

/* 在要觀測的程式碼前後切換 */
gpio_pin_set_dt(&debug_pin, 1);
/* ... 要觀測的程式碼 ... */
gpio_pin_set_dt(&debug_pin, 0);
```

---

## 9. API 快速參考

### 9.1 GPIO 設定 API

| 函式 | 說明 | 返回值 |
|------|------|--------|
| `gpio_is_ready_dt(const struct gpio_dt_spec *spec)` | 檢查 GPIO 控制器是否已初始化就緒 | `true`：就緒；`false`：未就緒 |
| `gpio_pin_configure_dt(const struct gpio_dt_spec *spec, gpio_flags_t extra_flags)` | 設定腳位模式（自動合併 Devicetree 旗標） | `0`：成功；負值：錯誤碼 |
| `gpio_pin_configure(const struct device *port, gpio_pin_t pin, gpio_flags_t flags)` | 設定腳位模式（手動指定所有旗標） | `0`：成功；負值：錯誤碼 |

### 9.2 GPIO 讀寫 API

| 函式 | 說明 | 返回值 |
|------|------|--------|
| `gpio_pin_get_dt(const struct gpio_dt_spec *spec)` | 讀取腳位邏輯值（考慮 active-low） | `0` 或 `1`：邏輯值；負值：錯誤 |
| `gpio_pin_get(const struct device *port, gpio_pin_t pin)` | 讀取腳位物理值 | `0` 或 `1`：物理值；負值：錯誤 |
| `gpio_pin_set_dt(const struct gpio_dt_spec *spec, int value)` | 設定腳位邏輯值（考慮 active-low） | `0`：成功；負值：錯誤碼 |
| `gpio_pin_set(const struct device *port, gpio_pin_t pin, int value)` | 設定腳位物理值 | `0`：成功；負值：錯誤碼 |
| `gpio_pin_toggle_dt(const struct gpio_dt_spec *spec)` | 切換腳位狀態 | `0`：成功；負值：錯誤碼 |

### 9.3 GPIO Port 批次 API

| 函式 | 說明 | 返回值 |
|------|------|--------|
| `gpio_port_get(const struct device *port, gpio_port_value_t *value)` | 讀取整個 port 的所有腳位值 | `0`：成功；負值：錯誤碼 |
| `gpio_port_set_masked(const struct device *port, gpio_port_pins_t mask, gpio_port_value_t value)` | 批次設定指定腳位的值 | `0`：成功；負值：錯誤碼 |
| `gpio_port_set_bits(const struct device *port, gpio_port_pins_t pins)` | 將指定腳位設為高電位 | `0`：成功；負值：錯誤碼 |
| `gpio_port_clear_bits(const struct device *port, gpio_port_pins_t pins)` | 將指定腳位設為低電位 | `0`：成功；負值：錯誤碼 |
| `gpio_port_toggle_bits(const struct device *port, gpio_port_pins_t pins)` | 切換指定腳位的狀態 | `0`：成功；負值：錯誤碼 |

### 9.4 GPIO 中斷 API

| 函式 | 說明 | 返回值 |
|------|------|--------|
| `gpio_pin_interrupt_configure_dt(const struct gpio_dt_spec *spec, gpio_flags_t flags)` | 設定腳位的中斷模式 | `0`：成功；負值：錯誤碼 |
| `gpio_init_callback(struct gpio_callback *callback, gpio_callback_handler_t handler, gpio_port_pins_t pin_mask)` | 初始化回呼結構體 | 無（void） |
| `gpio_add_callback(const struct device *port, struct gpio_callback *callback)` | 將回呼註冊到 GPIO 控制器 | `0`：成功；負值：錯誤碼 |
| `gpio_remove_callback(const struct device *port, struct gpio_callback *callback)` | 移除已註冊的回呼 | `0`：成功；負值：錯誤碼 |

### 9.5 回呼函式簽名

```c
/*
 * GPIO 中斷回呼函式型別定義
 *
 * @param dev   觸發中斷的 GPIO 控制器裝置
 * @param cb    對應的 gpio_callback 結構體
 * @param pins  觸發中斷的腳位遮罩（可用 BIT(n) 檢查）
 */
typedef void (*gpio_callback_handler_t)(
    const struct device *dev,
    struct gpio_callback *cb,
    gpio_port_pins_t pins
);
```

### 9.6 Devicetree 巨集參考

| 巨集 | 說明 | 範例 |
|------|------|------|
| `DT_ALIAS(alias)` | 透過別名取得節點 ID | `DT_ALIAS(led0)` |
| `DT_NODELABEL(label)` | 透過節點標籤取得節點 ID | `DT_NODELABEL(led0)` |
| `DT_PATH(...)` | 透過路徑取得節點 ID | `DT_PATH(leds, led_0)` |
| `GPIO_DT_SPEC_GET(node_id, prop)` | 從節點取得 GPIO 規格 | `GPIO_DT_SPEC_GET(DT_ALIAS(led0), gpios)` |
| `GPIO_DT_SPEC_GET_BY_IDX(node_id, prop, idx)` | 取得節點中第 idx 個 GPIO 規格 | `GPIO_DT_SPEC_GET_BY_IDX(node, gpios, 1)` |

---

## 10. 延伸閱讀

### 10.1 Nordic 官方文件

- [nRF54L15 產品頁面](https://www.nordicsemi.com/Products/nRF54L15)
- [nRF54L15 資料手冊（Datasheet）](https://docs.nordicsemi.com/bundle/ps_nrf54L15/page/keyfeatures_html5.html) — GPIO 電氣規格、暫存器定義
- [nRF Connect SDK GPIO 驅動文件](https://docs.nordicsemi.com/bundle/ncs-latest/page/zephyr/hardware/peripherals/gpio.html)
- [nRF54L15 DK 使用者指南](https://docs.nordicsemi.com/bundle/ug_nrf54l15_dk/page/UG/dk/intro.html) — 開發板電路圖、腳位對應

### 10.2 Zephyr 官方文件

- [Zephyr GPIO API 參考](https://docs.zephyrproject.org/latest/hardware/peripherals/gpio.html)
- [Zephyr Devicetree 指南](https://docs.zephyrproject.org/latest/build/dts/index.html)
- [Zephyr GPIO Bindings](https://docs.zephyrproject.org/latest/build/dts/api/bindings.html#gpio)

### 10.3 SDK 範例專案

nRF Connect SDK 中包含以下 GPIO 相關範例：

```
# Zephyr 基礎 GPIO 範例
zephyr/samples/basic/blinky/           — LED 閃爍（最簡單的 GPIO 輸出）
zephyr/samples/basic/button/           — 按鈕輸入（GPIO 輸入 + 中斷）
zephyr/samples/basic/blinky_pwm/       — PWM 控制 LED 亮度

# Nordic 特有範例
nrf/samples/peripheral/gpio/           — Nordic GPIO 範例（若存在）
```

### 10.4 相關主題

學完 GPIO 基礎後，建議繼續學習：

- **PWM**：精確控制 LED 亮度或馬達速度
- **GPIOTE**：GPIO Tasks and Events，更進階的 GPIO 事件處理
- **SPI / I²C**：使用 GPIO 實現的串列通訊協定
- **電源管理**：GPIO 在低功耗模式下的行為
- **Devicetree 進階**：pin mux、pinctrl 設定
