---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [peripheral]
difficulty: [intermediate]
keywords: [TIMER, COUNTER, PPI, 延遲, 事件觸發, 計數]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-01
---

# nRF54L15 計時器與計數器：精確延遲、事件觸發、PPI 連接

## 概述

nRF54L15 提供多個硬體 TIMER 和 COUNTER 外設，用於精確延遲、事件計數和 PPI 無硬體介入連接。TIMER 基於時鐘計數，COUNTER 計算外部脈衝；兩者皆可觸發中斷、產生事件供 PPI 使用。

## Devicetree + Kconfig

**nrf54l15dk_nrf54l15_cpuapp.overlay**
```devicetree
&timer0 {
    status = "okay";
};

&timer1 {
    status = "okay";
};
```

**prj.conf**
```
CONFIG_TIMER=y
CONFIG_PWM=y
CONFIG_COUNTER=y
```

## 程式碼範例

```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/timer/system_timer.h>
#include <zephyr/drivers/counter.h>
#include <zephyr/sys/printk.h>

// 使用系統時鐘延遲的基本範例
void basic_timer_example(void) {
    // 延遲 1000 ms
    k_msleep(1000);
    printk("延遲 1 秒完成\n");
}

// 高精度延遲（使用 cycle-based API）
void precise_delay_example(void) {
    // 取得目前時鐘週期
    uint32_t start = k_cycle_get_32();

    // 延遲 100 微秒（需根據 SoC 時鐘頻率調整）
    while ((k_cycle_get_32() - start) <
           (CONFIG_SYS_CLOCK_HW_CYCLES_PER_SEC / 10000)) {
        // 忙等
    }
    printk("100 微秒延遲完成\n");
}

// Counter 事件計數範例
void counter_callback(const struct device *dev,
                      uint8_t chan_id, uint32_t ticks,
                      void *user_data) {
    printk("計數器事件觸發，次數：%u\n", ticks);
}

void counter_event_example(void) {
    const struct device *counter_dev = DEVICE_DT_GET(DT_ALIAS(counter0));

    if (!device_is_ready(counter_dev)) {
        printk("計數器設備未就緒\n");
        return;
    }

    // 設定上限值（計數到此值時觸發事件）
    struct counter_top_cfg cfg = {
        .ticks = 100,
        .callback = counter_callback,
        .user_data = NULL,
    };

    counter_set_top_value(counter_dev, &cfg);
    counter_start(counter_dev);

    k_msleep(2000);
    counter_stop(counter_dev);
    printk("計數完成\n");
}

// Timer 中斷範例
volatile uint32_t timer_count = 0;

void timer_isr(const struct device *dev, void *user_data) {
    timer_count++;
    if (timer_count % 10 == 0) {
        printk("時間中斷次數：%u\n", timer_count);
    }
}

void timer_interrupt_example(void) {
    // 此範例展示概念；實際使用需透過 PWM 或
    // 平台特定的 TIMER 驅動來設定中斷
    printk("Timer 中斷配置（取決於平台驅動支援）\n");
}

// 主程式
int main(void) {
    printk("===== nRF54L15 Timer/Counter 示範 =====\n");

    basic_timer_example();
    precise_delay_example();
    counter_event_example();
    timer_interrupt_example();

    return 0;
}
```

## 常見問題

**Q: Timer 與 Counter 的差異是什麼？**
A: TIMER 基於內部時鐘計數（用於延遲、PWM），COUNTER 計算外部脈衝輸入。選擇取決於應用場景。

**Q: 如何精確控制微秒級延遲？**
A: 使用 `k_cycle_get_32()` 和 SoS_CLOCK_HW_CYCLES_PER_SEC 計算，但應考慮中斷干擾；對超低抖動應用建議使用專用硬體計時器。

**Q: PPI 如何連接 TIMER 和其他外設？**
A: PPI 是硬體無軟體介入的事件路由機制。例如 TIMER 溢位事件可直接觸發 GPIO 切換，需透過 Devicetree 配置 PPI 通道。

## API 快速參考

| 函式 | 說明 |
|------|------|
| `k_msleep(ms)` | 毫秒級睡眠（低功耗） |
| `k_usleep(us)` | 微秒級睡眠 |
| `k_cycle_get_32()` | 取得目前時鐘週期計數 |
| `counter_start(dev)` | 啟動計數器 |
| `counter_stop(dev)` | 停止計數器 |
| `counter_set_top_value(dev, cfg)` | 設定計數上限及回呼 |

---

**相關資源：**
- nRF54L15 PS v0.3 § Timer/Counter
- nRF Connect SDK 官方文件：Timer 和 Counter 驅動
