---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [peripheral]
difficulty: [intermediate]
keywords: [RRAM, NVS, 非揮發性存儲, 設定檔持久化, Flash Storage]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-01
---

# RRAM 儲存操作：NVS 非揮發性存儲、設定檔持久化

## 概述
NVS（Non-Volatile Storage）是 Zephyr 提供的鍵值對儲存系統，直接操作 RRAM（可程式化隨機存取記憶體）。用於設定檔、校準參數、日誌等需要持久化的小型資料。

## Devicetree + Kconfig

**nrf54l15dk_nrf54l15_cpuapp.overlay:**
```dtsi
/ {
    nvs_storage: nvs_storage {
        compatible = "fixed-partitions";
        #address-cells = <1>;
        #size-cells = <1>;

        nvs_partition: partition@f4000 {
            label = "nvs_storage";
            reg = <0xf4000 0x6000>;
        };
    };
};
```

**prj.conf:**
```ini
CONFIG_NVS=y
CONFIG_NVS_LOOKUP_CACHE=y
CONFIG_NVS_CACHE_SIZE=512
CONFIG_FLASH=y
CONFIG_FLASH_MAP=y
```

## 程式碼範例

```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/flash.h>
#include <zephyr/fs/nvs.h>
#include <stdio.h>

static struct nvs_fs nvs;

void nvs_init(void)
{
    // 初始化 NVS 系統
    struct flash_area *fa;
    const struct flash_area *fap = flash_area_image_open(FLASH_AREA_ID(nvs_storage));
    if (!fap) {
        printf("NVS Flash Area 開啟失敗\n");
        return;
    }

    nvs.flash_device = fap->dev;
    nvs.offset = fap->fa_off;
    nvs.sector_size = fap->fa_size;
    nvs.sector_count = fap->fa_size / fap->fa_size;

    int ret = nvs_init(&nvs, DT_CHOSEN_ZEPHYR_FLASH_CONTROLLER_LABEL);
    if (ret) {
        printf("NVS 初始化失敗: %d\n", ret);
    } else {
        printf("NVS 初始化成功\n");
    }
}

void nvs_write_param(uint16_t key, const char *value)
{
    // 寫入字串參數
    int ret = nvs_write_string(&nvs, key, value);
    if (ret) {
        printf("NVS 寫入失敗 (key=%u): %d\n", key, ret);
    } else {
        printf("NVS 寫入成功: %s\n", value);
    }
}

void nvs_read_param(uint16_t key, char *buffer, size_t len)
{
    // 讀取字串參數
    ssize_t ret = nvs_read_string(&nvs, key, buffer, len);
    if (ret < 0) {
        printf("NVS 讀取失敗 (key=%u): %zd\n", key, ret);
    } else {
        printf("NVS 讀取成功: %s\n", buffer);
    }
}

void nvs_write_data(uint16_t key, const void *data, size_t len)
{
    // 寫入二進制資料
    int ret = nvs_write(&nvs, key, data, len);
    if (ret > 0) {
        printf("NVS 二進制寫入: %d 位元組\n", ret);
    }
}

void main(void)
{
    nvs_init();

    // 測試：寫入參數
    nvs_write_param(1, "device_name=MyDevice");

    // 讀取參數
    char buffer[64];
    nvs_read_param(1, buffer, sizeof(buffer));

    // 二進制資料範例
    uint32_t counter = 12345;
    nvs_write_data(2, &counter, sizeof(counter));
}
```

## 常見問題

**Q: NVS 的存儲容量有多大？**
A: 受限於分配的 Flash 分區大小（上例為 24KB）。實際可用空間因管理開銷約減少 10-15%。

**Q: 重複寫入同一 Key 會發生什麼？**
A: NVS 會覆寫舊值。無需先刪除；寫入時自動處理。

**Q: 斷電時資料會遺失嗎？**
A: 不會。NVS 直接寫入 Flash，Power-Loss Safe 設計確保完整性（除非在寫入操作中途斷電，該項可能遺失）。

## API 快速參考

| 函式 | 描述 |
|------|------|
| `nvs_init(fs, dev)` | 初始化 NVS 系統 |
| `nvs_write(fs, key, data, len)` | 寫入二進制資料 |
| `nvs_read(fs, key, data, len)` | 讀取二進制資料 |
| `nvs_write_string(fs, key, str)` | 寫入字串 |
| `nvs_read_string(fs, key, buf, len)` | 讀取字串 |
| `nvs_delete(fs, key)` | 刪除指定 Key |
