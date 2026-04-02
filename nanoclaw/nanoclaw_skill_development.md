---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [peripheral, wireless, build-system]
difficulty: intermediate
keywords: [skill-development, AI-integration, sensor-fusion, callback-pattern, message-queue]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-03
---

# NanoClaw Skill 開發：為 nRF54L15 寫自訂 AI 技能

## 概述

NanoClaw 技能是 nRF54L15 上實現特定功能的模組化元件，與 AI Agent 通過結構化訊息隊列互動。技能負責感測器數據採集、即時處理、決策邏輯，並將結果傳回 Agent。本文示範溫度監測技能的完整開發流程。

## Devicetree + Kconfig

**prj.conf:**
```ini
CONFIG_ZEPHYR_LOG=y
CONFIG_LOG_MODE_IMMEDIATE=y
CONFIG_ADC=y
CONFIG_SENSOR=y
CONFIG_TEMP_NRF5=y
CONFIG_POLL=y
CONFIG_KERNEL_MEM_POOL=y
CONFIG_DYNAMIC_THREAD_STACK_SIZE=y
```

**nrf54l15-skill.overlay:**
```devicetree
/ {
  adc0: adc@400e8000 {
    status = "okay";
    #address-cells = <1>;
    #size-cells = <0>;

    channel@0 {
      reg = <0>;
      zephyr,gain = "ADC_GAIN_1";
      zephyr,reference = "ADC_REF_INTERNAL";
      zephyr,acquisition-time = <ADC_ACQ_TIME_DEFAULT>;
      zephyr,input-positive = <NRF_SAADC_AIN4>;
      zephyr,resolution = <12>;
    };
  };
};
```

## 程式碼範例

```c
/* nanoclaw_skill.c - NanoClaw 技能框架 */
#include <zephyr/kernel.h>
#include <zephyr/drivers/adc.h>
#include <zephyr/drivers/sensor.h>
#include <zephyr/logging/log.h>
#include <string.h>
#include <stdint.h>

LOG_MODULE_REGISTER(nanoclaw_skill, LOG_LEVEL_INF);

/* ===== 訊息結構定義 ===== */
typedef struct {
    uint32_t timestamp;           // 時間戳記（毫秒）
    uint16_t sensor_id;           // 感測器編號
    int16_t value;                // 原始值（°C * 100）
    uint8_t status;               // 狀態碼
} skill_sensor_data_t;

typedef struct {
    char skill_name[32];          // 技能名稱
    skill_sensor_data_t data;     // 感測器數據
    uint32_t priority;            // 優先級
} skill_message_t;

/* ===== 全域訊息隊列 ===== */
K_MSGQ_DEFINE(skill_msgq, sizeof(skill_message_t), 16, 4);
K_MUTEX_DEFINE(skill_state_lock);

/* ===== 感測器讀取線程 ===== */
static int read_temperature(int16_t *temp_raw) {
    // 模擬溫度採樣（實際應使用 ADC / I2C 感測器驅動）
    const struct device *adc = device_get_binding(DT_LABEL(DT_ALIAS(adc0)));
    if (!adc) {
        LOG_ERR("ADC 設備初始化失敗");
        return -ENODEV;
    }

    // 簡化版：返回模擬溫度
    *temp_raw = 2500 + (k_uptime_get() % 500 - 250);
    return 0;
}

/* ===== 技能回調函式（AI Agent 調用） ===== */
int skill_execute_callback(const char *skill_name, void *context) {
    int16_t temp;

    // 讀取感測器
    if (read_temperature(&temp) != 0) {
        LOG_WRN("感測器讀取失敗");
        return -EIO;
    }

    // 建構訊息
    skill_message_t msg = {
        .data = {
            .timestamp = k_uptime_get_32(),
            .sensor_id = 1,
            .value = temp,
            .status = (temp > 3000) ? 1 : 0  // 過溫警告
        },
        .priority = 10
    };
    strncpy(msg.skill_name, skill_name, sizeof(msg.skill_name) - 1);

    // 發送至隊列
    if (k_msgq_put(&skill_msgq, &msg, K_NO_WAIT) != 0) {
        LOG_WRN("訊息隊列已滿");
        return -EBUSY;
    }

    LOG_INF("技能執行: %s, 溫度=%d°C", skill_name, temp / 100);
    return 0;
}

/* ===== AI Agent 輪詢線程 ===== */
void skill_worker_thread(void *arg1, void *arg2, void *arg3) {
    skill_message_t msg;

    while (1) {
        if (k_msgq_get(&skill_msgq, &msg, K_MSEC(1000)) == 0) {
            // 處理訊息：可發送至雲端 AI、本地決策引擎等
            LOG_INF("[%s] 溫度=%d°C (狀態=%d)",
                    msg.skill_name, msg.data.value / 100, msg.data.status);

            if (msg.data.status == 1) {
                LOG_WRN("⚠️ 溫度過高！觸發警報");
                // 呼叫警報技能、連接 Telegram 等
            }
        }
    }
}

/* ===== 初始化函式 ===== */
void nanoclaw_skill_init(void) {
    LOG_INF("初始化 NanoClaw 技能系統");

    // 啟動技能處理線程
    static K_THREAD_STACK_DEFINE(skill_stack, 1024);
    static struct k_thread skill_thread_data;
    k_thread_create(&skill_thread_data, skill_stack, K_THREAD_STACK_SIZEOF(skill_stack),
                    skill_worker_thread, NULL, NULL, NULL,
                    5, 0, K_NO_WAIT);
}

/* ===== 訊息隊列查詢 API ===== */
int skill_get_latest_status(char *buffer, size_t max_len) {
    // 從隊列讀取最新訊息（非阻塞）
    skill_message_t msg;
    if (k_msgq_get(&skill_msgq, &msg, K_NO_WAIT) == 0) {
        snprintf(buffer, max_len, "skill:%s,temp:%d,ts:%u",
                 msg.skill_name, msg.data.value / 100, msg.data.timestamp);
        return 0;
    }
    return -EAGAIN;
}

/* ===== 主程式 ===== */
void main(void) {
    LOG_INF("🚀 NanoClaw 技能系統啟動");
    nanoclaw_skill_init();

    // 定期執行技能（模擬 AI Agent 調度）
    for (int i = 0; i < 60; i++) {
        skill_execute_callback("temperature_monitor", NULL);
        k_sleep(K_SECONDS(1));
    }
}
```

## 常見問題

**Q: 技能與 AI Agent 如何通訊？**
A: 使用訊息隊列（k_msgq_*）解耦。技能發送結構化訊息，Agent 非阻塞輪詢；支援優先級、超時。

**Q: 如何在多個感測器間均衡資源？**
A: 用線程優先級（k_thread_create 的 prio 參數）調控感測器採樣頻率；高優先級感測器（如安全相關）優先執行。

**Q: 訊息隊列滿了怎麼辦？**
A: 返回 -EBUSY，Agent 可實現重試邏輯或降級機制。建議用環形緩衝取代固定隊列。

## API 快速參考

| 函式 | 簽名 | 用途 |
|------|------|------|
| `skill_execute_callback` | `int (const char *name, void *ctx)` | 執行技能，採集數據並入隊 |
| `skill_get_latest_status` | `int (char *buf, size_t len)` | 查詢最新技能狀態 |
| `k_msgq_put` | `int (struct k_msgq *q, const void *data, k_timeout_t timeout)` | 非阻塞訊息發送 |
| `k_msgq_get` | `int (struct k_msgq *q, void *data, k_timeout_t timeout)` | 非阻塞訊息接收 |
| `k_thread_create` | `k_tid_t (...)` | 建立技能處理執行緒 |
