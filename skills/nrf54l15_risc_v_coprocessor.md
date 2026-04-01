---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [peripheral, power]
difficulty: advanced
keywords: [RISC-V, 協處理器, 任務卸載, IPC, 郵箱, 低功耗]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-02
---

# RISC-V 協處理器：任務卸載、IPC 通訊、效能優化

## 概述

nRF54L15 內建 RISC-V 低功耗協處理器，可獨立執行感測、信標掃描等輕量任務，與主核心（ARM Cortex-M33）通過郵箱（mailbox）IPC 通訊。卸載耗電任務至協處理器可降低主核心功耗，達到整體系統功耗最優。

## Devicetree + Kconfig

### nrf54l15_cpuapp.dts (節選)
```devicetree
/ {
  chosen {
    zephyr,ipc_mbox = &mbox;
  };
};

&mbox {
  status = "okay";
};

&cpuppr {
  status = "okay";
};
```

### prj.conf (主核心)
```
CONFIG_IPC_DRIVER=y
CONFIG_MBOX=y
CONFIG_MBOX_NRFX_MBOX=y
CONFIG_MAIN_STACK_SIZE=2048
CONFIG_SYSTEM_WORKQUEUE_STACK_SIZE=2048
```

### cpupr/prj.conf (協處理器)
```
CONFIG_CORTEX_M_GENERIC=y
CONFIG_SYS_CLOCK_HW_CYCLES_PER_SEC=64000000
```

## 程式碼範例：任務卸載與 IPC 通訊

### main.c (主核心 - cpuapp)
```c
#include <zephyr/kernel.h>
#include <zephyr/ipc/ipc_service.h>
#include <zephyr/logging/log.h>

LOG_MODULE_REGISTER(app);

static struct ipc_ep_sample {
  struct ipc_ept ept;
} ep_sample;

/* IPC 訊息結構 */
struct ipc_msg {
  uint32_t cmd;        /* 1: 啟動協處理器任務, 2: 取得結果 */
  uint32_t sensor_id;  /* 感測器編號 */
  uint32_t data;       /* 資料欄位 */
};

static void ipc_ept_receive_callback(const void *data, size_t len, void *priv)
{
  const struct ipc_msg *msg = (const struct ipc_msg *)data;
  LOG_INF("收到協處理器回應: cmd=%u, data=%u", msg->cmd, msg->data);
}

void main(void)
{
  int ret;
  const struct ipc_config config = {
    .cb = ipc_ept_receive_callback,
    .priv = &ep_sample,
  };

  /* 開啟 IPC 端點 */
  ret = ipc_service_open_instance(&ep_sample.ept, &config);
  if (ret != 0) {
    LOG_ERR("IPC 開啟失敗: %d", ret);
    return;
  }

  /* 發送任務給協處理器: 讀取感測器 ID=1 */
  struct ipc_msg cmd_msg = {
    .cmd = 1,
    .sensor_id = 1,
    .data = 0,
  };

  ret = ipc_service_send(&ep_sample.ept, &cmd_msg, sizeof(cmd_msg));
  if (ret < 0) {
    LOG_ERR("IPC 發送失敗: %d", ret);
  }

  /* 主核心可進入低功耗狀態 */
  while (1) {
    k_sleep(K_SECONDS(10));
  }
}
```

## 常見問題

**Q: 協處理器如何獨立運行？**
A: 協處理器執行專用映像檔（cpupr/），可獨立於主核心，透過郵箱通訊協調。

**Q: IPC 郵箱延遲多少？**
A: 典型 < 100µs（取決於負載），足以支援即時感測與控制。

**Q: 卸載至協處理器省電多少？**
A: 主核心可進深度睡眠（uA 級功耗），比持續運行省 70-90%。

## API 快速參考

```c
/* 開啟 IPC 端點 */
int ipc_service_open_instance(struct ipc_ept *ept,
                               const struct ipc_config *cfg);

/* 發送訊息 */
int ipc_service_send(struct ipc_ept *ept, const void *data, size_t len);

/* 註冊接收回調 */
/* 已在 ipc_config.cb 設定 */

/* 進入睡眠（讓協處理器運行） */
void k_sleep(k_timeout_t timeout);
```

## 參考資源

- [nRF54L15 PS v1.0](https://infocenter.nordicsemi.com/topic/ps_nrf54l15/power_management.html)
- [IPC Service (Zephyr Docs)](https://docs.zephyrproject.org/latest/services/ipc/ipc_service.html)
- [nRF Connect SDK 郵箱驅動](https://sdk.nordicsemi.com/p/nrf-connect-sdk/latest/doc/nrf/references/drivers/mbox.html)
