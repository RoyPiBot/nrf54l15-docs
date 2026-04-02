---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [build-system, wireless, peripheral]
difficulty: advanced
keywords: [NanoClaw, AI Agent, Container Isolation, Lightweight Framework, Thread Management]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-03
---

# NanoClaw 架構總覽：極簡 AI Agent 框架、容器隔離設計

## 概述

NanoClaw 是為嵌入式系統設計的輕量級 AI Agent 框架，運行在 nRF54L15 上。通過 Zephyr kernel 的 Thread 隔離和記憶體管理，實現容器級隔離，允許多個 Agent 獨立執行推理任務，同時保持系統穩定性和可預測的資源消耗。

## Devicetree + Kconfig

### prj.conf
```conf
# 核心 Zephyr 配置
CONFIG_KERNEL_SHELL=y
CONFIG_SYSTEM_WORKQUEUE_STACK_SIZE=2048

# Thread & Memory 隔離
CONFIG_THREAD_STACK_INFO=y
CONFIG_THREAD_MONITOR=y
CONFIG_NUM_COOP_PRIORITIES=8
CONFIG_NUM_PREEMPT_PRIORITIES=8

# NanoClaw Agent 框架
CONFIG_NANOCLAW_ENABLED=y
CONFIG_NANOCLAW_MAX_AGENTS=4
CONFIG_NANOCLAW_AGENT_STACK_SIZE=4096
CONFIG_NANOCLAW_AGENT_PRIORITY=5

# 無線通訊 (BLE)
CONFIG_BT=y
CONFIG_BT_CENTRAL=y
CONFIG_BT_PERIPHERAL=y

# 記憶體最佳化
CONFIG_HEAP_MEM_POOL_SIZE=8192
CONFIG_MAIN_STACK_SIZE=2048
```

### app.overlay
```devicetree
/ {
	nanoclaw: nanoclaw@0 {
		compatible = "nanoclaw,agent-framework";
		max-agents = <4>;
		agent-stack-size = <4096>;
		status = "okay";
	};
};
```

## 程式碼範例

### 核心 Agent 定義
```c
#include <zephyr/kernel.h>
#include <zephyr/logging/log.h>

LOG_MODULE_REGISTER(nanoclaw, LOG_LEVEL_INF);

/* Agent 狀態列舉 */
enum agent_state {
	AGENT_IDLE,
	AGENT_RUNNING,
	AGENT_WAITING,
	AGENT_ERROR
};

/* Agent 結構 - 容器隔離單位 */
typedef struct {
	uint8_t id;
	enum agent_state state;
	k_thread_t thread;
	uint8_t stack[4096];
	struct k_msgq inbox;
	void (*task_fn)(void *);
	void *task_data;
} nanoclaw_agent_t;

/* Agent 執行函式 */
static void agent_entry(void *arg1, void *arg2, void *arg3) {
	nanoclaw_agent_t *agent = (nanoclaw_agent_t *)arg1;

	agent->state = AGENT_RUNNING;
	LOG_INF("Agent %d 啟動，開始推理任務", agent->id);

	/* 執行 Agent 工作 */
	if (agent->task_fn) {
		agent->task_fn(agent->task_data);
	}

	agent->state = AGENT_IDLE;
	LOG_INF("Agent %d 完成任務", agent->id);
}

/* 建立 Agent - 容器隔離 */
int nanoclaw_agent_create(nanoclaw_agent_t *agent, uint8_t id,
			  void (*task_fn)(void *), void *data) {
	agent->id = id;
	agent->state = AGENT_IDLE;
	agent->task_fn = task_fn;
	agent->task_data = data;

	/* 使用獨立 Stack 建立隔離 Thread */
	agent->thread = k_thread_create(
		&agent->thread, agent->stack, sizeof(agent->stack),
		agent_entry, agent, NULL, NULL,
		5,  /* priority */
		0,  /* options */
		K_FOREVER  /* 不自動啟動 */
	);

	if (!agent->thread) {
		LOG_ERR("Agent %d 建立失敗", id);
		return -ENOMEM;
	}

	return 0;
}

/* 啟動 Agent */
void nanoclaw_agent_start(nanoclaw_agent_t *agent) {
	agent->state = AGENT_RUNNING;
	k_thread_start(&agent->thread);
}

/* 等待 Agent 完成 */
int nanoclaw_agent_join(nanoclaw_agent_t *agent) {
	return k_thread_join(&agent->thread, K_FOREVER);
}

/* main - 多 Agent 協作示例 */
void sample_task(void *data) {
	int agent_id = (intptr_t)data;
	LOG_INF("Agent %d 執行推理...", agent_id);
	k_sleep(K_SECONDS(1));
	LOG_INF("Agent %d 推理完成", agent_id);
}

int main(void) {
	nanoclaw_agent_t agents[2];

	LOG_INF("NanoClaw 框架初始化");

	/* 建立 2 個隔離 Agent */
	for (int i = 0; i < 2; i++) {
		nanoclaw_agent_create(&agents[i], i, sample_task, (void *)(intptr_t)i);
	}

	/* 並行啟動 */
	for (int i = 0; i < 2; i++) {
		nanoclaw_agent_start(&agents[i]);
	}

	/* 等待所有 Agent 完成 */
	for (int i = 0; i < 2; i++) {
		nanoclaw_agent_join(&agents[i]);
	}

	LOG_INF("所有 Agent 完成");
	return 0;
}
```

## 常見問題

**Q: 如何防止一個 Agent 的故障影響其他 Agent？**
A: 每個 Agent 運行在獨立 Thread 中，擁有獨立 Stack 和資源上下文。Thread 崩潰不會波及其他 Thread，系統可檢測並隔離故障 Agent。

**Q: Agent 間如何通訊？**
A: 使用 Zephyr `k_msgq` (訊息佇列) 或 `k_sem` (信號量) 實現安全 IPC，避免直接記憶體共享。

**Q: 嵌入式系統記憶體有限，4 個 Agent 夠用嗎？**
A: 可根據 `prj.conf` 調整 `CONFIG_NANOCLAW_MAX_AGENTS` 和 `CONFIG_NANOCLAW_AGENT_STACK_SIZE`，典型配置 4-8 個 Agent，每個 2-4 KB Stack。

## API 快速參考

| 函式 | 功能 |
|------|------|
| `nanoclaw_agent_create()` | 建立隔離 Agent 容器 |
| `nanoclaw_agent_start()` | 啟動 Agent Thread |
| `nanoclaw_agent_join()` | 等待 Agent 完成 |
| `k_thread_create()` | Zephyr Thread 建立（底層） |
| `k_msgq_put()` | 發送訊息給 Agent |
| `k_msgq_get()` | Agent 接收訊息 |
