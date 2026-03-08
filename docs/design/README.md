# NoC Behavior Model - Design Documents

本目錄包含 NoC Behavior Model 的設計規格文件。

---

## 文件索引

### Architecture Overview

| # | 文件 | 說明 |
|---|------|------|
| 00 | [System Architecture](00_architecture.md) | **入口文件** — 全局架構圖、data flow、軟硬體介面、I/O pattern、co-sim |

### Core — 01~04

| # | 文件 | 說明 |
|---|------|------|
| 01 | [System Overview](01_overview.md) | 拓撲參數、固定設計參數 |
| 02 | [Flit Format](02_flit.md) | **基準文件** — 固定 408-bit flit、Header/Payload 格式 |
| 03 | [Router](03_router.md) | Ports、XY Routing、Wormhole、Pipeline、Multicast、Reduction |
| 04 | [Network Interface](04_network_interface.md) | NMU/NSU、AXI ↔ Flit 轉換、RoB、ECC |

### Architecture — 05~08

| # | 文件 | 說明 |
|---|------|------|
| 05 | [Physical Channel](05_physical_channel.md) | 雙通道 Req/Rsp 架構、HoL Blocking 分析 |
| 06 | [QoS Design](06_qos.md) | QoS 優先級設計、Generator、Probe |
| 07 | [Memory Operations](07_memory_operations.md) | Host-to-NoC 傳輸 |
| 08 | [Multicast](08_multicast.md) | Rectangle Multicast 設計 |

### Simulation & Verification — 09~10

| # | 文件 | 說明 |
|---|------|------|
| 09 | [Simulation Platform](09_simulation.md) | NocConfig、API (5 Groups)、Cycle Model、Channel\<T\>、Hot-Swap、DPI-C |
| 10 | [Verification](10_verification.md) | Scoreboard、效能指標、測試策略、Coverage |

### Appendix

| # | 文件 | 說明 |
|---|------|------|
| A2 | [Width Converter](A2_width_converter.md) | AXI 寬度轉換（32b~1024b ↔ 256b） |

---

## 圖片資源

| 檔案 | 內容 |
|------|------|
| [platform_diagram.svg](../images/platform_diagram.svg) | 平台全局圖（User Interface / NocSystem Internals / Golden Verification） |
| [hot_swap.svg](../images/hot_swap.svg) | Hot-Swap 概念圖（C++ ↔ RTL 切換） |
