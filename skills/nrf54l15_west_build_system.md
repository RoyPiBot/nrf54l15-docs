---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: build-system
difficulty: intermediate
keywords: [west, workspace, manifest, multi-image, board, overlay, devicetree]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-02
---

# West 建構系統：專案結構、多重映像、自訂板

## 概述

West 是 Zephyr 的工作區管理和建構工具。nRF54L15 通常需要多個映像（應用核心 + 網路核心或安全核心），West 協調多次編譯、自訂板定義、和依賴管理。本文涵蓋專案佈局、多映像配置、和板客製化。

## 專案結構

典型的 nRF54L15 West 工作區結構：

```
my-nrf54l15-project/
├── west.yml                 # Manifest：定義存放庫和依賴
├── app/
│   ├── prj.conf             # 應用核心配置
│   └── src/main.c
├── net/
│   ├── prj.conf             # 網路核心配置（可選）
│   └── src/main.c
├── boards/
│   └── nrf54l15/            # 自訂板定義
│       ├── nrf54l15_cpuapp.dts
│       ├── nrf54l15_cpunet.dts
│       └── Kconfig.board
└── zephyr/                  # 自動檢出，不需提交
    └── ...
```

**west.yml 最小示例：**

```yaml
manifest:
  version: "0.13"
  projects:
    - name: zephyr
      url: https://github.com/zephyrproject-rtos/zephyr.git
      revision: v3.6.0
    - name: nrf
      url: https://github.com/nrfconnect/sdk-nrf.git
      revision: v2.6.0
  self:
    path: app
```

## 多重映像編譯

nRF54L15 可同時執行應用和網路核心。West 支援通過 `child_image` 機制來協調多次編譯。

**應用核心 prj.conf（app/prj.conf）：**

```conf
# 啟用多映像支援
CONFIG_NRF_DEFAULT_IPC_RAD=y

# 配置網路核心為子映像
CONFIG_PARTITION_MANAGER=y
CONFIG_NCS_SAMPLE_MULTICORE_SYNC=y
```

**網路核心專案（net/ 目錄）：**

```c
#include <zephyr/kernel.h>

int main(void) {
    printk("網路核心啟動\n");
    while(1) {
        k_sleep(K_MSEC(1000));
    }
    return 0;
}
```

**編譯多映像：**

```bash
# 在 app/ 目錄
west build -b nrf54l15dk/nrf54l15/cpuapp \
  -- -DCONFIG_PARTITION_MANAGER=y \
  -Dnet_BUILD=y

# 或指定網路核心專案位置
west build -b nrf54l15dk/nrf54l15/cpuapp \
  -T sample:../net
```

## 自訂板定義

在 `boards/` 目錄建立板配置，以客製化 devicetree 和 Kconfig。

**boards/nrf54l15/nrf54l15_cpuapp.dts：**

```dts
/dts-v1/;
#include <nordic/nrf54l15_cpuapp.dtsi>

/ {
    chosen {
        zephyr,console = &uart0;
        zephyr,sram = &sram0;
        zephyr,flash = &flash0;
    };
};

&uart0 {
    status = "okay";
    current-speed = <115200>;
    pinctrl-0 = <&uart0_default>;
    pinctrl-1 = <&uart0_sleep>;
    pinctrl-names = "default", "sleep";
};
```

**boards/nrf54l15/Kconfig.board：**

```kconfig
config BOARD_NRF54L15
    bool "nRF54L15 自訂板配置"
    depends on SOC_NRF54L15
    select NRF_GPIOTE if GPIO
    help
      自訂 nRF54L15 應用核心設定
```

**boards/nrf54l15/board.cmake：**

```cmake
board_runner_args(pyocd "--target=nrf54l15" "--frequency=4000000")
```

## 程式碼範例

**應用核心 main.c：**

```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/uart.h>
#include <zephyr/logging/log.h>

LOG_MODULE_REGISTER(main, CONFIG_LOG_DEFAULT_LEVEL);

int main(void) {
    // 初始化日誌系統
    LOG_INF("nRF54L15 應用核心啟動");

    // 檢查核心配置
    LOG_INF("運行於 CPUAPP");

    // 主迴圈
    while (1) {
        LOG_INF("心跳");
        k_sleep(K_SECONDS(2));
    }

    return 0;
}
```

**編譯指令：**

```bash
cd app
west build -b nrf54l15dk/nrf54l15/cpuapp
west flash
west espresso monitor  # 查看日誌輸出
```

## 常見問題

**Q1：如何指定自訂板的優先順序？**
- 在 `app/` 或專案根目錄的 `boards/` 資料夾中定義，West 會優先使用本地定義。

**Q2：多映像編譯失敗，如何除錯？**
- 使用 `west build -v` 檢視詳細編譯輸出；檢查 `net/prj.conf` 和 `app/prj.conf` 的衝突選項。

**Q3：如何控制映像燒錄順序？**
- 使用分區管理器（Partition Manager）在 `prj.conf` 中定義 `CONFIG_PARTITION_MANAGER=y`；west 會自動燒錄多映像。

## API 快速參考

| 函式 | 用途 |
|------|------|
| `west build` | 編譯當前專案或指定映像 |
| `west flash` | 燒錄韌體到開發板 |
| `west update` | 更新 manifest 中的所有存放庫 |
| `k_sleep(K_SECONDS(n))` | 延遲 n 秒 |
| `LOG_INF(fmt, ...)` | 輸出資訊級日誌 |
