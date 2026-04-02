---
chip: nRF54L15
sdk: nRF Connect SDK (Zephyr)
category: [build-system, tool]
difficulty: [advanced]
keywords: [J-Link, nrfjprog, 量產燒錄, 批次程式設計, 程式驗證, 燒錄速度]
board: nrf54l15dk/nrf54l15/cpuapp
last_updated: 2026-04-03
---

# nRF54L15 量產燒錄：J-Link、nrfjprog、批次處理

## 概述

量產環境需要快速、可靠的固件程式設計。SEGGER J-Link 與 Nordic 的 `nrfjprog` 工具提供命令列批次介面，支援多裝置平行燒錄、自動驗證與 QA 流程整合。本文涵蓋基本設定、批次腳本、效能最佳化與常見陷阱。

## 硬體與工具準備

### J-Link 連接
- **SWD 線序**（nRF54L15DK 標準）：
  ```
  Pin 1 (VCC)  → 3.3V
  Pin 2 (GND)  → GND
  Pin 3 (SWDIO)→ SWDIO
  Pin 4 (SWCLK)→ SWCLK
  ```
- **EMU 驅動**：Windows 需 J-Link 驅動；Linux 自動識別
- **nrfjprog 安装**：
  ```bash
  # Linux/macOS
  npm install -g nrfjprog
  # 或從 Nordic 官網下載，加入 PATH
  ```

## 基礎燒錄流程

### 單次燒錄
```bash
# 查詢連接裝置
nrfjprog --com

# 燒錄 HEX 檔案
nrfjprog --program build/zephyr/merged.hex --chipversion nrf54l15

# 重設並運行
nrfjprog --reset

# 一行完成
nrfjprog --erase-all && nrfjprog --program app.hex && nrfjprog --reset
```

### 驗證與除錯
```bash
# 讀回校驗和
nrfjprog --readmemdump32 0x0 32 --save verify.bin

# 讀 UICR（User Information Configuration Register）
nrfjprog --readmemdump32 0xFFF80000 64

# 顯示詳細操作
nrfjprog --program app.hex -v
```

## 批次處理腳本

### Bash 批次燒錄範例
```bash
#!/bin/bash
# nrf54_batch_program.sh - 量產燒錄腳本

set -e
HEX_FILE="${1:-app.hex}"
DELAY=3  # 裝置切換延遲（秒）

log() { echo "[$(date +%T)] $*"; }

log "檢查 HEX 檔案: $HEX_FILE"
[[ -f "$HEX_FILE" ]] || { echo "錯誤: 找不到 $HEX_FILE"; exit 1; }

log "列出所有 J-Link 連接"
DEVICES=$(nrfjprog --com 2>/dev/null | grep "nrf54l15" | awk '{print $NF}')
DEVICE_COUNT=$(echo "$DEVICES" | wc -w)

[[ $DEVICE_COUNT -eq 0 ]] && { echo "錯誤: 未找到 nRF54L15 裝置"; exit 1; }
log "發現 $DEVICE_COUNT 個裝置"

SUCCESS=0
FAIL=0

for SN in $DEVICES; do
  log "=== 程式設計 $SN ==="
  if nrfjprog --snr "$SN" --erase-all && \
     nrfjprog --snr "$SN" --program "$HEX_FILE" && \
     nrfjprog --snr "$SN" --reset; then
    log "✓ $SN 成功"
    ((SUCCESS++))
  else
    log "✗ $SN 失敗"
    ((FAIL++))
  fi
  sleep "$DELAY"
done

log "========================================="
log "完成：成功 $SUCCESS，失敗 $FAIL，總計 $DEVICE_COUNT"
exit $([[ $FAIL -eq 0 ]] && echo 0 || echo 1)
```

### Python 多執行緒版本（加速）
```python
#!/usr/bin/env python3
# nrf_parallel_program.py
import subprocess
import threading
import sys
import time

HEX_FILE = sys.argv[1] if len(sys.argv) > 1 else "app.hex"
THREADS = 4  # 平行執行數

def program_device(snr, results, lock):
    """燒錄單一裝置"""
    try:
        subprocess.run(
            ["nrfjprog", "--snr", snr, "--erase-all"],
            check=True, capture_output=True, timeout=30
        )
        subprocess.run(
            ["nrfjprog", "--snr", snr, "--program", HEX_FILE],
            check=True, capture_output=True, timeout=60
        )
        subprocess.run(
            ["nrfjprog", "--snr", snr, "--reset"],
            check=True, capture_output=True, timeout=10
        )
        with lock:
            results[snr] = "✓"
            print(f"✓ {snr}")
    except Exception as e:
        with lock:
            results[snr] = f"✗ {e}"
            print(f"✗ {snr}: {e}")

# 取得裝置清單
devices_raw = subprocess.run(
    ["nrfjprog", "--com"], capture_output=True, text=True
).stdout
devices = [line.split()[-1] for line in devices_raw.split("\n")
           if "nrf54l15" in line.lower()]

print(f"發現 {len(devices)} 個裝置：{devices}")

results, lock = {}, threading.Lock()
threads = []

for i, snr in enumerate(devices):
    if i % THREADS == 0 and threads:
        for t in threads:
            t.join()
        threads = []

    t = threading.Thread(target=program_device, args=(snr, results, lock))
    t.start()
    threads.append(t)

for t in threads:
    t.join()

# 統計
success = sum(1 for v in results.values() if v == "✓")
print(f"\n========== 完成：{success}/{len(devices)} 成功 ==========")
sys.exit(0 if success == len(devices) else 1)
```

## 常見問題

**Q: 燒錄速度過慢？**
A: 檢查 SWD 速率。預設通常是 4 MHz。增加至 10 MHz：`--snr <SN> --clk-speed 10000`。但若線路長或干擾多，應降速至 1-2 MHz。

**Q: "Device not found" 錯誤？**
A: 確認 USB 連接、J-Link 驅動是否安裝。Linux 需 udev 規則：從 Nordic 官網下載 50-jlink.rules 至 `/etc/udev/rules.d/`。

**Q: 批次燒錄中途某裝置失敗，如何重試？**
A: 將失敗 SN 單獨儲存，於迴圈外重新程式設計。或在腳本中加 retry 邏輯。

## API 快速參考

| 命令 | 功能 | 例 |
|------|------|-----|
| `--com` | 列舉已連接裝置 | `nrfjprog --com` |
| `--snr <SN>` | 指定裝置序號 | `nrfjprog --snr 680123456 --reset` |
| `--program <HEX>` | 燒錄 HEX 檔案 | `nrfjprog --program app.hex` |
| `--erase-all` | 全晶片擦除 | `nrfjprog --erase-all` |
| `--reset` | 軟重設 | `nrfjprog --reset` |
| `--readmemdump32 <ADDR> <LEN>` | 讀記憶體 | `nrfjprog --readmemdump32 0 32` |
| `--clk-speed <kHz>` | 設定 SWD 速率 | `--clk-speed 4000` |
| `--verify` | 讀回驗證 | `nrfjprog --program app.hex --verify` |
| `--log <FILE>` | 操作記錄 | `--log program.log` |

## 效能與可靠性建議

1. **硬體**：使用優質 SWD 線（<30cm），隔離 EMI 源
2. **速率平衡**：開發環境用 4 MHz，量產批量用 10 MHz（經驗證後）
3. **自動化驗證**：每次燒錄後執行簽名校驗或首次開機測試
4. **多裝置**：平行燒錄前在單一裝置驗證韌體，再擴展
5. **日誌記錄**：`--log` 輸出用於 QA 審計

