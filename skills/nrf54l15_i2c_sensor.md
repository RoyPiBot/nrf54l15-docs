---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [peripheral]
difficulty: [intermediate]
keywords: [I2C, 感測器, 裝置掃描, 暫存器讀寫, 多位元組操作]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-01
---

# I2C 感測器讀取

## 概述
在 nRF54L15 上透過 I2C 總線掃描、檢測和讀寫外設暫存器。支援單位元組與多位元組操作，適用於溫度、濕度、加速度計等常見感測器。

## Devicetree + Kconfig

**prj.conf**
```
CONFIG_I2C=y
CONFIG_I2C_NRFX=y
```

**overlay 檔案（nrf54l15dk.overlay）**
```devicetree
&i2c20 {
    status = "okay";
    pinctrl-0 = <&i2c20_default>;
    pinctrl-names = "default";
};

&pinctrl {
    i2c20_default: i2c20_default {
        group1 {
            psels = <NRF_PSEL(TWIM_SDA, 1, 2)>,
                    <NRF_PSEL(TWIM_SCL, 1, 3)>;
        };
    };
};
```

## 程式碼範例

```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/i2c.h>
#include <stdio.h>

#define I2C_ADDR 0x68  // MPU6050 預設位址
const struct device *i2c_dev;

// I2C 裝置掃描函式
void scan_i2c_devices(void) {
    printf("掃描 I2C 裝置...\n");
    for (uint8_t addr = 0x08; addr < 0x78; addr++) {
        struct i2c_msg msg;
        msg.buf = NULL;
        msg.len = 0;
        msg.flags = I2C_MSG_READ | I2C_MSG_STOP;

        // 嘗試讀取該位址
        if (i2c_transfer(i2c_dev, &msg, 1, addr) == 0) {
            printf("  位址: 0x%02x\n", addr);
        }
    }
}

// 讀取單位元組暫存器
uint8_t read_register(uint8_t reg) {
    uint8_t data = 0;
    struct i2c_msg msg[2];

    // 第一個訊息：寫入暫存器位址
    msg[0].buf = &reg;
    msg[0].len = 1;
    msg[0].flags = I2C_MSG_WRITE;

    // 第二個訊息：讀取資料
    msg[1].buf = &data;
    msg[1].len = 1;
    msg[1].flags = I2C_MSG_READ | I2C_MSG_STOP;

    i2c_transfer(i2c_dev, msg, 2, I2C_ADDR);
    return data;
}

// 讀取多位元組暫存器
void read_multi_bytes(uint8_t reg, uint8_t *buf, uint8_t len) {
    struct i2c_msg msg[2];

    msg[0].buf = &reg;
    msg[0].len = 1;
    msg[0].flags = I2C_MSG_WRITE;

    msg[1].buf = buf;
    msg[1].len = len;
    msg[1].flags = I2C_MSG_READ | I2C_MSG_STOP;

    i2c_transfer(i2c_dev, msg, 2, I2C_ADDR);
}

// 寫入暫存器
void write_register(uint8_t reg, uint8_t value) {
    uint8_t buf[2] = {reg, value};
    i2c_write_dt(&i2c_dev, buf, 2);
}

void main(void) {
    // 初始化 I2C 設備
    i2c_dev = device_get_binding("I2C_20");
    if (!i2c_dev) {
        printf("I2C 初始化失敗\n");
        return;
    }
    printf("I2C 初始化成功\n");

    // 掃描連接的裝置
    scan_i2c_devices();

    // 讀取感測器 ID
    uint8_t whoami = read_register(0x75);
    printf("WHO_AM_I: 0x%02x (應為 0x68)\n", whoami);

    // 讀取多位元組資料（6 位元組加速度計）
    uint8_t accel[6] = {0};
    read_multi_bytes(0x3b, accel, 6);
    printf("加速度: X=%d Y=%d Z=%d\n",
           (int16_t)((accel[0] << 8) | accel[1]),
           (int16_t)((accel[2] << 8) | accel[3]),
           (int16_t)((accel[4] << 8) | accel[5]));

    // 喚醒感測器
    write_register(0x6b, 0x00);
    printf("MPU6050 已喚醒\n");
}
```

## 常見問題

**Q1: 掃描裝置無回應**
- 檢查上拉電阻（通常 4.7kΩ）是否安裝在 SDA/SCL 線上
- 驗證 Devicetree 中的腳位定義（例：SDA=P1.02, SCL=P1.03）
- 確保感測器電源正確（VCC, GND）

**Q2: 讀寫暫存器失敗**
- 確認感測器位址正確（通常從規格書或標籤確認）
- 檢查 I2C 線路接線（無短路、無開路）
- 對感測器執行軟重置或硬重置

**Q3: 多位元組資料亂碼**
- 驗證位元組順序（大端/小端）符合感測器規格
- 確保暫存器位址的連續性（有些感測器單位元組讀取需特殊處理）
- 檢查 I2C 時脈是否超過感測器最大頻率

## API 快速參考

```c
// 掃描或轉接訊息
int i2c_transfer(const struct device *dev,
                 struct i2c_msg *msgs, uint8_t num_msgs,
                 uint16_t addr);

// 簡化讀寫（需設定 i2c_dt_spec）
int i2c_read_dt(const struct i2c_dt_spec *spec,
                uint8_t *buf, uint32_t num_bytes);

int i2c_write_dt(const struct i2c_dt_spec *spec,
                 const uint8_t *buf, uint32_t num_bytes);

// 獲取設備句柄
const struct device *device_get_binding(const char *name);
```
