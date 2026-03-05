# NoC Behavior Model - 設計文件

本目錄包含 NoC Behavior Model 的設計規格文件。

---

## 架構圖

系統整體架構見 [noc_env.txt](../images/noc_env.txt)（ASCII 架構圖）。

---

## 文件索引

### Group A: 架構設計 (Architecture) — 01~10

| 序號 | 文件 | 說明 |
|------|------|------|
| 01 | [系統概述](01_overview.md) | V1 架構圖、拓撲參數、固定設計參數 |
| 02 | [Flit 格式](02_flit.md) | **基準文件** — 固定 408-bit flit 設計、Header/Payload 格式、Physical Link |
| 03 | [Router 規格](03_router.md) | Ports、XY Routing、Wormhole Arbiter、**CppRouter 內部 Pipeline Stages** |
| 04 | [Network Interface 規格](04_network_interface.md) | NMU、NSU、資料路徑、**CppNI 內部 Functions** |
| 05 | [內部介面架構](05_internal_interface.md) | Credit-Based Flow Control、Port Interface、**Channel\<T\>、TrafficManager、Allocator** |
| 06 | [Physical Channel 架構](06_physical_channel.md) | 雙通道 Req/Rsp 架構、HoL Blocking 分析 |
| 07 | [QoS 設計](07_qos.md) | QoS 優先級設計 |
| 08 | [Memory 操作](08_memory_operations.md) | Host-to-NoC 傳輸、交錯傳輸 |
| 09 | [Width Converter](09_width_converter.md) | AXI 寬度轉換（32b~1024b <-> 256b native） |
| 10 | [Rectangle Multicast](10_multicast.md) | Rectangle Multicast 設計 |

### Group B: 模擬平台 (Simulation) — 11~12

| 序號 | 文件 | 說明 |
|------|------|------|
| 11 | [模擬規格](11_simulation.md) | **I/O Pattern 定義**、4 層架構、**NocSystem API (6 組)**、NocConfig、DPI-C Bridge、Replaceable Components、8-Phase Cycle Model |
| 12 | [硬體參數指南](12_parameters_guide.md) | 參數調校參考 |

### Group C: 驗證與測試 (Verification) — 13~15

| 序號 | 文件 | 說明 |
|------|------|------|
| 13 | [Golden 驗證機制](13_golden_verification.md) | 資料驗證、比對流程 |
| 14 | [效能指標](14_metrics.md) | Stats 類別、統計收集 |
| 15 | [驗證策略](15_verification_strategy.md) | 單元測試、整合測試、Golden 驗證、RTL Co-sim、Coverage |

### 附錄

| 序號 | 文件 | 說明 |
|------|------|------|
| A1 | [Fixed-Width 分析](A1_fixed_width_analysis.md) | 統一 408-bit flit 設計分析、效率比較 |

---

## 快速參考

### 依功能分類

**核心元件**
- [Router](03_router.md) - XY Routing、Wormhole Switching、CppRouter Pipeline
- [Network Interface](04_network_interface.md) - AXI <-> Flit 轉換、CppNI Functions
- [Flit 格式](02_flit.md) - 固定 408-bit flit 設計（基準參數文件）
- [內部介面](05_internal_interface.md) - Channel\<T\>、TrafficManager、Allocator
- [Physical Channel](06_physical_channel.md) - 雙通道 Req/Rsp 架構

**I/O Pattern & API**
- [模擬規格](11_simulation.md) - Input/Output Pattern、NocSystem Public API (6 組)、DPI-C Bridge
- [範例 patterns](../../examples/patterns/) - config.json、traffic.json、memory .hex files

**操作模式**
- [Host-to-NoC](08_memory_operations.md) - Host 對節點傳輸

**驗證與分析**
- [Golden 驗證](13_golden_verification.md) - 資料正確性驗證
- [效能指標](14_metrics.md) - 統計收集
- [驗證策略](15_verification_strategy.md) - 測試方法論與 coverage

---

## 圖片資源

| 檔案 | 內容 |
|------|------|
| `docs/images/noc_env.txt` | **系統架構圖** |
| `docs/images/NI.jpg` | NI 內部架構 |
| `docs/images/test_bench.jpg` | 測試架構圖 |
| `docs/images/router_arch.jpg` | Router 內部結構 |
| `docs/images/pcie_env.jpg` | 外部參考圖 |
