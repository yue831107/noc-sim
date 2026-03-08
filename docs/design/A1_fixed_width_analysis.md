# Fixed-Width Design Rationale

本文件說明 NoC 採用統一 408-bit 固定寬度 flit 的設計動機與效率分析。本文為設計決策背景文件，非規範性規格。所有參數與欄位定義以 [Flit Format](02_flit.md) 為準。

---

## 1. 設計動機

### 1.1 設計方向

本 NoC 採用**統一 408-bit flit width**，所有 AXI channel（AW、W、AR、B、R）共用相同的 flit 格式。較短的 channel payload 以 zero-padding 對齊至 352 bits，與 56-bit header 合計恆為 408 bits。

### 1.2 為何選擇固定寬度

早期設計曾考慮 per-channel variable-width 方案（REQ 144b、W\_DAT 320b、R\_DAT 320b、RSP 42b），但在 RTL 實作與驗證評估後，改採統一固定寬度，理由如下：

| 面向 | Per-Channel Variable-Width | Unified Fixed-Width (408b) |
|------|---------------------------|----------------------------|
| Router crossbar | 每通道各自寬度，需 4 套 crossbar | 單一寬度，一套 crossbar 即可 |
| Buffer 設計 | 不同寬度 FIFO，合成複雜 | 統一寬度 FIFO，面積與 timing 最佳化容易 |
| Arbiter | 各通道獨立仲裁邏輯 | 統一仲裁邏輯，僅依 header `axi_ch` 分辨 |
| Physical link | 多條不同寬度 link，佈線複雜 | 每方向一條 410-bit link（含 valid/ready） |
| 驗證負擔 | 4 種寬度 × N 種 scenario 組合 | 單一寬度，驗證矩陣大幅縮減 |
| Payload 效率 | 各通道 100% 利用率 | W/R 93\~87%，AW/AR 30.7%，B 6.0% |

結論：**Router 簡化帶來的面積與驗證收益，遠超 AW/AR/B channel 的 padding 浪費**。典型 burst workload 下，W/R flit 佔絕大多數，整體 link utilization 仍屬高效。

### 1.3 設計目標

1. **全固定參數** — 消除 configurable width，簡化 RTL 實作與驗證
2. **Union-aligned payload** — 所有 channel 共用 352-bit payload 空間，較短 channel 以 zero-padding 對齊
3. **對稱 Req/Rsp link** — Request 與 Response physical link 寬度相同（408-bit flit + valid + ready = 410-bit），消除 request-response circular dependency
4. **可擴展性** — header 含 Virtual Channel、Multicast、QoS 等欄位，支援進階流控與路由功能

---

## 2. Header Structure

Header 固定 56 bits。完整欄位定義與 bit layout 見 [Flit Format](02_flit.md) §2。

---

## 3. Unified Flit Definitions

### 3.1 Flit Structure

```
  55                                    0   351                              0
  ┌────────────────────────────────────────┬──────────────────────────────────┐
  │             Header (56 bits)           │        Payload (352 bits)        │
  └────────────────────────────────────────┴──────────────────────────────────┘
  |<─────────────────────── Flit: 408 bits ───────────────────────────────>|
```

所有 channel 均使用此結構。Payload 352 bits 為 union-aligned 空間，由 W/R channel 主導寬度，較短 channel 以 zero-padding 補齊。

### 3.2 Network 與 Channel 對應

| Network | AXI Channel | axi\_ch Value | Payload Size | Padding |
|---------|-------------|---------------|-------------|---------|
| **Request** | AW (Write Address) | 0 | 108 bits | 244 bits |
| **Request** | W (Write Data) | 1 | 352 bits | 0 bits |
| **Request** | AR (Read Address) | 2 | 108 bits | 244 bits |
| **Response** | B (Write Response) | 3 | 64 bits | 288 bits |
| **Response** | R (Read Data) | 4 | 352 bits | 0 bits |

### 3.3 AW/AR Channel Payload (108 bits)

AW 與 AR 結構相同（欄位前綴 `aw` / `ar` 互換）：

| Field | Bit Range | Width | Description |
|-------|-----------|-------|-------------|
| awid | [7:0] | 8 | Transaction ID |
| awaddr | [71:8] | 64 | Write address |
| awlen | [79:72] | 8 | Burst length (0\~255) |
| awsize | [82:80] | 3 | Burst size |
| awburst | [84:83] | 2 | Burst type |
| awcache | [88:85] | 4 | Cache attributes |
| awlock | [89] | 1 | Exclusive access |
| awprot | [92:90] | 3 | Protection type |
| awregion | [96:93] | 4 | Memory region identifier |
| awuser | [104:97] | 8 | AXI user signal |
| aw_rsvd | [107:105] | 3 | Reserved (alignment) |
| | | **108** | |
| (padding) | [351:108] | 244 | Zero-padded to 352 bits |

### 3.4 W Channel Payload (352 bits)

佔滿整個 payload 空間，無 padding：

| Field | Bit Range | Width | Description |
|-------|-----------|-------|-------------|
| wlast | [0] | 1 | Last beat marker |
| wuser | [8:1] | 8 | AXI user signal |
| wdata | [264:9] | 256 | Write data |
| wstrb | [296:265] | 32 | Write strobe (per-byte enable) |
| wecc | [328:297] | 32 | ECC (SECDED per 64-bit granule) |
| w_rsvd | [351:329] | 23 | Reserved |
| | | **352** | |

### 3.5 B Channel Payload (64 bits)

| Field | Bit Range | Width | Description |
|-------|-----------|-------|-------------|
| bid | [7:0] | 8 | Write transaction ID |
| bresp | [9:8] | 2 | Write response status |
| buser | [17:10] | 8 | AXI user signal |
| ecc_fail | [18] | 1 | ECC error detected |
| multicast_status | [20:19] | 2 | Multicast completion status |
| b_rsvd | [63:21] | 43 | Reserved |
| | | **64** | |
| (padding) | [351:64] | 288 | Zero-padded to 352 bits |

### 3.6 R Channel Payload (352 bits)

佔滿整個 payload 空間，無 padding：

| Field | Bit Range | Width | Description |
|-------|-----------|-------|-------------|
| rlast | [0] | 1 | Last beat marker |
| rid | [8:1] | 8 | Read transaction ID |
| rresp | [10:9] | 2 | Read response status |
| ruser | [18:11] | 8 | AXI user signal |
| rdata | [274:19] | 256 | Read data |
| recc | [306:275] | 32 | ECC (SECDED per 64-bit granule) |
| r_rsvd | [351:307] | 45 | Reserved |
| | | **352** | |

---

## 4. Efficiency Analysis

### 4.1 Per-Channel Utilization

以非 reserved 欄位佔 352-bit payload 的比例計算：

| Channel | Effective Bits | Utilization | 說明 |
|---------|---------------|-------------|------|
| W | 329 | **93.5%** | wlast + wuser + wdata + wstrb + wecc |
| R | 307 | **87.2%** | rlast + rid + rresp + ruser + rdata + recc |
| AW | 108 | **30.7%** | 完整 AXI address channel 資訊 |
| AR | 108 | **30.7%** | 同 AW |
| B | 21 | **6.0%** | bid + bresp + buser + ecc\_fail + mc\_status |

W/R channel 為頻寬主體，utilization 在 87\~93% 之間，屬高效率。AW/AR/B 因 payload 較短而 utilization 較低，但這些 channel 在 burst 傳輸中佔比極小。

### 4.2 Burst Workload 效率分析

典型 burst write（`awlen=15`，16-beat burst）：

| Flit Type | Count | Payload Used | Padding Waste |
|-----------|-------|-------------|---------------|
| AW | 1 | 108 bits | 244 bits |
| W | 16 | 352 bits × 16 = 5,632 bits | 0 bits |
| B | 1 | 64 bits | 288 bits |
| **Total** | **18** | **5,804 bits** | **532 bits** |

- W flit 佔 packet 的 88.9%（16/18 flits）
- 整體 padding waste：532 / (18 × 352) ≈ **8.4%**

典型 burst read（`arlen=15`，16-beat burst）：

| Flit Type | Count | Payload Used | Padding Waste |
|-----------|-------|-------------|---------------|
| AR | 1 | 108 bits | 244 bits |
| R | 16 | 352 bits × 16 = 5,632 bits | 0 bits |
| **Total** | **17** | **5,740 bits** | **244 bits** |

- R flit 佔 packet 的 94.1%（16/17 flits）
- 整體 padding waste：244 / (17 × 352) ≈ **4.1%**

單次傳輸（`awlen=0`）padding waste 約 50%，但 burst workload 為主要使用場景，整體 link utilization 仍屬高效。

### 4.3 頻寬利用率

每方向每 cycle 傳輸一個 408-bit flit，其中 352 bits 為 payload 空間。以 W/R data channel 為主體時：

- W channel 每 flit 傳輸 256 bits data + 32 bits wstrb = 288 bits 有效負載
- R channel 每 flit 傳輸 256 bits data = 256 bits 有效負載
- 每方向每 cycle 有效 data throughput：**256 bits = 32 bytes**

---

## 5. Per-Channel Variable-Width 比較

### 5.1 Variable-Width 方案概述

早期設計曾考慮每 channel 使用獨立且最適寬度的 link：

| Channel | Variable Width | Content |
|---------|---------------|---------|
| REQ (AW/AR) | \~144b | Header + address payload |
| W\_DAT | \~320b | Header + wstrb + wdata |
| R\_DAT | \~320b | Header + rdata + metadata |
| RSP (B) | \~42b | Header + bid + bresp |

### 5.2 比較分析

| 指標 | Variable-Width | Fixed-Width (408b) |
|------|---------------|-------------------|
| **Payload 效率** | 各通道 93\~100% | W 93.5%, R 87.2%, AW/AR 30.7%, B 6.0% |
| **Router 複雜度** | 4 套不同寬度 crossbar、buffer、arbiter | 單一寬度，共用 crossbar/buffer/arbiter |
| **Physical link 數** | 每方向 2\~3 條不同寬度 link | 每方向 1 條 410-bit link |
| **總佈線寬度** | 144 + 320 + 320 + 42 = 826b（單向） | 410b（單向），雙方向 × 2 network = 1,640b |
| **合成友善度** | 不同寬度 FIFO 與 MUX 合成困難 | 統一寬度，timing closure 較容易 |
| **驗證成本** | 4 種寬度 × scenario 組合 | 單一寬度，驗證矩陣大幅縮減 |
| **Burst 整體效率** | 略高（B/AW/AR 無 padding） | 極接近（burst 中 W/R 佔主體，padding 影響 <10%） |

### 5.3 結論

Variable-width 方案的 per-channel 效率優勢僅體現在 AW/AR/B 三個低頻 channel。在實際 burst workload 中，W/R flit 佔 packet 的 89\~94%，整體效率差距極小。而 fixed-width 方案在 Router 設計、物理佈線、驗證複雜度等方面均有顯著優勢，為本設計最終選擇。

---

## 6. Router Architecture Implications

### 6.1 Crossbar 簡化

統一 408-bit flit width 使 Router 的 5×5 crossbar（N/S/E/W/Local）僅需一套 408-bit MUX 陣列。Variable-width 方案則需為每個 channel 寬度各設一套 crossbar，面積開銷顯著增加。

### 6.2 Buffer 統一

Input buffer（FIFO）均為 408-bit wide，深度可依需求設定。合成工具對統一寬度 SRAM/Register File 的面積與 timing 最佳化效果最好。不同寬度的 buffer 會造成合成工具難以進行 memory mapping。

### 6.3 Arbiter 統一

Arbiter 僅需讀取 header 的 `qos`（bit [3:0]）與 `axi_ch`（bit [6:4]）進行仲裁決策。由於所有 flit header 位置與寬度一致，arbiter 邏輯無需因 channel 寬度不同而區分處理。

### 6.4 雙子 Router 架構

邏輯 Router 由兩個結構對稱的子 Router 組成：

```
  ┌─────────────────────────────────────────────────────┐
  │                   Logical Router                     │
  │                                                      │
  │  ┌──────────────────┐    ┌──────────────────┐       │
  │  │    ReqRouter      │    │    RspRouter      │       │
  │  │  (AW, W, AR)      │    │    (B, R)         │       │
  │  │                    │    │                    │       │
  │  │  5×5 Crossbar     │    │  5×5 Crossbar     │       │
  │  │  408-bit wide     │    │  408-bit wide     │       │
  │  │  Unified buffers  │    │  Unified buffers  │       │
  │  │  QoS-aware arb    │    │  QoS-aware arb    │       │
  │  └──────────────────┘    └──────────────────┘       │
  │                                                      │
  └─────────────────────────────────────────────────────┘
```

兩個子 Router 結構完全相同，僅處理的 flit 類型不同。此對稱性得益於統一 flit width — ReqRouter 與 RspRouter 可共用相同的 RTL module（參數化 instance），減少設計與驗證工作量。

### 6.5 Physical Link

每條 physical link 結構如下：

| Field | Width | Description |
|-------|-------|-------------|
| valid | 1 | Data valid |
| ready | 1 | Receiver ready |
| flit | 408 | Flit data |
| **Total** | **410** | |

每方向一條 410-bit link，Req 與 Rsp 各一條，雙方向共 4 條。兩種 flow control mode（Valid/Ready / Credit-Based）的 physical link 寬度相同，差異僅在 header 內部 bit 解釋。

---

## Related Documents

- [Flit Format](02_flit.md) — 完整 flit 參數與欄位定義
- [Physical Channel Architecture](05_physical_channel.md) — 雙通道架構詳述
- [Width Converter](A2_width_converter.md) — 不同 AXI bus 寬度轉換
- [Router Specification](03_router.md) — Router 內部架構
- [Network Interface Specification](04_network_interface.md) — NI 設計
