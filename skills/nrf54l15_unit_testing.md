---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [build-system, tool]
difficulty: intermediate
keywords: [Zephyr Test Framework, 單元測試, 模擬硬體, mocking, host testing]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-03
---

# nRF54L15 單元測試框架

## 概述
Zephyr Test Framework (ZTF) 是 Zephyr RTOS 內建的測試框架，支援設備測試（device testing）與主機測試（host testing）。nRF54L15 可利用 cmock 進行硬體模擬，加速開發週期。

## Devicetree + Kconfig

**prj.conf** — 測試專案基本設定
```ini
CONFIG_ZTEST=y
CONFIG_ZTEST_MOCKING=y
CONFIG_ZTEST_ASSERT_VERBOSE=y
CONFIG_CMOCK_LIB=y
```

**board.overlay** — 可選，若測試無需特定硬體
```devicetree
&gpio0 {
  status = "okay";
};
```

## 程式碼範例

**tests/unit/test_sample.c** — 完整可編譯的測試範例
```c
#include <zephyr/ztest.h>
#include <cmock.h>
#include "../src/sample.h"

/* 被測函式宣告（原程式） */
int add(int a, int b);
void led_init(void);
int led_write(int pin, int state);

/* Mock 函式定義 */
int __wrap_led_write(int pin, int state) {
    check_expected(pin);
    check_expected(state);
    return 0;
}

/* 測試套件：算術運算 */
ZTEST_SUITE(arithmetic, NULL, NULL, NULL, NULL);

ZTEST(arithmetic, test_add_positive) {
    int result = add(2, 3);
    zassert_equal(result, 5, "2+3 should equal 5");
}

ZTEST(arithmetic, test_add_negative) {
    int result = add(-1, 1);
    zassert_equal(result, 0, "-1+1 should equal 0");
}

/* 測試套件：LED 控制（帶 mock） */
ZTEST_SUITE(led_control, NULL, NULL, NULL, NULL);

ZTEST(led_control, test_led_on) {
    /* 期望呼叫 led_write(5, 1) */
    expect_value(__wrap_led_write, pin, 5);
    expect_value(__wrap_led_write, state, 1);

    led_init();
    int ret = led_write(5, 1);
    zassert_equal(ret, 0, "LED write should succeed");
}

/* 快速檢查宏 */
ZTEST(arithmetic, test_boundaries) {
    zassert_between_inclusive(add(10, 10), 19, 21, "Boundary check");
}
```

**CMakeLists.txt** — 測試編譯設定
```cmake
cmake_minimum_required(VERSION 3.20)
project(nrf54l15_tests)

find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
project(unit_tests)

target_sources(app PRIVATE tests/unit/test_sample.c ../src/sample.c)
```

## 常見問題

**Q: 如何測試中斷處理器？**
A: 使用 `ztest_expect_value()` 與 cmock，模擬 ISR 呼叫，或在主機測試環境中隔離業務邏輯。

**Q: GPIO/SPI 等週邊如何 mock？**
A: 在 `CMakeLists.txt` 中以 `__wrap_` 前綴包裝驅動函式，或使用 Zephyr 的 fake driver API。

**Q: 如何執行測試？**
A: `west build -b nrf54l15dk/nrf54l15/cpuapp -t run` 或 `twister -T tests/` 進行批量執行。

## API 快速參考

| 函式 | 說明 |
|------|------|
| `ZTEST(suite, test_name)` | 定義測試函式 |
| `zassert_equal(a, b, msg)` | 相等斷言 |
| `zassert_true(cond, msg)` | 真值斷言 |
| `expect_value(func, param, val)` | 期望函式參數值（cmock） |
| `check_expected(param)` | 驗證期望值（cmock） |
| `ZTEST_SUITE(name, setup, teardown, before_all, after_all)` | 定義測試套件 |

**編譯 & 執行：**
```bash
cd nrf54l15-project
west build -b nrf54l15dk/nrf54l15/cpuapp tests/unit
west flash
# 或 host 測試（不需燒錄）
west build -b native_posix tests/unit && ./build/zephyr/zephyr.exe
```
