---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [peripheral]
difficulty: [intermediate]
keywords: [USB, CDC-ACM, HID, 虛擬串口, 鍵盤, 滑鼠, USB 裝置]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-02
---

# nRF54L15 USB 裝置模式

## 概述

nRF54L15 內建 USB 2.0 全速（FS）裝置控制器，支援多種 USB 類別：CDC-ACM（虛擬串口）、HID（人機介面）。
在 Zephyr 框架下，無需手動操作暫存器，利用 USB 中間層與驅動程式即可實現即插即用的 USB 設備。

## Devicetree + Kconfig

**nrf54l15dk_nrf54l15_cpuapp.overlay**
```dts
/ {
    chosen {
        zephyr,console = &uart0;
        zephyr,code-partition = &slot0_partition;
    };

    usb_device: zephyr,udc {
        status = "okay";
    };
};

&usbd {
    status = "okay";
};
```

**prj.conf**
```conf
CONFIG_USB_DEVICE_STACK=y
CONFIG_USB_DEVICE_VID=0x1915
CONFIG_USB_DEVICE_PID=0x521F
CONFIG_USB_DEVICE_MANUFACTURER="Nordic"
CONFIG_USB_DEVICE_PRODUCT="nRF54L15-USB"
CONFIG_USB_DEVICE_SERIAL_NUMBER_CODEPOINT_COUNT=12

# CDC-ACM（虛擬串口）
CONFIG_USB_DEVICE_CLASS_CDC_ACM=y
CONFIG_USB_CDC_ACM_DEVICE_COUNT=1

# HID（鍵盤滑鼠）
CONFIG_USB_DEVICE_CLASS_HID=y
CONFIG_USB_HID_DEVICE_COUNT=2
```

## 程式碼範例

```c
#include <zephyr/kernel.h>
#include <zephyr/usb/usb_device.h>
#include <zephyr/usb/class/usb_hid.h>
#include <zephyr/drivers/uart.h>
#include <zephyr/logging/log.h>

LOG_MODULE_REGISTER(usb_demo, LOG_LEVEL_INF);

// HID 鍵盤報告描述符
static const uint8_t hid_report_desc[] = {
    0x05, 0x01,        // 用途頁：通用桌面
    0x09, 0x06,        // 用途：鍵盤
    0xa1, 0x01,        // 集合（應用）
    0x75, 0x01,        // 報告大小：1 位
    0x95, 0x08,        // 報告計數：8
    0x05, 0x07,        // 用途頁：鍵盤/按鍵
    0x19, 0xe0,        // 用途最小值
    0x29, 0xe7,        // 用途最大值
    0x15, 0x00,        // 邏輯最小值：0
    0x25, 0x01,        // 邏輯最大值：1
    0x81, 0x02,        // 輸入（資料）
    0x75, 0x08,        // 報告大小：8 位
    0x95, 0x06,        // 報告計數：6
    0x15, 0x00,        // 邏輯最小值：0
    0x25, 0xe7,        // 邏輯最大值：231
    0x05, 0x07,        // 用途頁：鍵盤/按鍵
    0x19, 0x00,        // 用途最小值
    0x29, 0xe7,        // 用途最大值
    0x81, 0x00,        // 輸入（陣列）
    0xc0,              // 結束集合
};

static const struct usb_hid_config hid_config = {
    .report_desc = hid_report_desc,
    .report_desc_size = sizeof(hid_report_desc),
    .report_size = 8,
    .idle_timeout = 0,
};

// 按鍵報告：修飾符 + 保留 + 鍵碼(6個)
static uint8_t key_report[8] = {0, 0, 0, 0, 0, 0, 0, 0};

void main(void)
{
    int ret;

    LOG_INF("USB 裝置初始化中...");

    // 初始化 USB 裝置堆棧
    ret = usb_enable(NULL);
    if (ret != 0) {
        LOG_ERR("USB 初始化失敗: %d", ret);
        return;
    }

    LOG_INF("USB 裝置已啟用");

    // 主迴圈：模擬按鍵事件（每 2 秒發送 'A' 鍵）
    while (1) {
        k_sleep(K_MSEC(2000));

        // 設置 'A' 鍵的鍵碼（HID 鍵碼：0x04）
        key_report[2] = 0x04;
        ret = hid_int_ep_write(0, key_report, sizeof(key_report), NULL);
        if (ret != 0) {
            LOG_WRN("鍵盤報告發送失敗: %d", ret);
        }

        k_sleep(K_MSEC(100));

        // 清除鍵碼（按鍵釋放）
        key_report[2] = 0x00;
        hid_int_ep_write(0, key_report, sizeof(key_report), NULL);
    }
}
```

## 常見問題

**Q1: CDC-ACM 設備在 Linux/macOS 上無法辨認？**
確認 Kconfig 中 `CONFIG_USB_DEVICE_CLASS_CDC_ACM=y`，且 USB 線路正常。在 Linux 上執行 `dmesg | grep -i usb` 檢查日誌。

**Q2: HID 鍵盤報告無反應？**
檢查報告描述符格式是否符合 HID 規範。報告大小需與 `CONFIG_USB_HID_REPORT_SIZE` 一致；確認主機偵測到設備。

**Q3: USB 掛起時如何喚醒？**
實現 `CONFIG_USB_DEVICE_REMOTE_WAKEUP` 與遠程喚醒端點。檢查功耗狀態是否允許喚醒信號。

## API 快速參考

| 函式 | 說明 |
|------|------|
| `usb_enable(const struct usb_cfg_data *cfg)` | 初始化 USB 裝置堆棧 |
| `hid_int_ep_write(uint8_t id, const uint8_t *buf, size_t len, int *bytes_written)` | 發送 HID 報告 |
| `usb_disable()` | 關閉 USB 裝置 |
| `usb_is_configured()` | 檢查 USB 配置狀態 |
| `usb_device_set_data(struct usb_dev_data *dev_data)` | 設定驅動程式私有資料 |
