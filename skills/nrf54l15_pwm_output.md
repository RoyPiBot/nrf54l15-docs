---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [peripheral]
difficulty: [beginner]
keywords: [PWM, 脈衝寬度調變, 頻率, 佔空比, 多通道, LED, TIMER]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-01
---

# PWM 輸出控制：頻率設定、佔空比調整、多通道控制

## 概述
PWM（脈衝寬度調變）用於控制平均功率輸出，常應用於 LED 調光、馬達速度控制等場景。nRF54L15 透過 TIMER 模組和比較功能實現 PWM，Zephyr API 提供統一的 `pwm_*()` 函式族進行跨平台操作。

## Devicetree + Kconfig

**prj.conf**
```
CONFIG_PWM=y
CONFIG_PWM_NRF_TIMER=y
```

**boards/nrf54l15dk_nrf54l15_cpuapp.overlay**
```devicetree
/ {
  pwms {
    compatible = "pwm-leds";
    pwm_led0: pwm_led_0 {
      pwms = <&pwm0 0 PWM_MSEC(20) PWM_POLARITY_NORMAL>;
      label = "PWM LED 0";
    };
    pwm_led1: pwm_led_1 {
      pwms = <&pwm0 1 PWM_MSEC(20) PWM_POLARITY_NORMAL>;
      label = "PWM LED 1";
    };
  };
};

&pwm0 {
  status = "okay";
  prescaler = <0>;
};
```

## 程式碼範例

```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/pwm.h>
#include <zephyr/logging/log.h>

LOG_MODULE_REGISTER(pwm_demo, LOG_LEVEL_INF);

/* 取得 PWM 裝置指標 */
static const struct pwm_dt_spec pwm_led0 =
    PWM_DT_SPEC_GET_BY_IDX(DT_PATH(pwms, pwm_led0), 0);
static const struct pwm_dt_spec pwm_led1 =
    PWM_DT_SPEC_GET_BY_IDX(DT_PATH(pwms, pwm_led1), 0);

int main(void)
{
  /* 驗證 PWM 裝置就緒 */
  if (!device_is_ready(pwm_led0.dev)) {
    LOG_ERR("PWM LED0 device not ready");
    return -1;
  }
  if (!device_is_ready(pwm_led1.dev)) {
    LOG_ERR("PWM LED1 device not ready");
    return -1;
  }

  LOG_INF("PWM demo started");

  uint32_t period_us = PWM_MSEC(20); /* 50Hz */
  uint32_t pulse_us;

  while (1) {
    /* 漸進增加 LED0 亮度：0% → 100% */
    for (int i = 0; i <= 100; i += 10) {
      pulse_us = (period_us * i) / 100;
      pwm_set_pulse_dt(&pwm_led0, pulse_us);
      k_msleep(500);
    }

    /* 漸進降低 LED0 亮度：100% → 0% */
    for (int i = 100; i >= 0; i -= 10) {
      pulse_us = (period_us * i) / 100;
      pwm_set_pulse_dt(&pwm_led0, pulse_us);
      k_msleep(500);
    }

    /* LED1 固定 50% 佔空比 */
    pwm_set_pulse_dt(&pwm_led1, period_us / 2);

    k_msleep(2000);
  }

  return 0;
}
```

## 常見問題

**Q1: 如何改變 PWM 頻率？**
> 頻率 = 1 / 周期。在 Zephyr 中，周期由 `period_us` 決定。例如 `PWM_MSEC(20)` = 20ms = 50Hz。使用 `pwm_set_dt()` 同時更新周期和脈衝寬度。

**Q2: 為什麼 PWM 輸出不穩定？**
> 確認 prescaler、TIMER 模組狀態，以及 Devicetree 中的 TIMER 別名配置。執行 `west build -t menuconfig` 驗證 `CONFIG_PWM_NRF_TIMER` 已啟用。

**Q3: 能否在運行時改變多個通道的頻率？**
> 可以。若多通道共用同一 TIMER，需同時改變，否則會互相干擾。建議為每個通道使用獨立 TIMER，或透過上層應用層調度。

## API 快速參考

| 函式 | 用途 |
|------|------|
| `pwm_set_pulse_dt(const struct pwm_dt_spec *spec, uint32_t pulse)` | 設定脈衝寬度（保持周期不變） |
| `pwm_set_dt(const struct pwm_dt_spec *spec, uint32_t period, uint32_t pulse)` | 同時設定周期和脈衝寬度 |
| `pwm_get_cycles_per_sec(const struct device *dev, uint32_t *cycles_sec)` | 取得 PWM 時鐘頻率 |
| `device_is_ready(const struct device *dev)` | 驗證設備初始化完成 |
