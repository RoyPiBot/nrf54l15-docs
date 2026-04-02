---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [wireless, build-system, tool]
difficulty: intermediate
keywords: [NanoClaw, AI Agent, Zephyr, nRF54L15, embedded AI, 容器隔離]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-03
---

# NanoClaw + nRF54L15：用 AI Agent 驅動嵌入式開發

## 概述

NanoClaw 是一個極簡 AI Agent 框架，設計用於在資源受限的嵌入式設備上執行。nRF54L15 提供足夠的運算能力和記憶體來運行單一 Agent 執行循環，可用於無線感測網路、本地推理和邊緣 AI 應用。本文檔說明如何在 nRF54L15 上初始化和執行 NanoClaw Agent。

## Devicetree + Kconfig

**prj.conf** — 最小必要配置：
```ini
CONFIG_MAIN_STACK_SIZE=4096
CONFIG_THREAD_STACK_INFO=y
CONFIG_SHELL=y
CONFIG_UART_CONSOLE=y
CONFIG_CONSOLE_HANDLER=y
CONFIG_PRINTK=y
CONFIG_LOG=y
CONFIG_LOG_BACKEND_UART=y
```

**boards/nrf54l15dk_nrf54l15_cpuapp.overlay**：
```devicetree
/ {
    chosen {
        zephyr,console = &uart0;
        zephyr,sram = &sram0;
    };
};

&uart0 {
    status = "okay";
    current-speed = <115200>;
};
```

## 程式碼範例

```c
#include <zephyr/kernel.h>
#include <zephyr/logging/log.h>
#include <string.h>
#include <stdio.h>

LOG_MODULE_REGISTER(nanoclaw_demo, LOG_LEVEL_INF);

/* 極簡 NanoClaw Agent 結構 */
typedef struct {
    char agent_name[32];
    uint32_t step_count;
    char last_action[64];
} nanoclaw_agent_t;

/* 初始化 Agent */
static void agent_init(nanoclaw_agent_t *agent, const char *name) {
    strncpy(agent->agent_name, name, sizeof(agent->agent_name) - 1);
    agent->step_count = 0;
    memset(agent->last_action, 0, sizeof(agent->last_action));
    LOG_INF("Agent '%s' 已初始化", agent->agent_name);
}

/* 執行單一 Agent 步驟 */
static void agent_step(nanoclaw_agent_t *agent) {
    agent->step_count++;

    /* 模擬決策：根據步驟數選擇動作 */
    if (agent->step_count % 3 == 0) {
        strncpy(agent->last_action, "sense", sizeof(agent->last_action) - 1);
    } else if (agent->step_count % 3 == 1) {
        strncpy(agent->last_action, "process", sizeof(agent->last_action) - 1);
    } else {
        strncpy(agent->last_action, "transmit", sizeof(agent->last_action) - 1);
    }

    LOG_INF("[步驟 %u] Agent '%s' 執行: %s",
            agent->step_count, agent->agent_name, agent->last_action);
}

int main(void) {
    LOG_INF("=== NanoClaw nRF54L15 示範 ===");

    nanoclaw_agent_t agent;
    agent_init(&agent, "sensor_agent");

    /* 執行 10 個迭代 */
    for (int i = 0; i < 10; i++) {
        agent_step(&agent);
        k_sleep(K_MSEC(500));
    }

    LOG_INF("Agent 完成 %u 步，最後動作: %s",
            agent.step_count, agent.last_action);

    return 0;
}
```

## 常見問題

**Q: NanoClaw Agent 執行需要多少 RAM？**
A: 基礎實例約 1KB。nRF54L15 擁有 256KB SRAM，可支援多個 Agent 或複雜的狀態機。建議保留 50% 供系統棧使用。

**Q: 如何整合無線通訊？**
A: 在 `prj.conf` 中啟用 `CONFIG_BLUETOOTH=y` 或 `CONFIG_IEEE802154=y`，在 Agent step 中呼叫相應的 BLE/802.15.4 API 執行動作。

**Q: 可以在中斷處理中執行 Agent step 嗎？**
A: 不建議。Agent step 可能需要鎖定或條件變數。改用 k_work_queue 排隊任務至工作執行緒。

## API 快速參考

| 函式 | 說明 |
|------|------|
| `k_sleep(K_MSEC(ms))` | 阻塞延遲（毫秒） |
| `k_work_queue_start()` | 啟動工作隊列執行緒 |
| `k_work_submit()` | 提交非同步工作 |
| `LOG_INF()` | 記錄資訊等級訊息 |
| `strncpy()` | 安全字串複製 |
