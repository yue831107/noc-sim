# Flit Format — Fixed-Width Design

NoC 基本傳輸單元。全固定參數設計，消除 configurable width，簡化 RTL 實作與驗證。

---

## 1. Overview

### 1.1 Design Philosophy

全固定參數設計，理由：

1. **多數配置從未使用** — Node ID、Address 寬度等在 tape-out 前即確定
2. **Configurable width 增加 RTL 複雜度** — payload union 需 generate-if，bit-range 計算散佈各模組
3. **驗證矩陣爆炸** — N×P×A 組合使 coverage closure 困難

所有參數鎖定為單一固定值，設計目標支援最大 16×16 mesh（256 nodes）。

### 1.2 Design Parameters

| Parameter | Value | Description |
|-----------|-------|-------------|
| `X_WIDTH` | 4 | X coordinate width (max 16 columns) |
| `Y_WIDTH` | 4 | Y coordinate width (max 16 rows) |
| `NODE_ID_WIDTH` | 8 | src_id / dst_id width (X_WIDTH + Y_WIDTH) |
| `PORT_ID_WIDTH` | 2 | Port ID width (4 ports per router) |
| `AXI_ADDR_WIDTH` | 64 | AXI address width |
| `AXI_ID_WIDTH` | 8 | AXI transaction ID width |
| `AXI_DATA_WIDTH` | 256 | AXI data width (32 bytes) |
| `AXI_USER_WIDTH` | 8 | AXI user signal width |
| `ROB_IDX_WIDTH` | 5 | RoB index width (32 entries) |
| `QOS_WIDTH` | 4 | QoS priority width |
| `ECC_WIDTH` | 32 | Total ECC width (4 × 64-bit granules, 8-bit SECDED each) |
| `HEADER_WIDTH` | 56 | Header width (both versions) |
| `PAYLOAD_WIDTH` | 352 | Maximum payload width (W/R channels) |
| `FLIT_WIDTH` | 408 | Total flit width (Header + Payload) |

### 1.3 Flit Structure

```
  55                                        0   351                              0
  ┌────────────────────────────────────────────┬──────────────────────────────────┐
  │               Header (56 bits)             │        Payload (352 bits)        │
  └────────────────────────────────────────────┴──────────────────────────────────┘
  |<──────────────────────── Flit: 408 bits ────────────────────────────────>|
```

所有 AXI channel 共用 408-bit flit width，較短 payload 以 zero-padding 對齊至 352 bits。Request 與 Response flit 寬度對稱。

Flit 內 wdata/rdata 採 little-endian byte ordering（byte address 0 = data[7:0]），與 AXI bus 一致。Header 與 payload metadata 使用 MSB-first bit numbering（[MSB:LSB]），遵循 SystemVerilog 慣例。

---

## 2. Header Format (56 bits)

Header 固定 56 bits，提供 Valid/Ready mode（No-VC）與 Credit-Based mode（With-VC）兩種配置。兩者僅 bit [55:50] 的 vc_id/rsvd 配置不同，其餘欄位位置與寬度完全相同。

### 2.1 Bit Allocation

| Bit Range | Field | Width | Description |
|-----------|-------|-------|-------------|
| [3:0] | qos | 4 | QoS priority |
| [6:4] | axi_ch | 3 | AXI channel type |
| [14:7] | src_id | 8 | Source node ID |
| [22:15] | dst_id | 8 | Destination node ID (Multicast: don't care, 見 2.2.8) |
| [24:23] | port_id | 2 | Target local port index |
| [25] | last | 1 | Packet end marker |
| [26] | rob_req | 1 | RoB request flag |
| [31:27] | rob_idx | 5 | RoB index (32 entries) |
| [33:32] | commtype | 2 | Communication type |
| [49:34] | multicast_mask | 16 | Multicast bounding box `{x_min, x_max, y_min, y_max}` (見 2.2.8) |
| [52:50] | vc_id ‡ | 3 | Virtual Channel ID (0\~7) |
| [55:53] | rsvd ‡ | 3 | Reserved |
| | **Total** | **56** | |

**‡ Version-dependent fields:**

- **Valid/Ready mode (No-VC):** bit [55:50] 全為 rsvd（6b），無 vc_id。VC 由 signal-level 多組 valid/ready lines 實現。
- **Credit-Based mode (With-VC):** bit [52:50] 為 vc_id（3b），bit [55:53] 為 rsvd（3b）。VC ID 編碼於 header，搭配 credit-based flow control。

### 2.2 Field Definitions

#### 2.2.1 QoS (4 bits)

由 NI 自 AXI `awqos`（Write）或 `arqos`（Read）提取，作為 router arbitration 優先級依據。

| Value | Priority | Description |
|-------|----------|-------------|
| 0 | Lowest | Best effort |
| 15 | Highest | Real-time critical |

AW/AR payload 不重複儲存 qos，header 為唯一來源。Response flit 繼承對應 request 的 `qos`。Router arbiter 以 `qos` 為第一優先級比較依據。詳見 [QoS Design](06_qos.md)。

#### 2.2.2 AXI Channel Type (axi_ch, 3 bits)

| Value | Name | Channel | Description |
|-------|------|---------|-------------|
| 0 | AW | REQ | Write Address |
| 1 | W | REQ | Write Data |
| 2 | AR | REQ | Read Address |
| 3 | B | RSP | Write Response |
| 4 | R | RSP | Read Response |
| 5\~7 | — | — | Reserved |

Router 依據 `axi_ch` 判定 flit 所屬之 Request 或 Response network。

#### 2.2.3 Node ID (src_id / dst_id, 8 bits each)

採用 8-bit 座標編碼（[7:4]=y, [3:0]=x）：

| Bits | Field | Description |
|------|-------|-------------|
| [3:0] | x | X coordinate (0\~15) |
| [7:4] | y | Y coordinate (0\~15) |

XY routing 依據 dst_id 的 x/y 座標進行路由決策。src_id 用於 response 回程路由及 error handling 來源識別。src_id 置於 dst_id 前方的理由見 Section 2.3。

> **Multicast 模式**: 當 `commtype=1` 時，路由改用 `multicast_mask`（16 bits）中的 RCR bounding box 座標，`dst_id` 不參與路由（don't care）。詳見 Section 2.2.8。

#### 2.2.4 Port ID (port_id, 2 bits)

獨立於 node ID，指定目標 Router 的 local port：

| Value | Port | Typical Usage |
|-------|------|---------------|
| 0 | Port 0 | CPU / Processor |
| 1 | Port 1 | DMA Engine |
| 2 | Port 2 | Memory Controller |
| 3 | Port 3 | Accelerator |

Request 時指定目標 local port；response 時 NI 填入原始 requester 的 `port_id`。

#### 2.2.5 Flit Control: last (1 bit)

| Value | Meaning |
|-------|---------|
| 0 | Not last flit (more flits in this packet) |
| 1 | Last flit of packet (tail) |

Wormhole switching 依據 `last` 釋放 path lock。Single-flit packet（AW, AR, B）恆為 `last=1`；multi-flit packet（W, R）僅末尾設 `last=1`。

#### 2.2.6 RoB Fields: rob_req (1 bit), rob_idx (5 bits)

| Field | Width | Description |
|-------|-------|-------------|
| rob_req | 1 | 1 = request RoB reorder; 0 = no reorder |
| rob_idx | 5 | RoB entry index (0\~31) |

Source NI 分配 `rob_idx`，destination NI 依此 index 將 response 寫入對應 RoB slot。`rob_req=0` 時 `rob_idx` 為 don't care。RoB 架構見 Section 5。

#### 2.2.7 Communication Type (commtype, 2 bits)

Communication Type 定義：

| Value | Name | Description |
|-------|------|-------------|
| 0 | Unicast | Single destination (default) |
| 1 | Multicast | Multi-destination broadcast (bounding box) |
| 2 | ParallelReduction | In-network response reduction (B channel) |
| 3 | Reserved | — |

**commtype 轉換規則**：Dest NI 收到 `commtype=1`（Multicast）的 AW flit 後，產生的 B response flit 設為 `commtype=2`（ParallelReduction）。Response Router 據此在 backward path 逐 hop 合併 B response。詳見 [Router Specification](03_router.md) Section 10.3 及 [NI Specification](04_network_interface.md) Section 9。

#### 2.2.8 Multicast Bounding Box (multicast_mask, 16 bits)

當 `commtype=1`（Multicast）或 `commtype=2`（ParallelReduction）時，`multicast_mask`（16 bits）採用 **RCR (Rectangle Coordinate-Range)** 編碼，以 `(min, max)` 座標對表達任意軸對齊矩形，不受 power-of-2 對齊限制。詳見 [RCR-Multicast Spec](08_multicast.md)。

> **ParallelReduction 時的 multicast_mask**：B response flit 複製原始 Multicast AW flit 的 `multicast_mask`，使 Response Router 能用相同的 RCR 公式計算 `in_route_mask`。

**multicast_mask 內部編碼（16 bits, [49:34]）：**

| Mask Bits | Header Bits | Field | Width | Description |
|-----------|-------------|-------|-------|-------------|
| [15:12] | [49:46] | x_min | 4 | 矩形左邊界（含） |
| [11:8] | [45:42] | x_max | 4 | 矩形右邊界（含） |
| [7:4] | [41:38] | y_min | 4 | 矩形下邊界（含） |
| [3:0] | [37:34] | y_max | 4 | 矩形上邊界（含） |

對應 RCR spec 之 `multicast_hdr_t`：

```
multicast_mask = {x_min[3:0], x_max[3:0], y_min[3:0], y_max[3:0]}   // [49:34]

矩形範圍：X ∈ [x_min, x_max], Y ∈ [y_min, y_max]
目標節點數 = (x_max - x_min + 1) × (y_max - y_min + 1)
```

**Multicast 時各欄位語義：**

| Field | Unicast (commtype=0) | Multicast (commtype=1) |
|-------|---------------------|----------------------|
| `dst_id` [22:15] | 目標 node ID（[7:4]=y, [3:0]=x） | Don't care（不參與路由） |
| `multicast_mask` [49:34] | Don't care (=0) | RCR bounding box `{x_min, x_max, y_min, y_max}` |

RCR 路由使用 `src_id`（取得 source 座標）與 `multicast_mask`（取得矩形邊界），`dst_id` 不參與 multicast 路由。

**有效性約束：**
1. `x_min <= x_max <= MAX_X` (MAX_X = mesh_cols - 1)
2. `y_min <= y_max <= MAX_Y` (MAX_Y = mesh_rows - 1)
3. Source 應位於矩形內或邊界上

**退化情形（無需特殊硬體）：**

| 模式 | 編碼 |
|------|------|
| Unicast 至 (dx, dy) | x_min=x_max=dx, y_min=y_max=dy |
| 全網 Broadcast | (0, MAX_X, 0, MAX_Y) |
| 整行/整列 | 固定一軸 min=max，另一軸全展開 |

**Example:** `multicast_mask={x_min=2, x_max=4, y_min=1, y_max=3}` → X ∈ [2,4], Y ∈ [1,3]，共 3×3=9 nodes。

Multicast 僅適用 Request channel（AW/W）。Response 使用 **ParallelReduction**（`commtype=2`）在 Response Router 中逐 hop 合併 B response，Source NMU 僅收到 **1 個**已合併的 B response。詳見 [Router Specification](03_router.md) Section 10.3 及 [NI Specification](04_network_interface.md) Section 9。

#### 2.2.9 VC ID & Reserved

**vc_id (3 bits, Credit-Based mode only):**

支援最多 8 個 Virtual Channels（bit [52:50]）。用途含 Request/Response 分離避免 deadlock、traffic class 隔離。Valid/Ready mode 中此 3 bits 歸入 reserved。

**Reserved bits:**

| Mode | Bit Range | Width |
|------|-----------|-------|
| Valid/Ready (No-VC) | [55:50] | 6 |
| Credit-Based (With-VC) | [55:53] | 3 |

傳輸時設為 0，接收端忽略。

### 2.3 Field Ordering Rationale

| Priority | Category | Fields | Usage |
|----------|----------|--------|-------|
| 1 | Arbitration | qos | Arbiter priority comparison |
| 2 | Opcode | axi_ch | Flit type / network selection |
| 3 | Routing | src_id, dst_id, port_id | XY routing, local port selection |
| 4 | Flit Control | last | Wormhole path lock release |
| 5 | Flow Control | rob_req, rob_idx | Destination NI processing |
| 6 | Multicast | commtype, multicast_mask | Bounding box routing |
| 7 | VC / Reserved | vc_id, rsvd | Version-dependent |

高頻存取欄位置於 LSB 端，第一級 pipeline 僅需 qos + axi_ch 即可完成 arbitration 與 channel 分離。Multicast 與 reserved 置於 MSB 端，未來擴展不影響既有 bit position。

src_id 置於 dst_id 前方：response 路由以 src_id 為回程目的地，error handling 需優先識別來源。

### 2.4 Bit Layout Diagrams

#### Valid/Ready Mode (No-VC)

```
 55    50 49              34 33 32 31    27  26   25  24 23 22    15 14     7  6  4  3  0
┌───────┬─────────────────┬─────┬──────┬────┬────┬─────┬────────┬────────┬─────┬──────┐
│ rsvd  │ multicast_mask   │comm │ rob  │rob │last│port │ dst_id │ src_id │ axi │ qos  │
│  6b   │      16b         │type │idx 5b│req │ 1b │id 2b│   8b   │   8b   │ch 3b│  4b  │
│       │{xmin,xmax,       │ 2b  │      │ 1b │    │     │        │        │     │      │
│       │ ymin,ymax}       │     │      │    │    │     │        │        │     │      │
└───────┴─────────────────┴─────┴──────┴────┴────┴─────┴────────┴────────┴─────┴──────┘
```

#### Credit-Based Mode (With-VC)

```
 55 53 52 50 49              34 33 32 31    27  26   25  24 23 22    15 14     7  6  4  3  0
┌────┬────┬─────────────────┬─────┬──────┬────┬────┬─────┬────────┬────────┬─────┬──────┐
│rsvd│ vc │ multicast_mask   │comm │ rob  │rob │last│port │ dst_id │ src_id │ axi │ qos  │
│ 3b │id 3b│      16b        │type │idx 5b│req │ 1b │id 2b│   8b   │   8b   │ch 3b│  4b  │
│    │    │{xmin,xmax,       │ 2b  │      │ 1b │    │     │        │        │     │      │
│    │    │ ymin,ymax}       │     │      │    │    │     │        │        │     │      │
└────┴────┴─────────────────┴─────┴──────┴────┴────┴─────┴────────┴────────┴─────┴──────┘
```

---

## 3. Payload Format (352 bits max)

所有 payload union-aligned 至 352 bits（由 W/R channel 主導），較短 payload 以 zero-padding 補齊。

### 3.1 AW/AR Channel Payload (108 bits)

AW 與 AR 結構相同，以下以 AW 為例。AR 各欄位將 `aw` 前綴替換為 `ar`。

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

### 3.2 W Channel Payload (352 bits)

| Field | Bit Range | Width | Description |
|-------|-----------|-------|-------------|
| wlast | [0] | 1 | Last beat marker |
| wuser | [8:1] | 8 | AXI user signal |
| wdata | [264:9] | 256 | Write data |
| wstrb | [296:265] | 32 | Write strobe (per-byte enable) |
| wecc | [328:297] | 32 | ECC (SECDED per 64-bit granule) |
| w_rsvd | [351:329] | 23 | Reserved |
| | | **352** | |

`wecc`：每 64-bit wdata granule 產生 8-bit SECDED ECC，4 granules × 8 bits = 32 bits。由 responder 驗證，錯誤透過 `bresp` 回報。

### 3.3 B Channel Payload (64 bits)

| Field | Bit Range | Width | Description |
|-------|-----------|-------|-------------|
| bid | [7:0] | 8 | Write transaction ID |
| bresp | [9:8] | 2 | Write response status |
| buser | [17:10] | 8 | AXI user signal |
| ecc_fail | [18] | 1 | ECC error detected (SECDED uncorrectable) |
| multicast_status | [20:19] | 2 | Multicast completion status |
| b_rsvd | [63:21] | 43 | Reserved |
| | | **64** | |

**multicast_status encoding:**

| Value | Name | Description |
|-------|------|-------------|
| 0 | UNICAST | Non-multicast transaction |
| 1 | MC_ALL_OK | All destinations succeeded |
| 2 | MC_PARTIAL | Partial failure |
| 3 | MC_ALL_FAIL | All destinations failed |

`ecc_fail` 與 `multicast_status` 為 NoC 內部欄位，NI 記錄於 CSR，AXI interface 透過 `bresp` 回報（任一錯誤 → SLVERR）。`buser` 為 AXI user signal pass-through。

Multicast response aggregation：多個 destination 的 B response 經 ParallelReduction 在 source NI 彙整 — 全部 OKAY → MC_ALL_OK，部分非 OKAY → MC_PARTIAL，全部非 OKAY → MC_ALL_FAIL。Multicast 下 `ecc_fail` 表示任一 destination 偵測到 uncorrectable error。Aggregation 機制詳見 [NI Specification](04_network_interface.md)。

### 3.4 R Channel Payload (352 bits)

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

`recc`：每 64-bit rdata granule 產生 8-bit SECDED ECC，共 32 bits。由 requester 驗證，錯誤由軟體處理。

### 3.5 W/R Layout Convention

W 與 R channel 採用相同欄位排列慣例：control fields → data → ECC → reserved。

| Aspect | W Channel | R Channel |
|--------|-----------|-----------|
| Transaction ID | N/A (AXI4 removed WID) | rid (8b) |
| Response status | N/A | rresp (2b) |
| Byte enable | wstrb (32b) | N/A |
| Data | wdata (256b) | rdata (256b) |
| ECC | wecc (32b) | recc (32b) |

### 3.6 ECC Design

採用 SECDED（Single Error Correct, Double Error Detect）取代 DataCheck（odd parity per 8-bit）。

| Parameter | Value |
|-----------|-------|
| Data granule | 64 bits |
| ECC per granule | 8 bits (Hsiao SECDED) |
| Granules per 256-bit data | 4 |
| Total ECC width | 32 bits |
| Error correction | 1-bit per granule |
| Error detection | 2-bit per granule |

**End-to-end scope:**

```
Source NI (generate) → Router (pass-through) → ... → Dest NI (check)
```

- **Generate:** Source NI 於 flit 注入時計算 ECC，填入 `wecc`/`recc`
- **Transparent:** Router 不檢查、不修改 ECC（pass-through）
- **Check:** Destination NI 重算 ECC 比對 — 1-bit error 自動校正；2-bit error 設定 `ecc_fail=1`（B channel）或透過 `rresp` 回報

### 3.7 Union Alignment & Utilization

各 channel payload 以 union 方式共享 352-bit 空間：

| Channel | Network | Actual Payload | Padding | Flit Total |
|---------|---------|---------------|---------|------------|
| AW | REQ | 108 | 244 | 408 |
| W | REQ | 352 | 0 | 408 |
| AR | REQ | 108 | 244 | 408 |
| B | RSP | 64 | 288 | 408 |
| R | RSP | 352 | 0 | 408 |

**Effective utilization**（非 reserved 欄位佔 352-bit payload 比例）：

| Channel | Effective Bits | Utilization |
|---------|---------------|-------------|
| W | 329 (last+user+data+strb+ecc) | 93.5% |
| R | 307 (last+id+resp+user+data+ecc) | 87.2% |
| AW | 108 | 30.7% |
| AR | 108 | 30.7% |
| B | 21 (id+resp+user+ecc_fail+mc_status) | 6.0% |

典型 burst write（awlen=15）組成 1×AW + 16×W + 1×B = 18 flits，W flit 佔 89%，整體 padding waste 約 8%。單次傳輸（awlen=0）padding waste 約 50%，但 burst workload 下整體 link utilization 仍屬高效。

---

## 4. Physical Link Format

Request 與 Response 使用獨立 physical link（雙通道 full-duplex），消除 request-response circular dependency，為 protocol deadlock avoidance 的主要機制。每 cycle 傳輸一個完整 flit。

### 4.1 Link Signals

Request link 與 Response link 結構對稱，差異僅在 flit data 欄位名稱（req / rsp）：

| Field | Bit | Width | Description |
|-------|-----|-------|-------------|
| valid | [0] | 1 | Data valid |
| ready | [1] | 1 | Receiver ready |
| flit | [409:2] | 408 | Flit data (req or rsp) |
| | | **410** | |

兩版 header 的 physical link 寬度相同 — version 差異僅在 header 內部 bit 解釋，不影響 wire width。

### 4.2 Flow Control Modes

| Feature | Handshake (Ver. A) | Credit-Based (Ver. B) |
|---------|--------------------------|--------------------------|
| VC identification | Signal-level (per-VC valid/ready) | Header `vc_id` field |
| Wires per link | NumVC × 410 | 410 (shared) |
| Wire count (4 VC) | 1,640 wires/dir | 410 wires/dir |
| Header overhead | 0 bits | 3 bits (from rsvd) |
| Arbitration | Per-VC at receiver | Muxed at sender |

ASIC 實作推薦 Credit-Based mode — wire count 不隨 VC 數增長，適合高 VC 數與 wire-constrained 場景。Valid/Ready mode 適用 VC ≤ 2 且無需 credit tracking 的簡化設計。

---

## 5. RoB Design Analysis

### 5.1 rob_idx vs axi_id

兩者獨立，用途不同：

| Field | Location | Scope | Purpose |
|-------|----------|-------|---------|
| axi_id | Payload | Per-transaction stream | AXI per-ID ordering |
| rob_idx | Header | Shared across all IDs | RoB entry index |

### 5.2 RoB Architecture

```
                    ┌─────────────────────────────────────┐
                    │         Reorder Buffer              │
                    │  ┌─────┬─────┬─────┬─────┬─────┐   │
                    │  │  0  │  1  │  2  │ ... │ N-1 │   │
                    │  └─────┴─────┴─────┴─────┴─────┘   │
                    │         ↑                           │
                    │     rob_idx                         │
                    └─────────────────────────────────────┘
                              ↑
┌─────────────────────────────┴─────────────────────────────┐
│                    Status Table                            │
│  ┌──────────┬──────────┬──────────┬──────────┐            │
│  │ axi_id=0 │ axi_id=1 │ axi_id=2 │   ...    │            │
│  │ rob_idx  │ rob_idx  │ rob_idx  │          │            │
│  │ rob_req  │ rob_req  │ rob_req  │          │            │
│  └──────────┴──────────┴──────────┴──────────┘            │
└───────────────────────────────────────────────────────────┘
```

每個 NI 擁有獨立 32-entry RoB（per-source-NI），所有 outstanding transactions 共享（跨 axi_id 與 destination）。Source NI 於發送 request 時分配 `rob_idx`，response 攜帶相同 index 返回。

同一 `axi_id` 的多筆 outstanding transactions 各佔一個 rob_idx，per-ID ordering 由 Status Table 保證 — 同 ID responses 依 rob_idx 分配順序依序 release。

---

## 6. Functionality Supported by Packet Design

- **XY Routing** — `src_id` (8b), `dst_id` (8b)
  - 座標編碼（[7:4]=y, [3:0]=x），Router 從 `dst_id` 提取座標進行 X-first then Y routing
  - `src_id` 用於 response 回程路由與 error handling 來源識別
  - 支援最大 16×16 mesh（256 nodes）

- **Wormhole Switching & AXI Response Interleaving** — `last` (1b)
  - Packet-level path lock，`last=1` 釋放路徑
  - Single-flit packet（AW, AR, B）恆為 `last=1`；multi-flit（W, R）僅末尾設 `last=1`
  - 支援 AXI read data interleaving：不同 `rid` 的 R packet 可在 packet boundary 交錯傳輸

- **Multi-port Addressing** — `port_id` (2b)
  - 每 Router 最多 4 local ports（CPU / DMA / MemCtrl / Accelerator）
  - Response 時填入原始 requester 的 `port_id` 確保回程正確

- **QoS Arbitration** — `qos` (4b)
  - 16 級優先級（0=Best Effort, 15=Real-time Critical）
  - 置於 header LSB 端，支援單級 pipeline partial decode

- **Virtual Channel** — `vc_id` (3b, Credit-Based mode), `rsvd` (Valid/Ready mode)
  - Valid/Ready mode：signal-level VC（多組 valid/ready lines）
  - Credit-Based mode：header-level VC（max 8 VCs），搭配 credit-based flow control
  - 用途：Req/Rsp 分離避免 deadlock、traffic class 隔離

- **Multicast & Reduction** — `commtype` (2b), `multicast_mask` (16b)
  - 16-bit RCR bounding box `{x_min, x_max, y_min, y_max}` 編碼任意軸對齊矩形
  - Request 使用 Multicast broadcast，Response 使用 ParallelReduction gather

- **Out-of-Order Completion** — `rob_req` (1b), `rob_idx` (5b)
  - 32-entry RoB per NI，跨 axi_id 與 destination 共享
  - 允許 response 亂序返回，由 destination NI 依 `rob_idx` 重排

- **AXI Protocol Mapping** — `axi_ch` (3b), 5 channel payloads (352b max)
  - 五通道 union-aligned：W/R 佔滿 352b（utilization 87\~93%），AW/AR 108b，B 64b
  - 典型 burst（awlen=15）整體 padding waste 約 8%

- **End-to-End Data Integrity** — `wecc`/`recc` (32b), `ecc_fail` (1b)
  - SECDED ECC per 64-bit granule：1-bit correct, 2-bit detect
  - 保護範圍：Source NI generate → Router pass-through → Dest NI check

- **Dual Physical Network** — Req/Rsp link (410b each)
  - 獨立 full-duplex link，消除 Req-Rsp circular dependency（protocol deadlock avoidance）
  - 每方向每 cycle 32 bytes，Req/Rsp 對稱且 flit 寬度統一（408b）

---

## 7. Limitations & Out-of-Scope

- **AXI4-Stream / Atomic Operations** — 不在本設計範圍。僅支援 AXI4 五通道（AW/W/AR/B/R）
- **Header integrity** — ECC 僅保護 payload data（wdata/rdata），不涵蓋 56-bit header 及 payload metadata（wstrb、axi_id 等）。Header integrity 依賴 physical link layer（wire-level parity 或 CRC）。`dst_id` bit flip 將導致 misrouting
- **Multicast bounding box** — 16-bit RCR 編碼可表達任意軸對齊矩形，不受 power-of-2 對齊限制。但 L 形、對角線、稀疏子集等非矩形區域無法精確表達，需由軟體分解為多個矩形封包或 unicast
- **RoB sizing** — 32 entries 基於典型 embedded SoC workload（CPU + DMA）。高 outstanding 需求（如 GPU）可能需擴展 `ROB_IDX_WIDTH`

---

## Related Documents

- [QoS Design](06_qos.md)
- [Router Specification](03_router.md)
- [Network Interface Specification](04_network_interface.md)
- [Physical Channel Architecture](05_physical_channel.md)
- [Width Converter](A2_width_converter.md)

