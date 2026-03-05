# Physical Channel Architecture

本文件描述 NoC 的 Physical Channel 架構設計。採用 2-channel（Request + Response）dual-rail full-duplex 架構，所有 AXI channel 共用統一的 408-bit flit format。

---

## 1. Architecture Overview

### 1.1 雙通道設計

NoC 採用 Request / Response 雙通道分離架構。每個邏輯 Router 內部由兩個結構對稱的獨立子 Router 組成：

| Sub-Router | Network | AXI Channels | Direction (typical) |
|------------|---------|--------------|---------------------|
| **ReqRouter** | Request | AW, W, AR | Master → Slave |
| **RspRouter** | Response | B, R | Slave → Master |

兩個子 Router 共用相同的 routing algorithm（XY routing）、arbitration policy（QoS-Aware Round-Robin）、crossbar structure，僅處理的 flit 類型不同（依 header `axi_ch` 欄位區分）。

### 1.2 架構圖

```
                    Request Network                    Response Network
                   (AW, W, AR flits)                  (B, R flits)

  NSU                              NMU
  ┌────────┐                             ┌────────┐
  │        │── AW flit ──┐   ┌────────── │        │── B flit ──┐   ┌──────────
  │  AXI   │── W  flit ──┼──►│ReqRouter│ │  AXI   │             ├──►│RspRouter│
  │ Master │── AR flit ──┘   │(5x5 XB)│ │ Slave  │── R flit ──┘   │(5x5 XB)│
  │        │                 └────┬────┘ │        │                 └────┬────┘
  │        │◄── B flit ──────────-│------│        │◄── AW flit ─────────│------
  │        │◄── R flit ──────────-│------│        │◄── W  flit ─────────│------
  └────────┘                      │      └────────┘◄── AR flit ─────────│
                                  │                                     │
              ┌───────────────────┘                 ┌───────────────────┘
              ▼                                     ▼
      Req Physical Link                     Rsp Physical Link
       (410 bits/dir)                        (410 bits/dir)
```

完整的 Router-to-Router 連接：

```
  Router A                                                Router B
  ┌──────────────────┐                                   ┌──────────────────┐
  │  ┌─────────────┐ │   Req Link (410b) ──────────►    │ ┌─────────────┐  │
  │  │  ReqRouter  │ │   ◄────────── Req Link (410b)    │ │  ReqRouter  │  │
  │  └─────────────┘ │                                   │ └─────────────┘  │
  │                   │                                   │                  │
  │  ┌─────────────┐ │   Rsp Link (410b) ──────────►    │ ┌─────────────┐  │
  │  │  RspRouter  │ │   ◄────────── Rsp Link (410b)    │ │  RspRouter  │  │
  │  └─────────────┘ │                                   │ └─────────────┘  │
  └──────────────────┘                                   └──────────────────┘
        4 × 410 = 1,640 bits total (bidirectional, both networks)
```

### 1.3 設計參數

所有參數與 [Flit Format](02_flit.md) 一致：

| Parameter | Value | Description |
|-----------|-------|-------------|
| `FLIT_WIDTH` | 408 bits | Flit 總寬度 (Header + Payload) |
| `HEADER_WIDTH` | 56 bits | Header 寬度 |
| `PAYLOAD_WIDTH` | 352 bits | 最大 Payload 寬度 (union-aligned) |
| `AXI_DATA_WIDTH` | 256 bits | AXI 資料寬度 (32 bytes) |
| `NODE_ID_WIDTH` | 8 bits | Node ID 寬度 ({y[3:0], x[3:0]}) |
| `LINK_WIDTH` | 410 bits | Physical link 寬度 (valid + ready + flit) |

---

## 2. Physical Link Structure

### 2.1 Link Signal Definition

Request link 與 Response link 結構完全對稱。每條 physical link 固定 410 bits：

```
  409                                                              2  1  0
  ┌──────────────────────────────────────────────────────────────┬────┬────┐
  │                     flit (408 bits)                          │ready│valid│
  │           header (56b)  +  payload (352b)                    │ 1b  │ 1b │
  └──────────────────────────────────────────────────────────────┴────┴────┘
  |<────────────────────── Physical Link: 410 bits ──────────────────────>|
```

| Field | Bit | Width | Description |
|-------|-----|-------|-------------|
| valid | [0] | 1 | Data valid — sender 有 flit 可傳送 |
| ready | [1] | 1 | Receiver ready — downstream buffer 有空間 |
| flit | [409:2] | 408 | Flit data (header + payload) |
| | | **410** | |

### 2.2 Handshake Protocol

採用 valid/ready handshake（AXI-style）：flit 於 `valid && ready` 時完成一次傳輸，每 cycle 最多傳輸一個完整 flit。

```cpp
// Physical link transfer condition
bool transfer_occurred = port.out_valid && port.in_ready;
```

兩版 header（Version A: No-VC / Version B: With-VC）使用相同的 410-bit physical link — version 差異僅在 header 內部 bit 解釋，不影響 wire width。

### 2.3 Flow Control Variants

| Feature | Handshake (Ver. A) | Credit-Based (Ver. B) |
|---------|--------------------------|--------------------------|
| VC identification | Signal-level (per-VC valid/ready) | Header `vc_id` field |
| Wires per link (single VC) | 410 | 410 |
| Wires per link (4 VCs) | 4 × 410 = 1,640 | 410 (shared) |
| Header overhead | 0 bits | 3 bits (from rsvd) |

Version B（credit-based）wire count 不隨 VC 數增長，為 ASIC 實作推薦方案。

---

## 3. AXI Channel Mapping

### 3.1 axi_ch 欄位定義

Header 的 `axi_ch`（3 bits, bit [6:4]）決定 flit 所屬之 AXI channel 與 network：

| axi_ch | Name | Network | Flit Type | Description |
|--------|------|---------|-----------|-------------|
| 0 | AW | Request | Single-flit | Write Address |
| 1 | W | Request | Multi-flit | Write Data |
| 2 | AR | Request | Single-flit | Read Address |
| 3 | B | Response | Single-flit | Write Response |
| 4 | R | Response | Multi-flit | Read Data |
| 5~7 | — | — | — | Reserved |

### 3.2 Network 分流規則

NI 根據 `axi_ch` 將 flit 注入對應 network：

```cpp
// NI: determine target network based on axi_ch
Network select_network(uint8_t axi_ch) {
    switch (axi_ch) {
        case 0: // AW - Write Address
        case 1: // W  - Write Data
        case 2: // AR - Read Address
            return Network::REQUEST;
        case 3: // B  - Write Response
        case 4: // R  - Read Data
            return Network::RESPONSE;
        default:
            assert(false && "Invalid axi_ch");
    }
}
```

Router 內部不檢查 `axi_ch` — ReqRouter 僅接收 Request flit，RspRouter 僅接收 Response flit，network 分離在 NI 端完成。

### 3.3 Packet 結構

每個 AXI transaction 在 NoC 中映射為一或多個 flit 組成的 packet：

| AXI Channel | Flit 數量 | Packet 特性 |
|-------------|-----------|-------------|
| AW | 1 | Single-flit, `last=1` |
| W | awlen+1 | Multi-flit, 僅末尾 `last=1` |
| AR | 1 | Single-flit, `last=1` |
| B | 1 | Single-flit, `last=1` |
| R | arlen+1 | Multi-flit, 僅末尾 `last=1` |

**Write transaction packet sequence:**

```
Request Network:  [AW flit] → [W flit 0] → [W flit 1] → ... → [W flit N, last=1]
Response Network: [B flit, last=1]
```

**Read transaction packet sequence:**

```
Request Network:  [AR flit, last=1]
Response Network: [R flit 0] → [R flit 1] → ... → [R flit N, last=1]
```

---

## 4. Per-Direction Wire Count Analysis

### 4.1 單方向 Link

每個方向的 physical link 為 410 bits（Version A, single VC）：

| Component | Bits |
|-----------|------|
| valid | 1 |
| ready | 1 |
| flit (header + payload) | 408 |
| **Total per link** | **410** |

### 4.2 兩相鄰 Router 間總線數

每對相鄰 Router 之間有 4 條 physical link（Req 雙向 + Rsp 雙向）：

| Link | Direction | Width | Purpose |
|------|-----------|-------|---------|
| Req forward | A → B | 410 | Request flit (A to B) |
| Req reverse | B → A | 410 | Request flit (B to A) |
| Rsp forward | A → B | 410 | Response flit (A to B) |
| Rsp reverse | B → A | 410 | Response flit (B to A) |
| **Total** | | **1,640** | |

```
  Router A                          Router B
     │                                 │
     ├──── Req (410b) ────────────────►│
     │◄──── Req (410b) ────────────────┤
     │                                 │
     ├──── Rsp (410b) ────────────────►│
     │◄──── Rsp (410b) ────────────────┤
     │                                 │
         Total: 4 × 410 = 1,640 bits
```

### 4.3 Version B (Credit-Based) Wire Count 優勢

使用 Version B header 時，多 VC 場景下 wire count 維持不變：

| VCs | Version A (per-VC lines) | Version B (shared) |
|-----|--------------------------|---------------------|
| 1 | 4 × 410 = 1,640 | 4 × 410 = 1,640 |
| 2 | 4 × 820 = 3,280 | 4 × 410 = 1,640 |
| 4 | 4 × 1,640 = 6,560 | 4 × 410 = 1,640 |
| 8 | 4 × 3,280 = 13,120 | 4 × 410 = 1,640 |

Version A 的 wire count 隨 VC 數線性增長，Version B 透過 header 內 `vc_id` 欄位多工，wire count 恆定。

### 4.4 Mesh 整體 Wire Budget

以 4×4 mesh（16 nodes）為例，水平方向 link 數 = 3×4 = 12，垂直方向 link 數 = 4×3 = 12，總計 24 對 Router 連接：

| Topology | Router Pairs | Wires per Pair | Total Wires |
|----------|-------------|----------------|-------------|
| 4×4 mesh | 24 | 1,640 | 39,360 |
| 8×8 mesh | 112 | 1,640 | 183,680 |
| 16×16 mesh | 480 | 1,640 | 787,200 |

---

## 5. Payload Utilization Analysis

### 5.1 統一 Flit Width

所有 5 個 AXI channel 共用相同的 408-bit flit width（56-bit header + 352-bit payload）。較短 payload 以 zero-padding 對齊至 352 bits：

| Channel | Network | Actual Payload | Padding | Flit Total |
|---------|---------|---------------|---------|------------|
| AW | REQ | 108 | 244 | 408 |
| W | REQ | 352 | 0 | 408 |
| AR | REQ | 108 | 244 | 408 |
| B | RSP | 64 | 288 | 408 |
| R | RSP | 352 | 0 | 408 |

### 5.2 Effective Utilization

非 reserved 欄位佔 352-bit payload 的比例（資料來自 [Flit Format](02_flit.md) Section 3.7）：

| Channel | Effective Bits | Utilization | 主要內容 |
|---------|---------------|-------------|----------|
| W | 329 | 93.5% | wlast + wuser + wdata(256b) + wstrb(32b) + wecc(32b) |
| R | 307 | 87.2% | rlast + rid + rresp + ruser + rdata(256b) + recc(32b) |
| AW | 108 | 30.7% | awid + awaddr(64b) + burst params + awuser |
| AR | 108 | 30.7% | arid + araddr(64b) + burst params + aruser |
| B | 21 | 6.0% | bid + bresp + buser + ecc_fail + multicast_status |

### 5.3 Burst Workload 效率分析

典型 burst transaction 的整體 link utilization 取決於 data flit 佔比：

**Write burst (awlen=15, 16 beats):**
```
Packet composition: 1×AW + 16×W + 1×B = 18 flits
W flit ratio:       16/18 = 88.9%
Overall padding:    (1×244 + 16×0 + 1×288) / (18×352) = 8.4%
```

**Read burst (arlen=15, 16 beats):**
```
Packet composition: 1×AR + 16×R = 17 flits
R flit ratio:       16/17 = 94.1%
Overall padding:    (1×244 + 16×0) / (17×352) = 4.1%
```

**Single-beat transaction (awlen=0):**
```
Write: 1×AW + 1×W + 1×B = 3 flits, padding waste ≈ 50%
Read:  1×AR + 1×R = 2 flits, padding waste ≈ 35%
```

結論：burst workload 下 W/R flit 佔絕對多數，整體 link utilization 高效。單次傳輸的 padding waste 較高，但屬於非典型 workload。

---

## 6. Head-of-Line Blocking 特性

### 6.1 Request Channel 共用問題

Request network 承載三種 AXI channel（AW, W, AR），它們共用同一 physical link 與 Router input buffer。這產生以下 blocking 行為：

#### 6.1.1 W Burst 阻塞 AW/AR

W channel 的 multi-flit packet 在 wormhole switching 下鎖定 input→output 路徑。鎖定期間，同一 input buffer 中排在後方的 AW 或 AR flit 無法被仲裁：

```
Request Router input buffer (East port):
  [Position 0] W flit (BODY) → locked to output NORTH
  [Position 1] W flit (BODY) → follows locked path
  [Position 2] W flit (TAIL, last=1) → will release lock
  [Position 3] AR flit (HEAD) → wants output WEST, must wait

AR flit 被 W burst 阻塞，直到 TAIL flit 傳送完畢釋放 path lock。
```

#### 6.1.2 AW/AR 互相阻塞

AW 與 AR 均為 single-flit packet（`last=1`），不會長時間鎖定路徑。但當它們競爭同一 output port 時，仍需經 QoS-Aware Round-Robin 仲裁，落選者延遲一個 cycle。

#### 6.1.3 Blocking Duration 分析

W burst 造成的 blocking duration 與 `awlen` 成正比：

| awlen | W Flits | Blocking Cycles (worst case) |
|-------|---------|------------------------------|
| 0 | 1 | 1 (single-flit, immediate release) |
| 1 | 2 | 2 |
| 7 | 8 | 8 |
| 15 | 16 | 16 |
| 255 | 256 | 256 |

高 `awlen` 值的 W burst 可能導致 AR flit 長時間等待，影響 read latency。

### 6.2 Response Channel 特性

Response network 承載 B（single-flit）與 R（multi-flit）。R burst 同樣會鎖定路徑阻塞同 input buffer 中的 B flit。但 B flit 通常不具時間敏感性（write completion 已在 data 傳輸完成後），影響較小。

### 6.3 HoL Blocking 緩解機制

| 機制 | 說明 | 適用場景 |
|------|------|----------|
| **AXI burst length 限制** | 限制 `awlen` / `arlen` 上限，降低單一 packet 佔用時間 | 系統層級配置 |
| **Virtual Channels (VC)** | 不同 VC 擁有獨立 input buffer，W/R 佔用不阻塞其他 VC | Version B header |
| **QoS priority** | 高 `qos` 的 flit 在 HEAD 仲裁時優先獲得路徑 | 時間敏感流量 |
| **多 path 路由 (future)** | Adaptive routing 提供替代路徑 | 進階設計 |

> **Note:** VC 是最有效的 HoL blocking 解法。透過 Version B header 的 `vc_id` 將 AW/AR 與 W 分配至不同 VC，可完全消除 W burst 對 AW/AR 的阻塞。

---

## 7. Symmetric Req/Rsp Design 優勢

### 7.1 消除 Request-Response Circular Dependency

Protocol deadlock 的典型成因：Request 與 Response 共用同一 network 時，Request 佔滿 buffer 導致 Response 無法通過，而 Request 又在等待 Response 完成，形成 circular dependency。

```
Deadlock scenario (shared network):

  Node A ──[Req]──► buffer full ──[blocked]──► Node B
  Node A ◄──[Rsp]── buffer full ──[blocked]── Node B
                        ↑                        │
                        └───── circular wait ─────┘
```

雙通道分離設計從物理層面消除此 dependency：

```
Deadlock-free (separate networks):

  Request Network:   A ══[Req]══► buffer ══► B    (independent)
  Response Network:  A ◄══[Rsp]══ buffer ◄══ B    (independent)

  Request buffer full 不影響 Response 傳輸
  Response buffer full 不影響 Request 傳輸
```

這是 protocol-level deadlock avoidance 的主要機制，無需額外的 software intervention 或 timeout recovery。

### 7.2 簡化 Router 設計

Req/Rsp link 寬度對稱（均為 410 bits），兩個子 Router 可完全複用相同的硬體模組：

```cpp
// ReqRouter 與 RspRouter 繼承自相同的 XYRouter 基類
struct ReqRouter : public XYRouter {
    // 處理 axi_ch = {0 (AW), 1 (W), 2 (AR)}
    // 使用相同的 routing, arbitration, crossbar 邏輯
};

struct RspRouter : public XYRouter {
    // 處理 axi_ch = {3 (B), 4 (R)}
    // 使用相同的 routing, arbitration, crossbar 邏輯
};

struct Router {
    Coord     coord;
    ReqRouter req_router;  // 408-bit flit, 410-bit link
    RspRouter rsp_router;  // 408-bit flit, 410-bit link（完全對稱）
};
```

對稱設計的具體優勢：

| Aspect | Symmetric Design | Asymmetric Design |
|--------|-----------------|-------------------|
| RTL 模組 | 一套 XYRouter，實例化 2 次 | 兩套不同寬度的 Router |
| Verification | 一套 testbench 覆蓋兩個 network | 需分別驗證 |
| Timing closure | 相同 critical path | 不同 path 需分別優化 |
| Physical design | 統一 wire pitch | 不同 width 的 wire bundle |
| Buffer sizing | 相同 SRAM macro | 不同大小的 SRAM |

### 7.3 Full-Duplex 頻寬

Request 與 Response 使用獨立 physical link，可同時傳輸，提供 full-duplex 頻寬：

| Metric | Value |
|--------|-------|
| Per-link bandwidth | 408 bits/cycle（payload 有效吞吐量 = 256 bits = 32 bytes/cycle） |
| Per-direction (Req + Rsp) | 2 × 408 = 816 bits/cycle |
| Bidirectional total | 4 × 408 = 1,632 bits/cycle |

在典型 read-heavy workload 中，Request network 傳輸 AR flit（低 utilization），Response network 傳輸 R flit（高 utilization）；write-heavy workload 則相反。雙通道 full-duplex 確保兩個方向的頻寬不互相干擾。

### 7.4 設計 Trade-off

| 優點 | 代價 |
|------|------|
| 消除 Req-Rsp deadlock | Wire count 加倍（2 networks） |
| Router 模組複用 | 每個 Router 面積 ×2 |
| Full-duplex 頻寬 | 低負載時一半 link 閒置 |
| 簡化 flow control | — |

Wire count 加倍是主要代價，但在 NoC 設計中，deadlock-free 的保證遠比 wire 節省重要。此為主流 NoC 的共同設計選擇。

---

## 8. 3-Channel Architecture Overview

### 8.1 設計動機

2-channel（Req/Rsp）架構使用統一 408-bit flit，所有 AXI channel 共用相同寬度。這在承載 256-bit wide data（W/R）時效率高，但對 address-only channel（AW/AR/B）浪費大量 padding（見 Section 5）。

Narrow-Wide 3-channel 設計引入第三條 physical channel，將 **narrow 控制封包** 與 **wide 資料封包** 分離，避免 wide data burst 阻塞 narrow 控制流量。

### 8.2 三通道定義

| Channel | Flit Width | 用途 | 承載的 AXI Channels |
|---------|-----------|------|---------------------|
| **req** (Narrow) | 164 bits | Narrow request | NarrowAW, NarrowW, NarrowAR, WideAR |
| **rsp** (Narrow) | 164 bits | Narrow response | NarrowR, NarrowB, WideB |
| **wide** | 408 bits | Wide data | WideAW, WideW, WideR |

### 8.3 Narrow + Wide 雙 AXI 介面

3-channel 模式支援 **兩種 AXI 寬度介面** 同時存在：

| Interface | Data Width | 典型用途 |
|-----------|-----------|---------|
| Narrow AXI | 64 bits | CPU cache line fetch、CSR access、control plane |
| Wide AXI | 256 bits | DMA bulk transfer、GPU texture、video frame |

### 8.4 設計參數

| Parameter | Value | Description |
|-----------|-------|-------------|
| `NARROW_DATA_WIDTH` | 64 | Narrow AXI data width |
| `WIDE_DATA_WIDTH` | 256 | Wide AXI data width（= 現有 `AXI_DATA_WIDTH`） |
| `NARROW_ECC_WIDTH` | 8 | 1 × 64-bit granule, 8-bit SECDED |
| `WIDE_ECC_WIDTH` | 32 | 4 × 64-bit granule, 8-bit SECDED each |
| `NARROW_FLIT_WIDTH` | 164 | Narrow flit 總寬度 (Header 56 + Payload 108) |
| `WIDE_FLIT_WIDTH` | 408 | Wide flit 總寬度（與 2-Ch 相同） |
| `NARROW_LINK_WIDTH` | 166 | valid(1) + ready(1) + flit(164) |
| `WIDE_LINK_WIDTH` | 410 | valid(1) + ready(1) + flit(408) |

---

## 9. Narrow Flit Format（164 bits）

### 9.1 結構總覽

Narrow flit 使用與 [Flit Format](02_flit.md) **相同的 56-bit header**，payload 縮小為 108 bits：

```
  163                                              108 107                      0
  ┌──────────────────────────────────────────────────┬──────────────────────────┐
  │              Header (56 bits)                     │    Payload (108 bits)    │
  │  與 02_flit.md Section 2 完全一致                  │    (channel-dependent)   │
  └──────────────────────────────────────────────────┴──────────────────────────┘
```

Header 欄位（qos, axi_ch, src_id, dst_id, rob_idx, last, vc_id/rsvd）定義不變，確保 routing、arbitration、flow control 邏輯完全複用。

### 9.2 Narrow Req Payload（108 bits）

Union of NarrowAW / NarrowAR / WideAR（address channels）：

| Field | Bit Range | Width | Channel |
|-------|-----------|-------|---------|
| awid/arid | [7:0] | 8 | AW/AR |
| awaddr/araddr | [71:8] | 64 | AW/AR |
| awlen/arlen | [79:72] | 8 | AW/AR |
| awsize/arsize | [82:80] | 3 | AW/AR |
| awburst/arburst | [84:83] | 2 | AW/AR |
| awcache/arcache | [88:85] | 4 | AW/AR |
| awlock/arlock | [89] | 1 | AW/AR |
| awprot/arprot | [92:90] | 3 | AW/AR |
| awregion/arregion | [96:93] | 4 | AW/AR |
| awuser/aruser | [104:97] | 8 | AW/AR |
| rsvd | [107:105] | 3 | — |
| **Total** | | **108** | |

NarrowW union（同 108b 空間）：

| Field | Width | Description |
|-------|-------|-------------|
| wlast | 1 | Last beat |
| wuser | 8 | User signal |
| wdata | 64 | Narrow write data |
| wstrb | 8 | Write strobe (64b / 8) |
| wecc | 8 | 1 × SECDED |
| rsvd | 19 | Reserved |
| **Total** | **108** | |

### 9.3 Narrow Rsp Payload（108 bits）

NarrowR union：

| Field | Width | Description |
|-------|-------|-------------|
| rlast | 1 | Last beat |
| rid | 8 | Read ID |
| rresp | 2 | Read response |
| ruser | 8 | User signal |
| rdata | 64 | Narrow read data |
| recc | 8 | 1 × SECDED |
| rsvd | 17 | Reserved |
| **Total** | **108** | |

NarrowB / WideB union：

| Field | Width | Description |
|-------|-------|-------------|
| bid | 8 | Write response ID |
| bresp | 2 | Write response |
| buser | 8 | User signal |
| ecc_fail | 1 | ECC failure flag |
| multicast_status | 2 | Multicast aggregation status |
| rsvd | 87 | Reserved |
| **Total** | **108** | |

### 9.4 Wide Flit Format（408 bits）

與現有 [Flit Format](02_flit.md) **完全相同**（56-bit header + 352-bit payload）。Wide channel 承載 WideAW、WideW、WideR，payload 格式不變。

---

## 10. Channel Mapping

### 10.1 AXI Channel → Physical Channel 映射表

| AXI Channel | → req | → rsp | → wide | 理由 |
|-------------|-------|-------|--------|------|
| NarrowAW | ✓ | | | 窄地址請求 |
| NarrowAR | ✓ | | | 窄地址請求 |
| NarrowW | ✓ | | | 窄寫資料（64b 小封包） |
| NarrowR | | ✓ | | 窄讀回應 |
| NarrowB | | ✓ | | 窄寫回應 |
| WideAW | | | ✓ | 與 WideW 同 channel 保序 |
| WideAR | ✓ | | | 僅地址，小封包走 req |
| WideW | | | ✓ | 256b 大資料 |
| WideR | | | ✓ | 256b 大資料 |
| WideB | | ✓ | | 僅回應，小封包走 rsp |

### 10.2 關鍵設計原則

1. **WideAW + WideW 同 channel（wide）**：保證 write address/data ordering，避免 cross-channel reordering
2. **WideAR 走 req（非 wide）**：AR 為 address-only 小封包，不佔 wide 頻寬
3. **WideB 走 rsp（非 wide）**：B 為 response-only 小封包，不佔 wide 頻寬
4. **Narrow 全走 req/rsp**：64b 資料可直接塞入 narrow flit payload

### 10.3 axi_ch 欄位擴展

3-channel 模式下，header `axi_ch`（3 bits）需區分 Narrow 與 Wide 來源：

| axi_ch | Name | Network | Channel | Flit Width |
|--------|------|---------|---------|------------|
| 0 | NarrowAW | Request | req | 164 |
| 1 | NarrowW | Request | req | 164 |
| 2 | NarrowAR | Request | req | 164 |
| 3 | NarrowB | Response | rsp | 164 |
| 4 | NarrowR | Response | rsp | 164 |
| 5 | WideAW | Request | wide | 408 |
| 6 | WideW | Request | wide | 408 |
| 7 | WideAR/WideR/WideB | — | — | （需擴展 axi_ch 或使用其他機制區分） |

> **Note**: 3-channel 模式下 `axi_ch` 的 3 bits 不足以區分所有 10 種 channel。實作選項：
> - **Option A**: 擴展 `axi_ch` 至 4 bits（從 rsvd 借用 1 bit）
> - **Option B**: WideAR 與 NarrowAR 共用 `axi_ch=2`，由 channel type（req vs wide）隱式區分
> - **Option C**: WideB 與 NarrowB 共用 `axi_ch=3`，由 channel type 隱式區分
>
> **建議採用 Option B+C**：WideAR 走 req channel 時與 NarrowAR 共用 `axi_ch=2`（接收端由所在 channel 自動區分）；WideB 走 rsp channel 時與 NarrowB 共用 `axi_ch=3`。

---

## 11. 3-Channel Router Architecture

### 11.1 Logical Router 結構

3-channel 模式下，每個邏輯 Router 包含 **三個獨立的 sub-router**：

```
  ┌──────────────────────────────────────────────────────────┐
  │                     Logical Router                        │
  │                                                           │
  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐
  │  │   ReqRouter     │  │   RspRouter     │  │   WideRouter   │
  │  │  (164b flit)    │  │  (164b flit)    │  │  (408b flit)   │
  │  │  NarrowAW/AR/W  │  │  NarrowR/B     │  │  WideAW/W/R    │
  │  │  + WideAR       │  │  + WideB       │  │                │
  │  │  5×5 Crossbar   │  │  5×5 Crossbar  │  │  5×5 Crossbar  │
  │  └────────────────┘  └────────────────┘  └────────────────┘
  │                                                           │
  └──────────────────────────────────────────────────────────┘
```

### 11.2 Sub-Router 特性

| Sub-Router | Flit Width | 承載 AXI Channels | 特性 |
|------------|-----------|-------------------|------|
| **ReqRouter** | 164 bits | NarrowAW, NarrowW, NarrowAR, WideAR | Narrow flit, address/write 請求 |
| **RspRouter** | 164 bits | NarrowR, NarrowB, WideB | Narrow flit, 回應封包 |
| **WideRouter** | 408 bits | WideAW, WideW, WideR | Wide flit, 大資料傳輸 |

- ReqRouter 和 RspRouter 共用同一 **164-bit 寬度設計**，可複用 RTL 模組
- WideRouter 使用現有 **408-bit 寬度設計**
- 所有 sub-router 共用相同 routing algorithm（XY routing）和 arbitration policy

### 11.3 C++ 結構

```cpp
struct Router3Ch {
    Coord       coord;
    SubRouter   req_router;    // 164-bit flit, NarrowAW/AR/W + WideAR
    SubRouter   rsp_router;    // 164-bit flit, NarrowR/B + WideB
    SubRouter   wide_router;   // 408-bit flit, WideAW/W/R
};
```

---

## 12. NMU/NSU 3-Channel 分流

### 12.1 NMU 分流邏輯

NMU 接收 AXI Master 的請求，依據 AXI 介面寬度分流至不同 physical channel：

```
NMU (接 AXI Master):
  Narrow AXI Master ──► NarrowAW/W/AR ──► req channel (164b)
                    ◄── NarrowR/B     ◄── rsp channel (164b)

  Wide AXI Master   ──► WideAW/W      ──► wide channel (408b)
                    ──► WideAR        ──► req channel (164b)
                    ◄── WideR         ◄── wide channel (408b)
                    ◄── WideB         ◄── rsp channel (164b)
```

### 12.2 NSU 分流邏輯

NSU 從三個 channel 接收 flit，重組為 AXI Master 介面驅動本地 Slave（Memory）：

```
NSU (接 AXI Slave Memory):
  req channel (164b)  ──► NarrowAW/AR/W + WideAR ──► AXI Master (Narrow/Wide)
  rsp channel (164b)  ◄── NarrowB + WideB         ◄── AXI Slave response
  wide channel (408b) ──► WideAW/W                ──► AXI Master (Wide)
  wide channel (408b) ◄── WideR                   ◄── AXI Slave read data
```

### 12.3 NMU 分流決策表

| 條件 | 目標 Channel | Flit Width |
|------|-------------|------------|
| Narrow AXI AW/AR/W | req | 164 |
| Wide AXI AW | wide | 408 |
| Wide AXI W | wide | 408 |
| Wide AXI AR | req | 164 |
| Narrow AXI B/R (incoming) | rsp | 164 |
| Wide AXI B (incoming) | rsp | 164 |
| Wide AXI R (incoming) | wide | 408 |

---

## 13. Wire Count & HoL Blocking 比較

### 13.1 Physical Link 寬度

| Link Type | valid | ready | flit | Total |
|-----------|-------|-------|------|-------|
| Narrow (req/rsp) | 1 | 1 | 164 | **166 bits** |
| Wide | 1 | 1 | 408 | **410 bits** |

### 13.2 Per-Direction Wire Count

| 設計 | Links per direction | Wires per direction |
|------|---------------------|---------------------|
| 2-Channel | 2 × 410 (Req + Rsp) | **820** |
| 3-Channel | 2 × 166 + 410 (req + rsp + wide) | **742** |

### 13.3 Router Pair Wire Count

| 設計 | Per router pair (bidirectional) | vs 2-Ch |
|------|-------------------------------|---------|
| 2-Channel | 4 × 410 = **1,640** | baseline |
| 3-Channel | 2 × 742 = **1,484** | **-9.5%** |

3-channel 反而比 2-channel **少 9.5% wire**，因為 narrow channels 使用適當寬度的 flit 而非 408-bit 全寬 flit。

### 13.4 Mesh Wire Budget 比較

| Topology | Router Pairs | 2-Channel | 3-Channel | 差異 |
|----------|-------------|-----------|-----------|------|
| 4×4 mesh | 24 | 39,360 | 35,616 | -9.5% |
| 8×8 mesh | 112 | 183,680 | 166,208 | -9.5% |
| 16×16 mesh | 480 | 787,200 | 712,320 | -9.5% |

### 13.5 HoL Blocking 比較

| 場景 | 2-Ch | 3-Ch | 改善 |
|------|------|------|------|
| NarrowW burst 阻塞 NarrowAW/AR | 有 | 有（仍共用 req） | — |
| WideW burst 阻塞 NarrowAW/AR | **有**（共用 req） | **無**（WideW 在 wide） | ✓ |
| WideR burst 阻塞 NarrowB | **有**（共用 rsp） | **無**（WideR 在 wide） | ✓ |
| WideW 阻塞 WideR | N/A | **無**（方向相反） | ✓ |

**主要改善：寬資料（256b）的 burst 不再阻塞窄控制封包（64b）。**

---

## 14. AW/W Ordering 設計考量

### 14.1 問題：Cross-Channel Write Ordering

AXI 規範要求 AW 與 W 之間維持特定的 ordering relationship。在 3-channel 設計中：

| Write Type | AW Channel | W Channel | 同一 Physical Channel? |
|-----------|-----------|----------|----------------------|
| Narrow Write | req | req | **是** — 天然保序 |
| Wide Write | wide | wide | **是** — 天然保序 |

### 14.2 設計決策

**WideAW + WideW 均走 wide channel**，確保 write address 和 write data 在同一 physical path 上傳輸，不會因 cross-channel 路由差異導致亂序。

若 WideAW 走 req 而 WideW 走 wide，可能出現：

```
Cycle 1: WideAW → req channel → 到達 NSU（快，narrow link 無競爭）
Cycle 5: WideW[0] → wide channel → 到達 NSU（慢，wide data 較大）

問題：NSU 可能在收到 WideW 前就嘗試處理 WideAW，導致 W data 遺失
```

WideAW + WideW 同走 wide channel，本設計沿用此策略。

---

## 15. 2-Ch vs 3-Ch 設計選擇指南

### 15.1 選擇矩陣

| 條件 | 推薦設計 | 理由 |
|------|---------|------|
| 僅 256b AXI 介面 | **2-Channel** | 無 narrow 流量，3-Ch 無效益 |
| 同時有 64b + 256b AXI | **3-Channel** | 分離 narrow/wide，降低 HoL blocking |
| Wire budget 受限 | **3-Channel** | 少 9.5% wires |
| Router 設計簡化 | **2-Channel** | 2 sub-router vs 3 sub-router |
| 高 burst 工作負載 | **3-Channel** | Wide burst 不阻塞 narrow control |
| FPGA prototype 優先 | **2-Channel** | 較簡單，FPGA 資源較少 |
| ASIC tape-out | **3-Channel** | Wire 節省 + HoL 改善 |

### 15.2 效能 Trade-off 總結

| Aspect | 2-Channel | 3-Channel |
|--------|-----------|-----------|
| Sub-routers per logical router | 2 | 3 |
| Flit widths | 408b (uniform) | 164b + 408b |
| Wire count (per pair) | 1,640 | 1,484 (-9.5%) |
| RTL module reuse | ReqRouter = RspRouter | ReqRouter = RspRouter（164b）+ WideRouter（408b） |
| HoL blocking (wide vs narrow) | 有 | **消除** |
| AXI interface support | Single width (256b) | Dual width (64b + 256b) |
| Buffer total (per router) | 2 × 5 ports × depth | 3 × 5 ports × depth |
| Complexity | 低 | 中 |

### 15.3 本設計建議

- **V1 實作**：採用 2-channel（Section 1-7），確保架構簡潔可驗證
- **V2 進階**：引入 3-channel（Section 8-14），支援 Narrow+Wide 雙 AXI 介面

兩種模式由 compile-time configuration 選擇，core routing/arbitration 邏輯完全共用。

---

## Related Documents

- [Flit Format](02_flit.md) — Flit 結構、header 欄位定義、payload format、physical link format
- [Router Specification](03_router.md) — Router pipeline、ReqRouter/RspRouter 架構、wormhole switching
- [Network Interface Specification](04_network_interface.md) — NI 的 AXI-to-Flit 轉換與 network 分流
- [Internal Interface](05_internal_interface.md) — Abstract Interface、8-phase cycle 定義
- [QoS Design](07_qos.md) — QoS-Aware arbitration policy
