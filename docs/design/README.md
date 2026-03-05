# NoC Behavior Model - 設計文件

本目錄包含 NoC Behavior Model 的設計規格文件。

---

## 架構圖

系統整體架構見 [noc_env.txt](../images/noc_env.txt)（ASCII 架構圖）。

---

## 文件索引

### 核心元件

| 序號 | 文件 | 說明 |
|------|------|------|
| 01 | [系統概述](01_overview.md) | V1 架構圖、拓撲參數、固定設計參數 |
| 02 | [Router 規格](02_router.md) | Ports、XY Routing、Wormhole Arbiter、**CppRouter 內部 Pipeline Stages** |
| 03 | [Network Interface 規格](03_network_interface.md) | NMU、NSU、資料路徑、**CppNI 內部 Functions** |
| 04 | [Flit 格式](04_flit.md) | **基準文件** — 固定 408-bit flit 設計、Header/Payload 格式、Physical Link |
| 05 | [內部介面架構](05_internal_interface.md) | Credit-Based Flow Control、Port Interface、**Channel\<T\>、TrafficManager、Allocator** |

### 系統行為

| 序號 | 文件 | 說明 |
|------|------|------|
| 06 | [Memory 操作](06_memory_operations.md) | Host-to-NoC 傳輸、交錯傳輸 |
| 07 | [Golden 驗證機制](07_golden_verification.md) | 資料驗證、比對流程 |

### 模擬與分析

| 序號 | 文件 | 說明 |
|------|------|------|
| 08 | [模擬規格](08_simulation.md) | **I/O Pattern 定義**、4 層架構、**NocSystem API (6 組)**、NocConfig、DPI-C Bridge、Replaceable Components、8-Phase Cycle Model |
| 09 | [效能指標](09_metrics.md) | Stats 類別、統計收集 |
| 10 | [QoS 設計](10_qos.md) | QoS 優先級設計 |
| 11 | [驗證策略](11_verification_strategy.md) | 單元測試、整合測試、Golden 驗證、RTL Co-sim、Coverage |

### 附錄

| 序號 | 文件 | 說明 |
|------|------|------|
| A1 | [Physical Channel 架構](A1_physical_channel_modes.md) | 雙通道 Req/Rsp 架構、HoL Blocking 分析 |
| A2 | [Fixed-Width Channel Mode](A2_fixed_width_channel_mode.md) | 統一 408-bit flit 設計分析、效率比較 |
| A3 | [Width Converter](A3_width_converter.md) | AXI 寬度轉換（32b~1024b ↔ 256b native） |
| - | [硬體參數指南](hardware_parameters_guide.md) | 參數調校參考 |

---

## 快速參考

### 依功能分類

**核心元件**
- [Router](02_router.md) - XY Routing、Wormhole Switching、CppRouter Pipeline
- [Network Interface](03_network_interface.md) - AXI <-> Flit 轉換、CppNI Functions
- [Flit 格式](04_flit.md) - 固定 408-bit flit 設計（基準參數文件）
- [內部介面](05_internal_interface.md) - Channel\<T\>、TrafficManager、Allocator

**I/O Pattern & API**
- [模擬規格](08_simulation.md) - Input/Output Pattern、NocSystem Public API (6 組)、DPI-C Bridge
- [範例 patterns](../../examples/patterns/) - config.json、traffic.json、memory .hex files

**操作模式**
- [Host-to-NoC](06_memory_operations.md) - Host 對節點傳輸

**驗證與分析**
- [Golden 驗證](07_golden_verification.md) - 資料正確性驗證
- [效能指標](09_metrics.md) - 統計收集
- [驗證策略](11_verification_strategy.md) - 測試方法論與 coverage

---

## 圖片資源

| 檔案 | 內容 |
|------|------|
| `docs/images/noc_env.txt` | **系統架構圖** |
| `docs/images/NI.jpg` | NI 內部架構 |
| `docs/images/test_bench.jpg` | 測試架構圖 |
| `docs/images/router_arch.jpg` | Router 內部結構 |
| `docs/images/pcie_env.jpg` | 外部參考圖 |
