---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [wireless]
difficulty: [advanced]
keywords: [Thread, Border Router, CoAP, Multi-hop, IPv6, 6LoWPAN, OpenThread]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-02
---

# Thread 網路協定：邊界路由器、CoAP 通訊、多跳

## 概述
Thread 是基於 IEEE 802.15.4 的自組網狀網路協定，原生支援 IPv6 通訊。nRF54L15 可作為邊界路由器（連接 Thread 與外部網路）、路由器或終端節點，通過 CoAP 傳輸應用資料，自動支援多跳路由與自癒。

## Devicetree + Kconfig

**prj.conf（最小配置）**
```
CONFIG_OPENTHREAD_ENABLED=y
CONFIG_OPENTHREAD_FTD=y
CONFIG_NET_IPV6=y
CONFIG_NET_COAP=y
CONFIG_OPENTHREAD_BORDER_ROUTER=y
CONFIG_NET_L2_OPENTHREAD=y
CONFIG_NET_UDP=y
```

**boards/nrf54l15dk_nrf54l15_cpuapp.overlay**
```dts
/ {
  chosen {
    zephyr,ieee802154 = &radio;
  };
};
```

## 程式碼範例

### Thread 節點 + CoAP 伺服器

```c
#include <zephyr/kernel.h>
#include <zephyr/net/openthread.h>
#include <zephyr/net/coap.h>
#include <openthread/thread.h>

// CoAP 資源：查詢節點狀態
static int coap_get_status(const struct coap_resource *resource,
                           struct coap_client_request *request) {
  // 建立回應封包
  static uint8_t resp_buf[256];
  int resp_len = snprintf((char *)resp_buf, sizeof(resp_buf),
    "{\"status\":\"online\",\"role\":\"child\"}");

  coap_send_response(request, resp_buf, resp_len,
                     COAP_RESPONSE_CODE_CONTENT);
  return 0;
}

// CoAP 資源列表
static const struct coap_resource coap_resources[] = {
  { .path = "status", .get = coap_get_status }
};

// 初始化 OpenThread
void thread_setup(void) {
  otInstance *instance = openthread_get_default_instance();

  // 啟用 IPv6 及 Thread
  otIp6SetEnabled(instance, true);
  otThreadSetEnabled(instance, true);

  // 設定為 MTD（Minimal Thread Device）
  otSetDeviceRole(instance, OT_DEVICE_ROLE_CHILD);

  printk("Thread 已啟動\n");
}

// 初始化 CoAP 伺服器
void coap_setup(void) {
  coap_server_init();

  for (size_t i = 0; i < ARRAY_SIZE(coap_resources); i++) {
    coap_add_resource(&coap_resources[i]);
  }
  printk("CoAP 伺服器就緒\n");
}

int main(void) {
  thread_setup();
  coap_setup();

  // 主迴圈
  while (1) {
    k_sleep(K_SECONDS(10));
  }
  return 0;
}
```

## 常見問題

**Q: 邊界路由器的角色？**
A: 邊界路由器連接 Thread 網路與外部 IPv4/6 網路（如 Wi-Fi），實現跨邊界通訊。在 prj.conf 設定 `CONFIG_OPENTHREAD_BORDER_ROUTER=y` 並使用 FTD（Full Thread Device）模式。

**Q: 如何驗證多跳路由工作？**
A: 使用 OpenThread CLI 指令 `route table` 查看路由表；部署 3 個以上節點，觀察較遠節點是否經由中間節點轉發訊息。

**Q: CoAP 與 Bluetooth Mesh 有何區別？**
A: CoAP 基於 UDP，Thread 網路原生支援 IPv6；Mesh 為蘋果 HomeKit 方案。Thread 更適合智慧家居標準應用（Matter）。

## API 快速參考

```c
// OpenThread 核心 API
otInstance *openthread_get_default_instance(void);
otThreadSetEnabled(otInstance *aInstance, bool aEnabled);
otIp6SetEnabled(otInstance *aInstance, bool aEnabled);
otSetDeviceRole(otInstance *aInstance, otDeviceRole aRole);
otIp6GetUnicastAddresses(otInstance *aInstance);

// CoAP API
coap_server_init(void);
coap_add_resource(const struct coap_resource *resource);
coap_send_response(struct coap_client_request *request,
                   const uint8_t *payload, size_t payload_len,
                   uint8_t response_code);
```
