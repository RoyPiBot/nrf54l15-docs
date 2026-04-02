---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [peripheral]
difficulty: [intermediate]
keywords: [interrupt, NVIC, priority, preemption, zero-latency]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-02
---

# 中斷優先順序：NVIC 設定、零延遲中斷、搶占

## 概述
NVIC（巢狀向量化中斷控制器）管理 nRF54L15 的所有中斷優先級。Zephyr 支援零延遲中斷（優先級 -1，永不搶占）與普通中斷搶占，可實現即時響應。

## Devicetree + Kconfig

**prj.conf** — 啟用優先級配置：
```
CONFIG_NUM_IRQS=96
CONFIG_ZERO_LATENCY_IRQS=1
```

無需額外 .overlay（NVIC 由 Zephyr 自動初始化）。

## 代碼示例

```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/gpio.h>
#include <zephyr/irq.h>

static const struct device *gpio_dev;
static const uint8_t btn_pin = 8;

// 零延遲中斷（優先級 -1，永不被搶占）
void timer0_isr(const void *arg)
{
	// 關鍵即時任務：馬上執行，不可中斷
	printk("零延遲: 定時器中斷\n");
}

// 普通優先級中斷（可被搶占）
void button_isr(const struct device *dev,
                struct gpio_callback *cb,
                uint32_t pins)
{
	printk("普通: 按鈕中斷\n");
}

// GPIO 按鈕中斷初始化
void init_button_interrupt(void)
{
	gpio_dev = DEVICE_DT_GET(DT_NODELABEL(gpio0));
	if (!device_is_ready(gpio_dev)) {
		printk("GPIO 未就緒\n");
		return;
	}

	// 配置引腳為輸入
	gpio_pin_configure(gpio_dev, btn_pin,
	                   GPIO_INPUT | GPIO_PULL_UP);
	gpio_pin_interrupt_configure(gpio_dev, btn_pin,
	                              GPIO_INT_EDGE_FALLING);

	// 註冊中斷，優先級 0（可被優先級 -1 搶占）
	static struct gpio_callback btn_cb;
	gpio_init_callback(&btn_cb, button_isr, BIT(btn_pin));
	gpio_add_callback(gpio_dev, &btn_cb);
}

// 註冊零延遲中斷（TIMER0）
void init_timer_zero_latency(void)
{
	// IRQ_ZERO_LATENCY：優先級固定 -1，永不搶占
	IRQ_CONNECT(TIMER0_IRQn,
	            0,  // 相對優先級（忽略，因為已是 -1）
	            timer0_isr,
	            NULL,
	            IRQ_ZERO_LATENCY);
	irq_enable(TIMER0_IRQn);
}

void main(void)
{
	printk("nRF54L15 中斷優先順序演示\n");
	init_button_interrupt();
	init_timer_zero_latency();

	while (1) {
		k_sleep(K_MSEC(1000));
	}
}
```

## 常見問題

**Q: 零延遲中斷與普通中斷的區別？**
A: 零延遲中斷（IRQ_ZERO_LATENCY，優先級 -1）永不被搶占，執行時間最短。普通中斷（優先級 ≥ 0）可被更高優先級搶占。

**Q: 如何避免優先級反轉？**
A: 使用 Zephyr 互斥鎖（k_mutex）在共享資源上同步，或提高存取資源的中斷優先級。

**Q: 零延遲中斷是否支援中斷巢狀？**
A: 是，零延遲中斷可打斷普通中斷，但普通中斷無法打斷零延遲中斷。

## API 快速參考

| 函式 | 說明 |
|------|------|
| `IRQ_CONNECT(irq, pri, isr, arg, flags)` | 連接 ISR；flags 可為 `IRQ_ZERO_LATENCY` |
| `irq_enable(irq)` | 啟用中斷 |
| `irq_disable(irq)` | 禁用中斷 |
| `irq_lock()` / `irq_unlock(key)` | 臨界段保護 |
