---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [wireless, security]
difficulty: advanced
keywords: [Matter, Thread, smart home, commissioning, device types, interoperability]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-02
---

# nRF54L15 Matter 智慧家庭應用

## 概述

Matter 是統一的連網裝置標準，nRF54L15 通過 Matter SDK 支持智慧家庭應用開發。核心功能包括：
- **裝置類型定義**：燈、開關、鎖等標準終端設備
- **委派（Commissioning）**：安全的裝置入網流程
- **互通性**：跨品牌與多控制器支持

nRF54L15 可作為 Matter 終端設備或邊界路由器。

## Devicetree + Kconfig

### prj.conf

```conf
# Matter 基本配置
CONFIG_MATTER=y
CONFIG_CHIP_DEVICE_CONFIG_DEVICE_TYPE=65535
CONFIG_CHIP_CRYPTO_PSA=y
CONFIG_CHIP_ENABLE_UPDATING=y

# Thread（Matter 預設通訊協定）
CONFIG_OPENTHREAD_ENABLED=y
CONFIG_OPENTHREAD_FTD=y

# BLE（用於 Commissioning）
CONFIG_BT=y
CONFIG_BT_PERIPHERAL=y
CONFIG_BT_DEVICE_NAME="MatterDevice"

# 儲存
CONFIG_SETTINGS=y
CONFIG_SETTINGS_NVS=y

# 日誌
CONFIG_LOG=y
CONFIG_CHIP_LOG_LEVEL=2
```

### nrf54l15dk_nrf54l15_cpuapp.overlay

```devicetree
&nvs {
    status = "okay";
};

&rti {
    status = "okay";
};

/ {
    chosen {
        zephyr,ieee802154 = &ieee802154;
    };
};
```

## 程式碼範例：Matter 燈應用

```c
#include <zephyr/kernel.h>
#include <zephyr/logging/log.h>
#include <app/server/Server.h>
#include <app/clusters/on-off-server.h>

LOG_MODULE_REGISTER(matter_light, CONFIG_CHIP_LOG_LEVEL);

// Matter 叢集屬性回呼
static bool on_off_state = false;

void on_off_attribute_callback(chip::ClusterId clusterId,
                                chip::AttributeId attributeId,
                                uint8_t *value) {
    if (clusterId == chip::Clusters::OnOff::Id) {
        on_off_state = *value ? true : false;
        LOG_INF("燈狀態已變更：%s", on_off_state ? "開啟" : "關閉");
    }
}

// Matter 端點初始化
void setup_matter_endpoint(void) {
    chip::Server::GetInstance().Init();

    // 註冊 On/Off 叢集於端點 1
    chip::Clusters::OnOff::SetOnOffAttribute(
        1, // endpoint
        on_off_state
    );

    LOG_INF("Matter 端點已初始化，等待委派...");
}

void main(void) {
    setup_matter_endpoint();

    // 主迴圈：Matter stack 自動處理事件
    while (1) {
        k_sleep(K_SECONDS(1));
        LOG_DBG("Matter 堆疊運行中...");
    }
}
```

## 常見問題

**Q1：什麼是 Matter 委派（Commissioning）？**
A：將裝置加入 Matter 網路的安全過程。用戶掃描 QR 碼，控制器驗證裝置身份後下達網路憑據。

**Q2：nRF54L15 支持的裝置類型有哪些？**
A：支持 OnOff Light（燈）、Dimmable Light、Color Light、Smart Plug、Door Lock、Thermostat 等標準型別。在 Zephyr 裝置樹中定義 `zephyr,matter-device-type`。

**Q3：如何測試無 Thread 邊界路由器的 Commissioning？**
A：使用 Matter Controller（CHIP Tool）的 BLE Commissioning。建立 BLE 連線後，控制器通過 BLE 傳送網路憑據。

## API 快速參考

| 函式 | 說明 |
|------|------|
| `chip::Server::GetInstance().Init()` | 初始化 Matter 堆疊 |
| `chip::Clusters::OnOff::SetOnOffAttribute(endpoint, value)` | 設定 On/Off 屬性 |
| `chip::Clusters::OnOff::GetOnOffAttribute(endpoint)` | 讀取 On/Off 屬性 |
| `chip::DeviceLayer::PlatformMgr().StartEventLoopTask()` | 啟動事件迴圈 |
| `chip::Fabric::GetFabricIndex()` | 獲取委派後的 Fabric 索引 |

---

**延伸主題**：CoAP for Matter（應用層協議）、CBOR 序列化、Fabric 與安全政策。
