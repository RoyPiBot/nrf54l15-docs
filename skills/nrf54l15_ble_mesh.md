---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [wireless]
difficulty: [intermediate]
keywords: [Bluetooth Mesh, Provisioning, Models, Publish, Subscribe]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-02
---

# Bluetooth Mesh 網狀網路

Bluetooth Mesh 在 nRF54L15 上提供低功耗網狀聯網能力，支援多對多通訊、自動中繼、以及基於角色的節點架構（Provisioning、Relay、Friend）。本文涵蓋節點配置、模型定義、訊息傳遞的核心實作。

## Devicetree 與 Kconfig

**prj.conf：**
```
# Bluetooth & Mesh
CONFIG_BT=y
CONFIG_BT_OBSERVER=y
CONFIG_BT_BROADCASTER=y
CONFIG_BT_MESH=y
CONFIG_BT_MESH_RELAY=y
CONFIG_BT_MESH_FRIEND=y
CONFIG_BT_MESH_MODEL_GROUP_COUNT=4

# 日誌
CONFIG_LOG=y
CONFIG_BT_LOG_LEVEL_DBG=y
```

**nrf54l15dk_nrf54l15_cpuapp.overlay：**
```devicetree
&radio {
	status = "okay";
};

&timer0 {
	status = "okay";
};
```

## 程式碼範例

```c
#include <zephyr/kernel.h>
#include <zephyr/bluetooth/bluetooth.h>
#include <zephyr/bluetooth/mesh.h>

#define MOD_LF 0x0000  // 通用 On/Off 伺服模型 (LightFixture)

// 定義模型
static struct bt_mesh_model models[] = {
	BT_MESH_MODEL_GEN_ONOFF_SRV(&onoff_srv),
};

static struct bt_mesh_elem elements[] = {
	BT_MESH_ELEM(0, models, BT_MESH_MODEL_NONE),
};

static const struct bt_mesh_comp comp = {
	.cid = CONFIG_BT_COMPANY_ID,
	.pid = 0x0001,
	.vid = 0x0001,
	.elem = elements,
	.elem_count = ARRAY_SIZE(elements),
};

// 節點預配置狀態指示
static void mesh_ready(void)
{
	printk("Mesh 節點已就緒\n");
}

// 藍牙初始化回調
static void bt_ready(int err)
{
	if (err) {
		printk("藍牙初始化失敗 (err %d)\n", err);
		return;
	}

	// 初始化 Mesh
	err = bt_mesh_init(
		&bt_mesh_prov,  // Provisioning 模式設置
		&comp
	);
	if (err) {
		printk("Mesh 初始化失敗 (err %d)\n", err);
		return;
	}

	// 如果節點未預配，進入 beacon 模式等待
	if (!bt_mesh_is_provisioned()) {
		printk("等待預配...\n");
		bt_mesh_prov_enable();
	} else {
		mesh_ready();
		// 啟動 Mesh (掃描、監聽訊息)
		err = bt_mesh_start();
		if (err) {
			printk("Mesh 啟動失敗 (err %d)\n", err);
		}
	}
}

// 預配完成回調
static void prov_complete(uint16_t net_idx, uint16_t addr)
{
	printk("節點已預配，地址: 0x%04x\n", addr);
	mesh_ready();
	bt_mesh_start();
}

// 預配數據讀取
static void prov_input_complete(void)
{
	printk("預配輸入完成\n");
}

// Provisioning 組態
static const struct bt_mesh_prov bt_mesh_prov = {
	.uuid = NULL,  // 使用預設 UUID
	.oob_info = 0,
	.input_complete = prov_input_complete,
	.complete = prov_complete,
};

// 訊息處理與發佈範例
static int send_onoff_status(uint16_t addr)
{
	struct bt_mesh_msg_ctx ctx = {
		.net_idx = bt_mesh_default_sub_get(),
		.app_idx = bt_mesh_default_app_idx_get(),
		.addr = addr,
		.send_ttl = BT_MESH_TTL_DEFAULT,
	};

	uint8_t status = 1;  // On 狀態

	// 發布訊息到指定地址
	return bt_mesh_model_send(
		&models[0],
		&ctx,
		NULL,
		&status, 1
	);
}

// 應用主程式
int main(void)
{
	int err;

	printk("nRF54L15 Bluetooth Mesh 節點\n");

	// 啟用藍牙
	err = bt_enable(bt_ready);
	if (err) {
		printk("藍牙啟用失敗 (err %d)\n", err);
		return err;
	}

	// 處理預配與網狀網路事件
	while (1) {
		k_sleep(K_SECONDS(10));
		// 定期檢查 Mesh 狀態或發送心跳
	}

	return 0;
}
```

## 常見問題

**Q: 預配失敗，顯示「timeout」？**
A: 確認 Provisioner（配置器）和 Node 都啟用了藍牙掃描。檢查 `CONFIG_BT_MESH_PROV_TIMEOUT` 和電源供應。

**Q: 節點收不到訊息？**
A: 驗證訂閱群組位址是否正確配置。使用 `bt_mesh_model_subscribe()` 綁定接收群組；發佈端確認發向同一群組。

**Q: 中繼模式無法啟用？**
A: 設置 `CONFIG_BT_MESH_RELAY=y` 且預配節點必須啟用中繼功能（Relay 功能位元）。

## API 快速參考

```c
// 初始化 Mesh
int bt_mesh_init(const struct bt_mesh_prov *prov,
                 const struct bt_mesh_comp *comp);

// 啟動網狀網路
int bt_mesh_start(void);

// 發佈訊息
int bt_mesh_model_send(struct bt_mesh_model *model,
                       struct bt_mesh_msg_ctx *ctx,
                       struct net_buf_simple *buf,
                       const void *data, size_t len);

// 訂閱群組
int bt_mesh_model_subscribe(struct bt_mesh_model *model,
                            uint16_t addr);

// 預配使能
void bt_mesh_prov_enable(void);

// 檢查預配狀態
bool bt_mesh_is_provisioned(void);
```
