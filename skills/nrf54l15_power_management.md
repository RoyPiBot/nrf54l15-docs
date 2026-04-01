---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [power]
difficulty: [intermediate]
keywords: [睡眠, 喚醒源, 電流最適化, IDLE, SLEEP, RTC, GPIO]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-01
---

# nRF54L15 低功耗管理

## 概述
nRF54L15 支援多種睡眠模式（IDLE、SLEEP）及靈活的喚醒源（GPIO、RTC、UART）。通過 Zephyr PM 架構可輕鬆實現毫瓦級功耗。

## Devicetree + Kconfig

**build/nrf54l15dk_nrf54l15_cpuapp.overlay**
```dts
&gpio0 {
  wakeup-source;
};

&rtc0 {
  status = "okay";
};
```

**prj.conf**
```
CONFIG_PM=y
CONFIG_PM_DEVICE=y
CONFIG_SYS_PM_POLICY_DEFAULT=y
CONFIG_SYS_PM_POLICY_DEVICE=y
CONFIG_RTC=y
CONFIG_GPIO=y
CONFIG_COUNTER=y
```

## 程式碼範例

```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/gpio.h>
#include <zephyr/drivers/counter.h>
#include <zephyr/pm/pm.h>

/* 定義 GPIO 節點（按鈕腳位） */
static const struct gpio_dt_spec button =
  GPIO_DT_SPEC_GET(DT_NODELABEL(sw0), gpios);

/* RTC 節點 */
static const struct device *const rtc_dev =
  DEVICE_DT_GET(DT_NODELABEL(rtc0));

/* 喚醒中斷回調 */
static void button_callback(const struct device *dev,
                            struct gpio_callback *cb,
                            uint32_t pins)
{
  printk("按鈕喚醒！\n");
}

int main(void)
{
  int ret;
  struct gpio_callback button_cb;

  if (!gpio_is_ready_dt(&button)) {
    printk("GPIO 設備未就緒\n");
    return -1;
  }

  /* 配置 GPIO 為輸入並設為喚醒源 */
  ret = gpio_pin_configure_dt(&button, GPIO_INPUT);
  if (ret < 0) return ret;

  gpio_init_callback(&button_cb, button_callback, BIT(button.pin));
  ret = gpio_add_callback(button.port, &button_cb);
  if (ret < 0) return ret;

  ret = gpio_pin_interrupt_configure_dt(&button,
                                        GPIO_INT_EDGE_TO_ACTIVE);
  if (ret < 0) return ret;

  printk("裝置進入睡眠，等待 GPIO 喚醒...\n");

  while (1) {
    /* CPU 進入睡眠，中斷喚醒 */
    pm_state_force(0, &(struct pm_state_info){0});
    k_sleep(K_FOREVER);
  }

  return 0;
}
```

## 常見問題

**Q1: 如何區分 IDLE 和 SLEEP 模式？**
IDLE 保持時鐘，SLEEP 關閉時鐘，電流更低但喚醒延遲更長。Zephyr PM 自動選擇。

**Q2: GPIO 喚醒無效怎麼辦？**
確保 Devicetree 中 GPIO 設了 `wakeup-source`，且中斷配置為 `GPIO_INT_EDGE_TO_ACTIVE`。

**Q3: RTC 可以喚醒嗎？**
可以，使用 `counter_set_channel_alarm()` 設定 RTC 觸發，會喚醒 CPU。

## API 快速參考

| 函式 | 說明 |
|------|------|
| `pm_state_force(idx, &info)` | 強制進入指定 PM 狀態 |
| `gpio_pin_interrupt_configure_dt(&spec, flags)` | 配置 GPIO 中斷（喚醒源） |
| `counter_set_channel_alarm(dev, ch, cfg)` | 設定 RTC 鬧鐘喚醒 |
| `k_sleep(duration)` | CPU 睡眠指定時間 |
| `sys_pm_force_power_state()` | 手動設定電源狀態 |
