---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [security, build-system, tool]
difficulty: advanced
keywords: [DFU, bootloader, secure-boot, OTA, firmware-update, image-management]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-02
---

# DFU 韌體更新：安全啟動、OTA 更新、映像管理

## 概述

DFU (Device Firmware Update) 機制允許遠端或本機安全地更新裝置韌體。nRF54L15 支援安全啟動 (Secure Boot) 和 OTA 更新，透過簽名驗證和雙映像管理確保韌體完整性。

## Devicetree + Kconfig

**prj.conf：**
```
CONFIG_BOOTLOADER_MCUBOOT=y
CONFIG_MCUBOOT_IMG_MANAGER=y
CONFIG_IMG_MANAGER=y
CONFIG_DFU_MCUBOOT=y
CONFIG_SECURE_BOOT=y
CONFIG_BUILD_OUTPUT_BIN=y
CONFIG_FLASH_LOG=y
```

**app.overlay（可選 DFU 分區）：**
```devicetree
&mram1x {
  erase-block-size = <4096>;
  partitions {
    compatible = "fixed-partitions";
    #address-cells = <1>;
    #size-cells = <1>;

    boot_partition: partition@0 {
      label = "mcuboot";
      reg = <0x0 0x8000>;
    };

    slot0_partition: partition@8000 {
      label = "image-0";
      reg = <0x8000 0x40000>;
    };

    slot1_partition: partition@48000 {
      label = "image-1";
      reg = <0x48000 0x40000>;
    };

    storage_partition: partition@88000 {
      label = "storage";
      reg = <0x88000 0x18000>;
    };
  };
};
```

## 程式碼範例

**DFU 韌體更新與確認（Zephyr）：**
```c
#include <zephyr/kernel.h>
#include <zephyr/dfu/mcuboot.h>
#include <zephyr/devicetree.h>
#include <zephyr/logging/log.h>

LOG_MODULE_REGISTER(dfu_demo, LOG_LEVEL_INF);

// DFU 完成後確認新韌體，標記為永久啟動
int dfu_confirm_image(void) {
    // 要求 mcuboot 將此映像標記為有效
    if (boot_write_img_confirmed()) {
        LOG_ERR("韌體確認失敗");
        return -1;
    }
    LOG_INF("韌體已確認為永久映像");
    return 0;
}

// 查詢當前啟動映像與其他映像狀態
void dfu_check_status(void) {
    // 檢查是否從交換啟動（OTA 更新後首次啟動）
    if (mcuboot_is_img_confirmed()) {
        LOG_INF("映像已確認");
    } else {
        LOG_INF("映像待確認（可能是新 OTA 更新）");
    }
}

// 觸發重啟以啟動另一個映像（手動切換）
void dfu_trigger_swap(void) {
    // 在 mcuboot 分區中標記要求交換
    int ret = boot_request_upgrade(BOOT_UPGRADE_TEST);
    if (ret == 0) {
        LOG_INF("交換已請求，重啟後生效");
        k_sleep(K_SECONDS(1));
        sys_reboot(SYS_REBOOT_COLD);
    } else {
        LOG_ERR("交換請求失敗：%d", ret);
    }
}

void main(void) {
    LOG_INF("DFU 範例啟動");

    // 應用初始化後驗證韌體
    dfu_check_status();

    // 若系統正常，標記映像為永久有效
    dfu_confirm_image();

    LOG_INF("系統執行中");
}
```

## 常見問題

**Q1：什麼是 MCUBoot？**
A：MCUBoot 是 Zephyr 內建的 Bootloader，負責驗證簽名、管理映像交換和安全啟動。nRF54L15 使用 MCUBoot 實現 OTA 更新。

**Q2：如何簽名與上傳新韌體？**
A：使用 `imgtool` 簽名（`west sign -t imgtool -- --key key.pem`），再透過 OTA 通訊協議（如 FOTA over BLE/MQTT）上傳至 `image-1` 分區。

**Q3：映像確認失敗會發生什麼？**
A：若未呼叫 `boot_write_img_confirmed()`，MCUBoot 將在下次啟動時回滾至舊映像。這保護了有問題的新韌體。

## API 快速參考

| 函式 | 說明 |
|------|------|
| `boot_write_img_confirmed()` | 標記當前映像為永久有效 |
| `mcuboot_is_img_confirmed()` | 檢查映像是否已確認 |
| `boot_request_upgrade(type)` | 請求映像交換（BOOT_UPGRADE_TEST 或 PERMANENT） |
| `sys_reboot(SYS_REBOOT_COLD)` | 冷重啟 |
| `imgtool` (west CLI) | 簽名與驗證應用程式映像 |
