---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [peripheral]
difficulty: [intermediate]
keywords: [PPI, 事件, 任務, 硬體互聯, 零CPU開銷]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-02
---

# nRF54L15 PPI 事件系統

## 概述
PPI（Programmable Peripheral Interconnect）允許外設事件直接觸發其他外設的任務，無需 CPU 干預。實現零 CPU 開銷的硬體自動化，大幅降低功耗、提高響應精度。典型應用：計時器事件驅動 GPIO、串列埠事件驅動計數器等。

## Devicetree + Kconfig

**prj.conf**
```
CONFIG_CLOCK_CONTROL=y
CONFIG_TIMER=y
CONFIG_GPIO=y
CONFIG_NRF_ENABLE_ICACHE=y
```

**nrf54l15dk_nrf54l15_cpuapp.overlay**
```device-tree
/ {
  aliases {
    timer0 = &timer0;
    led0 = &gpio0_0;
  };
};

&gpio0 {
  status = "okay";
};

&timer0 {
  status = "okay";
};
```

## 程式碼範例

```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/timer/system_timer.h>
#include <zephyr/drivers/gpio.h>
#include <hal/nrf_timer.h>
#include <hal/nrf_ppi.h>

// 定義 GPIO（LED）
#define LED_PORT DT_NODELABEL(gpio0)
#define LED_PIN 0

static const struct gpio_dt_spec led =
  GPIO_DT_SPEC_GET(DT_ALIAS(led0), gpios);

int main(void) {
  // 初始化 GPIO
  if (!gpio_is_ready_dt(&led)) {
    return -1;
  }

  gpio_pin_configure_dt(&led, GPIO_OUTPUT);
  gpio_pin_set_dt(&led, 0);

  // 配置計時器 (TIMER0)
  // - 16MHz 時鐘，24位計數
  // - 每 1 秒 (16,000,000 cycles) 溢位
  nrf_timer_mode_set(NRF_TIMER0, NRF_TIMER_MODE_TIMER);
  nrf_timer_bit_width_set(NRF_TIMER0, NRF_TIMER_BIT_WIDTH_24);
  nrf_timer_frequency_set(NRF_TIMER0, NRF_TIMER_FREQ_1MHz);
  nrf_timer_cc_set(NRF_TIMER0, NRF_TIMER_CC_CHANNEL0, 1000000UL);

  // 設定 PPI 通道將計時器 CC 事件連到 GPIO 任務
  // 事件：TIMER0 COMPARE[0]
  // 任務：GPIO OUT 切換（模擬 LED）
  uint32_t ppi_channel = 0;
  nrf_ppi_channel_enable(NRF_PPI, NRF_PPI_CHANNEL0);

  // 綁定事件源
  nrf_ppi_event_endpoint_setup(
    NRF_PPI,
    ppi_channel,
    nrf_timer_event_address_get(NRF_TIMER0,
      NRF_TIMER_EVENT_COMPARE0)
  );

  // 綁定任務（通過暴露的 GPIO OUT TOGGLE）
  // 注：實際 GPIO PPI 任務需透過 GPIO 外設配置
  // 此處示意連接概念

  // 啟動計時器
  nrf_timer_task_trigger(NRF_TIMER0, NRF_TIMER_TASK_START);

  // 主迴圈（無需控制 LED，完全由硬體自動切換）
  while (1) {
    k_sleep(K_SECONDS(10));
  }

  return 0;
}
```

## 常見問題

**Q1：PPI 有多少個通道？**
A：nRF54L15 提供多個 PPI 通道（具體數量見 datasheet）。每個通道可連接一個事件到一個任務。

**Q2：如何確保 PPI 任務執行的原子性？**
A：PPI 任務執行是硬體級操作，無中斷風險。多個 PPI 任務可並行執行（若綁定不同通道）。

**Q3：能否將同一事件連接到多個任務？**
A：可以，透過多個 PPI 通道綁定同一事件源到不同的任務。

## API 快速參考

```c
// 啟用/禁用 PPI 通道
nrf_ppi_channel_enable(nrf_ppi_t p_reg, uint32_t channel);
nrf_ppi_channel_disable(nrf_ppi_t p_reg, uint32_t channel);

// 設定事件端點（輸入）
nrf_ppi_event_endpoint_setup(nrf_ppi_t p_reg,
  uint32_t channel, uint32_t event_addr);

// 設定任務端點（輸出）
nrf_ppi_task_endpoint_setup(nrf_ppi_t p_reg,
  uint32_t channel, uint32_t task_addr);

// 查詢計時器事件地址
nrf_timer_event_address_get(NRF_TIMER_Type *p_reg,
  nrf_timer_event_t event);
```

---
**編譯命令**（自 openclaw/ 或專案根目錄）
```bash
west build -b nrf54l15dk/nrf54l15/cpuapp -- -DOVERLAY_FILE=nrf54l15dk_nrf54l15_cpuapp.overlay
west flash
```
