---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [security]
difficulty: advanced
keywords: [TrustZone, M-Profile, 安全分區, 記憶體隔離, CMSE, NSC]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-02
---

## 概述

nRF54L15 內建 ARM M-Profile TrustZone（ARMv8-M）。將 CPU 分為**安全世界（Secure）**與**非安全世界（Non-Secure）**，透過硬體強制的記憶體隔離與控制流保護，確保敏感代碼與密鑰無法洩露。

## Devicetree + Kconfig

**prj.conf**（Secure 側）：
```conf
CONFIG_ARM_TRUSTZONE_M=y
CONFIG_ARM_TRUSTZONE_M_SECURE=y
CONFIG_STDOUT_CONSOLE=y
CONFIG_LOG=y
```

**prj.conf**（Non-Secure 側）：
```conf
CONFIG_ARM_TRUSTZONE_M=y
CONFIG_ARM_TRUSTZONE_M_NONSECURE=y
CONFIG_STDOUT_CONSOLE=y
```

Devicetree 無須修改；Zephyr 根據 Kconfig 自動管理 TrustZone 分割與 NSC（Non-Secure Callable）區域。

## 程式碼範例

**Secure 應用（cpu app）**：

```c
#include <zephyr/kernel.h>
#include <zephyr/sys/printk.h>
#include <arm_cmse.h>

/* 安全世界的敏感資料（硬體保護，只能在 Secure 存取） */
static volatile uint32_t crypto_key = 0xDEADBEEF;

/* Non-Secure 呼叫入口點
   __attribute__((cmse_nonsecure_entry)) 標記該函式可從 Non-Secure 呼叫
   Zephyr 自動生成 NSC stub */
__attribute__((cmse_nonsecure_entry))
uint32_t secure_aes_transform(uint32_t plaintext) {
    /* 只有 Secure 可直接存取 crypto_key */
    return plaintext ^ crypto_key;
}

/* 驗證位址安全屬性（可選） */
__attribute__((cmse_nonsecure_entry))
int secure_check_memory(uint32_t addr) {
    arm_cmse_address_info_t info =
        __arm_cmse_addr_read_check((void *)addr, 4, 0);
    return info.flags.secure ? 1 : 0;
}

void main(void) {
    printk("[Secure] TrustZone 安全世界已啟動\n");
    printk("[Secure] 密鑰已加載: 0x%08x\n", crypto_key);

    /* Secure 世界無限期等待 Non-Secure 呼叫 */
    while (1) {
        k_sleep(K_SECONDS(10));
        printk("[Secure] 心跳\n");
    }
}
```

**Non-Secure 應用（編譯為獨立影像）**：

```c
#include <zephyr/kernel.h>
#include <zephyr/sys/printk.h>

/* Non-Secure 宣告 Secure 函式指標
   Zephyr linker 自動建立跨界重定向 */
typedef uint32_t (*secure_aes_fn_t)(uint32_t);

void main(void) {
    printk("[NonSecure] 非安全世界已啟動\n");

    /* 取得 NSC 入口（Zephyr 提供） */
    extern uint32_t __nsc_secure_aes_transform;
    secure_aes_fn_t aes =
        (secure_aes_fn_t)&__nsc_secure_aes_transform;

    /* 跨界呼叫：CPU 切至 Secure，執行，返回 */
    uint32_t result = aes(0x12345678);
    printk("[NonSecure] 變換結果: 0x%08x\n", result);

    while (1) {
        k_sleep(K_SECONDS(5));
    }
}
```

## 常見問題

**Q: Secure 與 Non-Secure 如何通信？**
A: Non-Secure 透過 NSC（Non-Secure Callable）stub 呼叫 Secure 函式。Secure 不主動呼叫 Non-Secure；需異步通知時使用 IPC（如 RADIO TIMER）或 shared memory。

**Q: 記憶體隔離由誰保證？**
A: ARM M-Profile 硬體。每條 fetch/load/store 皆檢查 SAU（Security Attribution Unit）與 IDAU（Implementation-Defined），違反隔離規則觸發 SecureFault。

**Q: 如何除錯 Secure 代碼？**
A: 用支援 TrustZone 的除錯工具（如 nRF Command Line Tools + SoftDevice）。在 Secure 側加斷點，仍可單步執行。

## API 快速參考

| 函式/宏 | 用途 |
|---------|------|
| `__attribute__((cmse_nonsecure_entry))` | 標記函式為 NSC 入口（Non-Secure 可呼叫） |
| `__arm_cmse_addr_read_check(addr, size, flags)` | 檢查位址是否為 Secure 記憶體 |
| `__arm_cmse_addr_write_check(addr, size, flags)` | 檢查位址寫入權限 |
| `CONFIG_ARM_TRUSTZONE_M_SECURE` | 啟用 Secure 分區編譯 |
| `CONFIG_ARM_TRUSTZONE_M_NONSECURE` | 啟用 Non-Secure 分區編譯 |
