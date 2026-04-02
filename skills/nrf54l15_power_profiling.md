---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [power, tool]
difficulty: intermediate
keywords: [功耗, PPK2, 電流量測, 功耗優化, DCDC, 待機模式]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-03
---

# nRF54L15 功耗量測與優化

## 概述
使用 nRF PowerProfiler Kit 2（PPK2）進行精確的電流量測與功耗分析，並通過 Kconfig 調優實現低功耗應用。支援靜態漏電、待機、活躍等工作模式的功耗基準測試。

## PPK2 連接與配置

### 硬體連接
- **GND** → nRF54L15 DK P3 GND
- **VDD** → nRF54L15 DK P3 VDD (3.3V)
- **TP (Target Power)** → DK 電源隔離點
- 使用 Micro USB 連線至電腦

### nRF Connect Desktop 設定
1. 開啟 Power Profiler 2 應用程式
2. 選擇 Segger J-Link 連線
3. 採樣率設定為 1 kHz（推薦平衡精度與性能）
4. 勾選「Voltage Regulator Mode」監控 DCDC 狀態

## Devicetree + Kconfig

### prj.conf
```
# 基礎電源管理
CONFIG_PM=y
CONFIG_PM_DEVICE=y

# 低功耗藍牙（若使用）
CONFIG_BT_LL_SW_SPLIT=y
CONFIG_BT_RX_STACK_SIZE=2560

# DCDC 轉換器優化
CONFIG_DCDC_ENABLED=y
CONFIG_DCDC_MODE=1

# 棧大小優化
CONFIG_SYSTEM_WORKQUEUE_STACK_SIZE=2048

# 可選：Shell 用於調試
CONFIG_SERIAL_SHELL=y
```

### nrf54l15dk_nrf54l15_cpuapp.overlay
```devicetree
/ {
	power-states {
		idle: idle {
			compatible = "zephyr,power-state";
			power-state-name = "IDLE";
			min-residency-us = <10000>;
		};
		suspend: suspend {
			compatible = "zephyr,power-state";
			power-state-name = "SUSPEND_TO_RAM";
			min-residency-us = <100000>;
		};
	};
};
```

## 程式碼範例

```c
#include <zephyr/kernel.h>
#include <zephyr/sys/printk.h>
#include <zephyr/pm/pm.h>

// 工作迴圈：活躍 → 睡眠 → 喚醒
int main(void) {
	printk("PowerProfiler Demo\n");

	// 確認電源管理啟用
	#ifdef CONFIG_PM
		printk("PM enabled\n");
	#endif

	for (int cycle = 0; cycle < 10; cycle++) {
		// 活躍期：模擬工作（GPIO 翻轉或運算）
		printk("Cycle %d: Active...\n", cycle);
		for (volatile int i = 0; i < 10000; i++);  // 忙等 ~1ms

		// 睡眠期：進入自動待機狀態
		// Zephyr kernel 會自動選擇最低功耗狀態
		k_msleep(5000);  // 在 PPK2 上觀察電流脈衝
	}

	return 0;
}
```

**編譯與燒錄**
```bash
west build -p always -b nrf54l15dk/nrf54l15/cpuapp
west flash
```

## 常見問題

**Q: 如何區分靜態漏電與動態消耗？**
> 分別量測「完全待機無中斷」（基準線）和「有定時器中斷」（脈衝），差值即為軟體開銷。使用 Power Profiler CSV 匯出並在試算表中分析。

**Q: DCDC 模式相比 LDO 省電多少？**
> DCDC 效率 ~90%，LDO ~70%。啟用 `CONFIG_DCDC_MODE=1` 可減少 20-30% 功耗。DK 內建 TPS62762，自動切換 DCDC/LDO。

**Q: PPK2 測量與實際應用不符？**
> 檢查清單：(1) 外部元件電源是否隔離（SD 卡、傳感器）、(2) 軟體無意中持續觸發中斷、(3) 使用 Power Profiler 內建的 1Hz 濾波取平均值。

## API 快速參考

```c
// 電源狀態管理
pm_state_force(state);                    // 強制進入指定狀態

// 裝置級電源控制
pm_device_suspend(dev);                   // 暫停裝置
pm_device_resume(dev);                    // 恢復裝置
pm_device_state_get(dev, &state);         // 查詢狀態

// 自動睡眠
k_msleep(duration_ms);                    // kernel 自動選擇最低功耗狀態

// 中斷喚醒源（待機時使用）
irq_enable(NRF_RTC0_IRQ);                 // 啟用 RTC 中斷喚醒
```

## 測試流程

1. **靜態功耗** (10–20 µA)
   - 刻錄空白韌體，關閉所有活動
   - 記錄 1 分鐘平均電流

2. **待機+定時喚醒** (30–100 µA)
   - 啟用 RTC，每秒觸發一次 ISR
   - 在 Power Profiler 上觀察規則脈衝

3. **活躍模式** (5–10 mA)
   - GPIO 快速翻轉，記錄峰值與均值

4. **資料分析**
   - 在 Power Profiler 匯出 CSV
   - 計算不同狀態的平均電流與峰值電流
