# Nordic nRF54L15 超詳細開發文件庫

> 🤖 由 RoyPiBot (Raspberry Pi 5 上的 Claude) 自動產生
> 📅 每 30 分鐘更新一份新文件
> 🎯 目標：讓任何人或 AI Agent 都能快速上手 nRF54L15 開發

## 📁 目錄結構

```
nrf54l15-docs/
├── skills/       # AI-friendly 技能文件（可直接被 AI Agent 使用）
├── guides/       # 完整開發指南（含步驟、程式碼、說明）
├── examples/     # 完整可編譯的範例專案
├── references/   # API 參考與規格整理
├── nanoclaw/     # NanoClaw + nRF54L15 整合方案
└── .state/       # 佇列與狀態管理
```

## 🔵 nRF54L15 晶片概覽

| 特性 | 規格 |
|------|------|
| 主處理器 | ARM Cortex-M33 @ 128 MHz |
| 協處理器 | RISC-V @ 128 MHz |
| 記憶體 | 1.5 MB RRAM + 256 KB RAM |
| 無線 | BLE 5.4/6.0, Thread, Zigbee, Matter |
| 製程 | TSMC 22nm ULL |
| 封裝 | 2.4 x 2.2 mm WLCSP |
| SDK | nRF Connect SDK (Zephyr RTOS) |

## 🦀 NanoClaw 整合

NanoClaw 是輕量級 AI Agent 框架，可用於：
- 自動化 nRF54L15 韌體開發流程
- 作為 IoT 閘道器的 AI 控制層
- 提供可被 AI Agent 消費的 Skill 文件

## 📊 產生進度

文件由 cron 排程每 30 分鐘自動產生，推送到 GitHub。

### 已完成的 Skill 文件

| 檔案 | 主題 | 大小 |
|------|------|------|
| `skills/nrf54l15_gpio_control.md` | GPIO 基礎操作：輸入輸出、中斷、上下拉電阻 | ~43 KB |
| `skills/nrf54l15_ble_advertising.md` | BLE 廣播：設定廣播封包、掃描回應、間隔 | ~48 KB |
| `skills/nrf54l15_ble_connection.md` | BLE 連線：GATT Server/Client、特徵值讀寫 | ~48 KB |
| `skills/nrf54l15_uart_serial.md` | UART 串口通訊：波特率、DMA、收發資料 | ~4 KB |

### 待產生主題（佇列中）
- PWM 控制、SPI/I2C 通訊、Timer 計時器
- BLE Mesh、Thread/Zigbee 網路
- 低功耗模式、OTA 韌體更新
- NanoClaw 整合方案

## 🛠️ 技術棧

- **文件產生**：Claude (Anthropic) on Raspberry Pi 5
- **目標 SDK**：nRF Connect SDK (Zephyr RTOS)
- **自動化**：cron + bash scripts
- **版本控制**：Git + GitHub (RoyPiBot/nrf54l15-docs)
