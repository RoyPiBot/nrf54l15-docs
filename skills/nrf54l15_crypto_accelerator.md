---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: security
difficulty: advanced
keywords: [AES, SHA, ECC, Crypto Accelerator, Hardware Acceleration, Side-Channel Protection]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-02
---

# nRF54L15 硬體加密加速器：AES/SHA/ECC 操作與側通道防護

## 概述

nRF54L15 內置 **CRACEN**（Crypto Accelerator Engine），提供硬體加速的密碼學操作：
- **AES-128/256**：ECB、CBC、CTR、GCM 模式
- **SHA-256/512**：訊息摘要運算
- **ECC P-256**：ECDSA、ECDH 密鑰協商

相比軟體實現，硬體加速提供 **10-50 倍性能提升** 與更強的 **側通道防護**（恆時運算、隨機化）。

## Devicetree 配置

在 `nrf54l15dk_nrf54l15_cpuapp.dts` 或 `.overlay` 中啟用：

```dts
/* 硬體加密加速器已內置，無需額外配置 */
/* 僅需在 Kconfig 中啟用驅動程式 */
```

## Kconfig 配置

在 `prj.conf` 中設定：

```ini
# 啟用 PSA Crypto 框架（推薦）
CONFIG_PSA_CRYPTO_DRIVER_CRACEN=y
CONFIG_PSA_CRYPTO_CLIENT=y

# 或使用 Zephyr Crypto API
CONFIG_CRYPTO=y
CONFIG_CRYPTO_MBEDTLS_AES=y
CONFIG_CRYPTO_MBEDTLS_SHA256=y

# 側通道防護
CONFIG_PSA_CRYPTO_HW_ACCELERATION=y
CONFIG_PSA_CRYPTO_RANDOMIZED_OPERATIONS=y
```

## 程式碼範例

```c
#include <zephyr/kernel.h>
#include <zephyr/sys/printk.h>
#include <psa/crypto.h>

/* AES-256 ECB 加密範例（使用 PSA Crypto） */
int main(void) {
    /* 初始化 PSA Crypto */
    psa_status_t status = psa_crypto_init();
    if (status != PSA_SUCCESS) {
        printk("PSA Crypto 初始化失敗: %d\n", status);
        return -1;
    }

    /* AES-256 密鑰（示例，勿用於產品環境） */
    uint8_t key[32] = {
        0x00, 0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07,
        0x08, 0x09, 0x0a, 0x0b, 0x0c, 0x0d, 0x0e, 0x0f,
        0x10, 0x11, 0x12, 0x13, 0x14, 0x15, 0x16, 0x17,
        0x18, 0x19, 0x1a, 0x1b, 0x1c, 0x1d, 0x1e, 0x1f
    };

    /* 匯入密鑰到硬體 */
    psa_key_attributes_t attributes = PSA_KEY_ATTRIBUTES_INIT;
    psa_set_key_usage_flags(&attributes, PSA_KEY_USAGE_ENCRYPT);
    psa_set_key_algorithm(&attributes, PSA_ALG_ECB_NO_PADDING);
    psa_set_key_type(&attributes, PSA_KEY_TYPE_AES);
    psa_set_key_bits(&attributes, 256);

    psa_key_id_t key_id;
    status = psa_import_key(&attributes, key, sizeof(key), &key_id);
    if (status != PSA_SUCCESS) {
        printk("密鑰匯入失敗: %d\n", status);
        return -1;
    }

    /* 待加密的訊息（16 位元組明文） */
    uint8_t plaintext[16] = {
        0x00, 0x11, 0x22, 0x33, 0x44, 0x55, 0x66, 0x77,
        0x88, 0x99, 0xaa, 0xbb, 0xcc, 0xdd, 0xee, 0xff
    };
    uint8_t ciphertext[16];
    size_t ciphertext_len;

    /* 執行加密（硬體加速） */
    status = psa_cipher_encrypt(key_id, PSA_ALG_ECB_NO_PADDING,
                                plaintext, sizeof(plaintext),
                                ciphertext, sizeof(ciphertext),
                                &ciphertext_len);

    if (status == PSA_SUCCESS) {
        printk("加密成功！密文: ");
        for (int i = 0; i < 16; i++) {
            printk("%02x ", ciphertext[i]);
        }
        printk("\n");
    }

    /* 清理資源 */
    psa_destroy_key(key_id);
    mbedtls_psa_crypto_free();
    return 0;
}
```

## 常見問題

**Q1: 如何驗證是否使用了硬體加密？**
A: 編譯時啟用 `CONFIG_PSA_CRYPTO_DRIVER_CRACEN=y`，執行時會自動使用硬體。可透過效能對比（軟體 vs 硬體）確認。

**Q2: 側通道防護的實現方式？**
A: CRACEN 採用 **恆時運算**（Constant-Time Operations）與 **隨機化** 機制防止時序攻擊和功耗分析，無需軟體介入。

**Q3: 支援哪些密鑰大小？**
A: AES 支援 128/192/256 位；ECC 支援 P-256；SHA 支援 256/512 位。詳見 PSA Crypto 規範。

## API 快速參考

| 函式 | 用途 |
|------|------|
| `psa_crypto_init()` | 初始化 PSA Crypto 與硬體加速器 |
| `psa_import_key()` | 將密鑰匯入硬體安全儲存 |
| `psa_cipher_encrypt()` | 硬體加速的對稱加密 |
| `psa_hash_compute()` | 硬體加速的訊息摘要（SHA-256/512） |
| `psa_sign_message()` | 硬體加速的 ECDSA 簽署 |
| `psa_destroy_key()` | 清理硬體中的密鑰 |
