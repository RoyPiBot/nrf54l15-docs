---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [security, build-system, tool]
difficulty: advanced
keywords: [OTA, mcuboot, 簽章驗證, 韌體更新, 回滾, DFU]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-02
---

# OTA 完整指南：mcuboot 設定、簽章驗證、回滾

## 概述

OTA（Over-The-Air）韌體更新是基於 mcuboot bootloader 的安全更新機制。nRF54L15 支援簽章驗證（ECDSA-P256）與雙槽回滾機制，確保壞韌體不會變磚。本指南涵蓋 mcuboot 配置、私鑰簽章與回滾流程。

## Devicetree + Kconfig 最小配置

### prj.conf
```
# 啟用 mcuboot 支援
CONFIG_BOOTLOADER_MCUBOOT=y
CONFIG_MCUBOOT_IMG_MANAGER=y

# 簽章驗證（ECDSA P-256）
CONFIG_MCUBOOT_VERIFY_IMG_SIGNATURE=y
CONFIG_MCUBOOT_SIGNATURE_KEY_FILE="bootloader_private.pem"

# 雙槽更新
CONFIG_MCUBOOT_UPDATE_FOOTER_ENABLED=y
CONFIG_MCUBOOT_IMAGE_VERSION="1.0.0"

# 回滾保護
CONFIG_MCUBOOT_HW_ROLLBACK_PROT=y
CONFIG_MCUBOOT_MEASURED_BOOT=n

# 應用韌體配置
CONFIG_IMG_MANAGER=y
CONFIG_FLASH_MAP=y
CONFIG_STREAM_FLASH=y
CONFIG_STREAM_FLASH_ERASE=y

# OTA DFU SMP 伺服器（應用側）
CONFIG_MCUMGR_SMP_BT=y
CONFIG_MCUMGR_CMD_IMG_MGMT=y
CONFIG_MCUMGR_CMD_OS_MGMT=y
CONFIG_MCUMGR_GRP_IMG_SUPPORTED_UPLOAD_LEN=1024
```

### nrf54l15_cpuapp.overlay（分區配置）
```devicetree
/ {
	/* 分區配置：bootloader → image-0（槽位A）→ image-1（槽位B） */
	chosen {
		nordic,pm-ext-flash = &mx25r64;
	};
};

&flash0 {
	/* 內部 Flash 分區 */
	partitions {
		compatible = "fixed-partitions";
		#address-cells = <1>;
		#size-cells = <1>;

		/* Bootloader 佔用 64 KB */
		boot_partition: partition@0 {
			label = "mcuboot";
			reg = <0x00000000 0x10000>;
		};

		/* 應用韌體槽位 A：192 KB */
		slot0_partition: partition@10000 {
			label = "image-0";
			reg = <0x10000 0x30000>;
		};

		/* 保留配置數據 */
		storage_partition: partition@40000 {
			label = "storage";
			reg = <0x40000 0x10000>;
		};
	};
};

/* 槽位 B 在外部 SPI Flash */
&mx25r64 {
	partitions {
		compatible = "fixed-partitions";
		#address-cells = <1>;
		#size-cells = <1>;

		/* 應用韌體槽位 B：192 KB */
		slot1_partition: partition@0 {
			label = "image-1";
			reg = <0x0 0x30000>;
		};
	};
};
```

## 程式碼範例

### main.c — OTA 應用與回滾邏輯
```c
#include <zephyr/kernel.h>
#include <zephyr/logging/log.h>
#include <zephyr/dfu/mcuboot.h>
#include <zephyr/sys/reboot.h>
#include <zephyr/net/socket.h>

LOG_MODULE_REGISTER(ota_app, LOG_LEVEL_INF);

/* 檢查並標記更新完成（提交槽位 B 的韌體） */
static int confirm_update(void) {
	int ret;

	/* 檢查是否有待確認的新韌體 */
	if (!boot_is_img_confirmed()) {
		LOG_INF("確認新韌體映像...");
		ret = boot_write_img_confirmed();
		if (ret) {
			LOG_ERR("確認失敗: %d", ret);
			return ret;
		}
		LOG_INF("新韌體已確認，下次重啟後成為活躍韌體");
	}
	return 0;
}

/* 監聽來自 DFU 客戶端的新韌體並驗證簽章 */
static int handle_ota_update(const uint8_t *fw_data, size_t fw_size) {
	/*
	 * 實際應用中，使用 MCUmgr SMP over BLE 或 HTTP
	 * 此處僅示意簽章驗證已在 mcuboot 層執行
	 * fw_data 寫入到槽位 B，mcuboot 自動驗證簽章
	 */
	LOG_INF("新韌體 %zu bytes，mcuboot 將驗證簽章", fw_size);
	return 0;
}

/* 觸發回滾機制（若新韌體無法工作） */
static int trigger_rollback(void) {
	LOG_WRN("觸發回滾到上一版本...");
	/* 清除確認標誌，mcuboot 將啟動槽位 A 的備份韌體 */
	int ret = boot_erase_img_bank(FLASH_AREA_IMAGE_1);
	if (ret) {
		LOG_ERR("清除槽位 B 失敗: %d", ret);
	}
	sys_reboot(SYS_REBOOT_COLD);
	return 0;
}

/* 主應用進入點 */
void main(void) {
	LOG_INF("=== nRF54L15 OTA 韌體更新應用 ===");
	LOG_INF("版本: 1.0.0");

	/* 確認當前韌體有效 */
	if (confirm_update() == 0) {
		LOG_INF("韌體狀態正常");
	}

	/* 模擬檢查是否需要回滾（例如應用無法啟動） */
	int health_check = 1;  // 若設為 0 則觸發回滾
	if (!health_check) {
		trigger_rollback();
	}

	/* 應用主迴圈 */
	while (1) {
		LOG_INF("應用執行中...");
		k_sleep(K_SECONDS(5));
	}
}
```

## 常見問題

**Q: 如何生成簽章用的私鑰？**
A: 使用 `imgtool genkey -k bootloader_private.pem -t ecdsa-p256 -e pem`。須安全保管此私鑰；公鑰已嵌入 mcuboot。

**Q: 回滾如何工作？**
A: 雙槽設計（A/B）：A 為活躍，B 為待驗證。若 B 簽章驗證失敗或應用 crash，mcuboot 自動啟動 A。需 `CONFIG_MCUBOOT_UPDATE_FOOTER_ENABLED=y`。

**Q: 如何在 BLE 上傳輸新韌體？**
A: 啟用 `CONFIG_MCUMGR_SMP_BT=y` 與 `CONFIG_MCUMGR_CMD_IMG_MGMT=y`，使用 nRF Connect for Mobile 或自訂 DFU 客戶端上傳。

## API 快速參考

```c
/* 檢查當前韌體映像是否已確認 */
bool boot_is_img_confirmed(void);

/* 確認當前槽位的新韌體為有效狀態 */
int boot_write_img_confirmed(void);

/* 清除指定槽位的韌體（用於回滾） */
int boot_erase_img_bank(uint32_t bank);

/* 獲取當前啟動分區資訊 */
int boot_current_slot(void);
```
