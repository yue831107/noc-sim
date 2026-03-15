---
document_id: NOC-SPEC-04
title: Network Interface
version: 1.0
status: Draft
last_updated: 2026-03-09
prerequisite: [02_flit.md, 03_router.md]
---

# Network Interface

## 1. Overview

### 1.1 用途

Network Interface (NI) 負責 AXI 與 NoC 之間的 Protocol Conversion，包含地址轉換、Flit Packing/Unpacking、Reorder Buffer 與 ECC 處理。

### 1.2 在系統中的位置

```
AXI Master ──► NI (NMU+NSU) ──► Router LOCAL port ──► Mesh
AXI Slave  ◄──                ◄──
```

NI 透過 Router 的 Eject port 連接，使用與 Router-Router **完全相同**的 port interface。每個 NI 包含 NMU（Network Master Unit）與 NSU（Network Slave Unit），可獨立啟用/停用。

> NI / Router 完整 block diagram 見 [Router 規格 §1.2](03_router.md)。

### 1.3 命名慣例（AMD Versal 風格）

| 名稱 | AXI 側角色 | 網路側角色 |
|------|-----------|-----------|
| **NMU** | AXI Slave（接收 Master 請求） | 發起 NoC transaction |
| **NSU** | AXI Master（驅動 Slave 介面） | 完成 NoC transaction |

### 1.4 設計目標

- AXI4 full protocol conversion（AW/W/AR/B/R）
- Reorder Buffer 支援 out-of-order response
- SECDED ECC end-to-end protection
- 可配置為 NMU-only / NSU-only / Full

---

## 2. Architecture & Block Diagram

### 2.1 Block Diagram

```
              ┌──────────────────────────────────────────────────┐
              │                Network Interface                  │
              │                                                  │
AXI Master ──►│  ┌──────────────────────────────────────────┐   │
(AW/AR/W)     │  │              NMU (Manager)                │   │
              │  │                                           │   │
              │  │  AddrTrans → QoS Gen → Flit Pack → ECC   │───►  Req Router
              │  │                                           │   │
AXI Master ◄──│  │  RoB ◄── Flit Unpack ◄── ECC Check       │   │
(B/R)         │  └──────────────────────────────────────────┘◄──  Rsp Router
              │                                                  │
AXI Slave  ◄──│  ┌──────────────────────────────────────────┐   │
(AW/AR/W)     │  │            NSU (Subordinate)              │◄──  Req Router
              │  │                                           │   │
              │  │  Flit Unpack → ReqInfo Store → MemOp      │   │
              │  │                                           │   │
AXI Slave  ──►│  │  Flit Pack ◄── ECC Gen ◄── AXI Response  │───►  Rsp Router
(B/R)         │  └──────────────────────────────────────────┘   │
              └──────────────────────────────────────────────────┘
```

> **Implementation Note：** NMU 與 NSU 可獨立啟用（`EN_MGR_PORT` / `EN_SBR_PORT`），未啟用時對應 sub-module 不 instantiate。NI 的 `tick()` 對應 Phase 8。

### 2.2 Sub-Module 職責

**NMU（Network Master Unit）：**

| Sub-Module | 職責 |
|------------|------|
| Address Translator | AXI address → dst_id + local_addr |
| QoS Generator | 產生 header qos 值（4 modes） |
| Flit Packer (AW/W/AR) | AXI request → 400-bit flit |
| ECC Generator | SECDED ECC 產生（W channel wdata） |
| Flit Unpacker (B/R) | Response flit → AXI B/R |
| ECC Checker | SECDED ECC 驗證（R channel rdata） |
| RoB | Reorder Buffer，per-ID in-order release |
| Injection Buffer | 已打包 flit 的注入 FIFO |

**NSU（Network Slave Unit）：**

| Sub-Module | 職責 |
|------------|------|
| Flit Unpacker (AW/W/AR) | Request flit → AXI request |
| Request Info Store | 暫存 request header（rob_idx, src_id, qos）供 response 配對 |
| W Reassembly Buffer | Multi-flit W burst 重組 |
| ECC Checker | SECDED ECC 驗證（W channel wdata） |
| Flit Packer (B/R) | AXI response → flit |
| ECC Generator | SECDED ECC 產生（R channel rdata） |

---

## 3. Parameters

NI 使用 config struct 組織參數，由上層 instantiation 時傳入。

### 3.1 AXI Configuration（AXI_CFG）

| Parameter | Type | Default | RTL 對應 | Description |
|-----------|------|---------|---------|-------------|
| `ADDR_WIDTH` | int | 64 | `AXI_CFG.ADDR_WIDTH` | AXI address 寬度 |
| `DATA_WIDTH` | int | 256 | `AXI_CFG.DATA_WIDTH` | AXI data 寬度（32 bytes） |
| `USER_WIDTH` | int | 8 | `AXI_CFG.USER_WIDTH` | AXI user signal 寬度 |
| `IN_ID_WIDTH` | int | 8 | `AXI_CFG.IN_ID_WIDTH` | AXI manager (incoming) txnID 寬度 |
| `OUT_ID_WIDTH` | int | 8 | `AXI_CFG.OUT_ID_WIDTH` | AXI subordinate (outgoing) txnID 寬度 |

### 3.2 NI Configuration（NI_CFG）

| Parameter | Type | Default | RTL 對應 | Description |
|-----------|------|---------|---------|-------------|
| `EN_SBR_PORT` | bool | true | `NI_CFG.EN_SBR_PORT` | 啟用 NSU（AXI subordinate 端） |
| `EN_MGR_PORT` | bool | true | `NI_CFG.EN_MGR_PORT` | 啟用 NMU（AXI manager 端） |
| `MAX_TXNS` | int | 32 | `NI_CFG.MAX_TXNS` | 最大 outstanding transactions |
| `MAX_UNIQUE_IDS` | int | 1 | `NI_CFG.MAX_UNIQUE_IDS` | 對下游發出的 unique txnID 數量 |
| `MAX_TXNS_PER_ID` | int | 32 | `NI_CFG.MAX_TXNS_PER_ID` | 每 txnID 最大 outstanding 數 |
| `B_ROB_TYPE` | enum | NoRoB | `NI_CFG.B_ROB_TYPE` | B response RoB 類型（見 3.4） |
| `B_ROB_SIZE` | int | 0 | `NI_CFG.B_ROB_SIZE` | B RoB 深度 |
| `R_ROB_TYPE` | enum | NoRoB | `NI_CFG.R_ROB_TYPE` | R response RoB 類型 |
| `R_ROB_SIZE` | int | 0 | `NI_CFG.R_ROB_SIZE` | R RoB 深度 |
| `CUT_AX` | bool | false | `NI_CFG.CUT_AX` | AW/AR 加 spill register（cut timing path） |
| `CUT_RSP` | bool | false | `NI_CFG.CUT_RSP` | Response 加 spill register |

### 3.3 Route Configuration（ROUTE_CFG）

| Parameter | Type | Default | RTL 對應 | Description |
|-----------|------|---------|---------|-------------|
| `ROUTE_ALGO` | enum | XYRouting | `ROUTE_CFG.ROUTE_ALGO` | Routing algorithm |
| `USE_ID_TABLE` | bool | false | `ROUTE_CFG.USE_ID_TABLE` | 使用 SAM table 轉換 dst_id |
| `XY_ADDR_OFFSET_X` | int | 32 | `ROUTE_CFG.XY_ADDR_OFFSET_X` | X coordinate 在 address 中的 bit offset |
| `XY_ADDR_OFFSET_Y` | int | 36 | `ROUTE_CFG.XY_ADDR_OFFSET_Y` | Y coordinate 在 address 中的 bit offset |
| `NUM_SAM_RULES` | int | 0 | `ROUTE_CFG.NUM_SAM_RULES` | SAM rules 數量（USE_ID_TABLE 時） |

### 3.4 RoB Type 列舉（`rob_type_e`）

| Value | Name | Description |
|-------|------|-------------|
| `NormalRoB` | 完整 RoB | 支援 response reordering，保留 AXI per-ID OoO 特性 |
| `SimpleRoB` | FIFO RoB | 不支援 reordering，不同 txnID 序列化。適合 B response |
| `NoRoB` | 無 RoB | Stall 直到前一筆完成。overhead 最低 |

### 3.5 Type Parameters

| Parameter | Description | 來源 |
|-----------|-------------|------|
| `flit_t` | Flit data type（400 bits） | [Flit Format](02_flit.md) |
| `hdr_t` | Flit header type（48 bits） | [Flit Format](02_flit.md) |
| `id_t` | Node ID type | `logic[ID_WIDTH-1:0]` |
| `noc_req_t` | Request link type（valid + flit） | Req network |
| `noc_rsp_t` | Response link type（valid + flit） | Rsp network |
| `axi_in_req_t` | AXI manager request channels | AXI typedef |
| `axi_in_rsp_t` | AXI manager response channels | AXI typedef |
| `axi_out_req_t` | AXI subordinate request channels | AXI typedef |
| `axi_out_rsp_t` | AXI subordinate response channels | AXI typedef |
| `sam_rule_t` | SAM address rule type | USE_ID_TABLE 時使用 |

### 3.6 Simulation Parameters（CA Model 專用）

| Parameter | Source | Default | Description |
|-----------|--------|---------|-------------|
| `NMU_BUFFER_DEPTH` | NocConfig | 2 | NMU injection buffer 深度 |
| `MAX_OUTSTANDING` | NocConfig | 8 | Traffic Manager 同時 in-flight 數（≤ MAX_TXNS） |
| `MAX_BURST_LEN` | NocConfig | 16 | AXI burst 最大 beat 數（= NSU reassembly depth） |

**參數約束：**
- `MAX_OUTSTANDING ≤ MAX_TXNS`（MAX_TXNS 為硬體上限，MAX_OUTSTANDING 為軟體配置）
- NSU reassembly depth = `MAX_BURST_LEN`

---

## 4. Interface Specification

### 4.1 AXI Side Ports

AXI side port 定義：

**AXI Manager Port（NMU 側，接收來自 AXI Master 的請求）：**

| Signal | Type | Description |
|--------|------|-------------|
| `axi_in_req_i` | `axi_in_req_t` | AXI request input（AW/W/AR + valid） |
| `axi_in_rsp_o` | `axi_in_rsp_t` | AXI response output（B/R + ready） |

**AXI Subordinate Port（NSU 側，驅動 AXI Slave）：**

| Signal | Type | Description |
|--------|------|-------------|
| `axi_out_req_o` | `axi_out_req_t` | AXI request output（AW/W/AR 至 local memory） |
| `axi_out_rsp_i` | `axi_out_rsp_t` | AXI response input（B/R from local memory） |

### 4.2 NoC Side Ports（NoC Link）

使用 Req/Rsp 雙 link 架構：

| Signal | Direction | Description |
|--------|-----------|-------------|
| `noc_req_o` | output | Request flit 輸出（NMU → Req Router） |
| `noc_rsp_o` | output | Response flit 輸出（NSU → Rsp Router） |
| `noc_req_i` | input | Request flit 輸入（Req Router → NSU） |
| `noc_rsp_i` | input | Response flit 輸入（Rsp Router → NMU） |

每條 noc link 內部包含 `valid`、`ready`、`data`（flit_t），與 Router port interface 完全相容。

### 4.3 Control Inputs

| Signal | Type | Description |
|--------|------|-------------|
| `id_i` | `id_t` | 本 NI 的 Node ID（XY 座標） |
| `route_table_i` | `route_t[]` | Routing table（SourceRouting 時使用） |

### 4.4 AXI Channel → NoC Link 映射

| AXI Channel | Direction | NoC Link | Flit axi_ch |
|-------------|-----------|-----------|-------------|
| AW | NMU → NSU | `noc_req` | 0 |
| W | NMU → NSU | `noc_req` | 1 |
| AR | NMU → NSU | `noc_req` | 2 |
| B | NSU → NMU | `noc_rsp` | 3 |
| R | NSU → NMU | `noc_rsp` | 4 |

### 4.5 NMU/NSU 方向總覽

| 元件 | NoC Link | 方向 | 說明 |
|------|-----------|------|------|
| NMU | `noc_req_o` | NI → Router | 注入 AW/W/AR flit |
| NMU | `noc_rsp_i` | Router → NI | 接收 B/R flit |
| NSU | `noc_req_i` | Router → NI | 接收 AW/W/AR flit |
| NSU | `noc_rsp_o` | NI → Router | 注入 B/R flit |

---

## 5. Functional Requirements

### FR-01: Address Translation

**描述：** NMU 將 AXI address 轉換為 NoC dst_id + local_addr。

**Address Format（XYRouting, !USE_ID_TABLE）：**

| Bits | Field | Description |
|------|-------|-------------|
| [XY_ADDR_OFFSET_X+3 : XY_ADDR_OFFSET_X] | dst_x | X coordinate（4 bits） |
| [XY_ADDR_OFFSET_Y+3 : XY_ADDR_OFFSET_Y] | dst_y | Y coordinate（4 bits） |
| [31:0] | local_addr | 節點內部地址 |

預設 `XY_ADDR_OFFSET_X=32`, `XY_ADDR_OFFSET_Y=36`，等同 `addr[39:32]` = Node ID。

**USE_ID_TABLE 模式：** 透過 SAM (System Address Map) 查表，將 address 映射至 dst_id。SAM rules 由 `sam_rule_t` 定義。

> **Testability：** 注入 `addr[39:32]=0x21` 驗證 `dst_id=(x=1, y=2)`。測試 SAM table 模式的 address-to-node 映射。

### FR-02: Flit Packing（AXI → Flit）

**描述：** NMU 將 AXI request 打包為 400-bit flit，NSU 將 AXI response 打包為 flit。

**Payload 使用率：**

| Channel | Network | Used (bits) | Padding (bits) | 主要欄位 |
|---------|---------|-------------|----------------|---------|
| AW | REQ | 108 | 244 | awid, awaddr, awlen, awsize, awburst, awcache, awprot, awuser |
| W | REQ | 352 | 0 | wlast, wuser, wdata[256], wstrb[32], wecc[32] |
| AR | REQ | 108 | 244 | arid, araddr, arlen, arsize, arburst, arcache, arprot, aruser |
| B | RSP | 64 | 288 | bid, bresp, buser, ecc_fail |
| R | RSP | 352 | 0 | rlast, rid, rresp, ruser, rdata[256], recc[32] |

Header 的欄位來源（NMU request path）：`qos` ← QoS Generator、`src_id` ← 本地座標、`dst_id` ← Address Translator、`last` ← single-flit(AW/AR)=1 / W 末尾=1、`rob_idx` ← RoB allocate。

Payload bit-level layout 詳見 [Flit Format](02_flit.md) Section 3。

> **Testability：** AXI AW/W/AR → flit → unpack round-trip 驗證所有欄位不失真。驗證 W payload 100% utilization（352 bits 全用）、AW payload padding 為 0。

### FR-03: Flit Unpacking（Flit → AXI）

**描述：** NSU 將 request flit 還原為 AXI request，NMU 將 response flit 還原為 AXI B/R。

**NSU 行為：**
- AW flit → 記錄 burst info + request header（rob_idx, src_id, qos）至 Request Info Store
- W flit → 逐 beat 重組至 reassembly buffer，完成後發起 AXI write 至 local memory
- AR flit → 記錄 request header，發起 AXI read 至 local memory

**NMU 行為：**
- B flit → 更新 RoB entry → 依 per-ID 順序 release
- R flit → 累積 rdata beats → 最後一個 beat 觸發 RoB release

> **Testability：** Pack → transfer → unpack end-to-end，比較 AXI request fields 完全一致。

### FR-04: Burst Handling

**描述：** NI 處理 AXI burst transaction，將單一 AXI 交易轉換為一或多個 flit。

**Flit 數量關係：**

| Transaction | Flit Count | Wormhole |
|-------------|------------|----------|
| Write (awlen=N) | 1 AW + (N+1) W = N+2 | AW 不鎖路，W burst 獨立鎖路 |
| Read Request | 1 AR | 不鎖路 |
| Read Response (arlen=N) | N+1 R | R burst 鎖路 |
| Write Response | 1 B | 不鎖路 |

**AW 與 W 獨立性：** AW 與 W 為兩個獨立 wormhole packet。AW→W ordering 由 NMU 注入順序保證（同 output port，FIFO 天然保證）。Router 無需 AW/W 關聯邏輯。

**Burst types：** FIXED（同地址）、INCR（遞增）、WRAP（wrap-around）均支援。NSU 依 burst type 計算每 beat address。

> **Testability：** 測試 `awlen=0`（2 flits）與 `awlen=15`（17 flits）。驗證 AW→W ordering 與 wormhole 鎖路行為。

### FR-05: Reorder Buffer (RoB)

**描述：** NMU 側 RoB 管理 outstanding transactions 的 response reordering，保證 per-AXI-ID in-order release。

**RoB 類型（由 `B_ROB_TYPE` / `R_ROB_TYPE` 選擇）：**

| Type | 行為 | 適用 |
|------|------|------|
| NormalRoB | 支援 reordering，保留 per-ID OoO | R response（多 destination） |
| SimpleRoB | FIFO，不同 txnID 序列化 | B response |
| NoRoB | Stall 直到前一筆完成 | 低 overhead 場景 |

**RoB Entry State Machine：**

```
FREE ──► ALLOCATED ──► RESPONSE_RECEIVED ──► READY_TO_RELEASE ──► FREE
          (AW/AR 發送)   (B/R 到達)          (all beats 收齊)    (per-ID 釋放)
```

**Per-ID Ordering：** 同 `axi_id` 的多筆 outstanding transactions 依 rob_idx 分配順序依序 release。不同 axi_id 之間可 out-of-order。

> **Implementation Note：** RoB 內部維護 per-ID linked list。`rob_idx` 作為 flit header 欄位（5 bits），最大 32 entries。NormalRoB 需 O(MAX_TXNS) storage；SimpleRoB 可用 circular buffer 實作。

**Backpressure：** RoB 無 free entry 時，deassert AXI `awready` / `arready`。

> **Testability：** 驗證 RoB 4 狀態轉移（FREE→ALLOCATED→RECEIVED→RELEASE→FREE）。測試不同 `axi_id` 的 OoO release。測試 RoB full 時的 backpressure（`awready=0`）。

### FR-06: ECC Generate / Check

**描述：** Source NI 產生 SECDED ECC，Destination NI 驗證。Router 為純透通，不檢查/修改 ECC。

**ECC 參數：**

| Property | Value |
|----------|-------|
| Data granule | 64 bits |
| ECC per granule | 8 bits (Hsiao SECDED) |
| Granules per 256-bit data | 4 |
| Total ECC width | 32 bits |
| 1-bit error | Correctable（per granule） |
| 2-bit error | Detectable（per granule） |

**ECC 時機：**

| NI | Channel | 動作 | 欄位 |
|----|---------|------|------|
| NMU | W (request) | Generate | payload wecc[32] |
| NSU | W (request) | Check | payload wecc[32] |
| NSU | R (response) | Generate | payload recc[32] |
| NMU | R (response) | Check | payload recc[32] |

**Error 處理：**

| Result | W Channel (NSU) | R Channel (NMU) |
|--------|-----------------|-----------------|
| 1-bit corrected | 使用校正資料，log CSR | 使用校正資料，log CSR |
| 2-bit uncorrectable | B response `ecc_fail=1`, `bresp=SLVERR` | `rresp=SLVERR`, log CSR |

> **Testability：** Generate→check round-trip（inject known data，驗證 ECC match）。Inject 1-bit error → 驗證自動校正。Inject 2-bit error → 驗證 SLVERR + CSR log。

### FR-07: QoS Generation

**描述：** NMU 的 QoS Generator 在 AW/AR flit 打包時計算 header `qos` 值。

**4 種模式：**

| Mode | qos 來源 | 說明 |
|------|---------|------|
| Bypass | `awqos` / `arqos` | AXI QoS 直通 |
| Fixed | CSR 配置值 | 固定優先級 |
| Limiter | 超標 → 阻塞 request | 硬性頻寬上限 |
| Regulator | 超標 → 降低 urgency/hurry | 軟性頻寬控制 |

W flit 的 `qos` 繼承對應 AW flit 值。Response flit 的 `qos` 繼承 request（由 NSU Request Info Store 提供）。

詳見 [QoS Design](06_qos.md)。

> **Testability：** 測試 4 種模式各自的 QoS output。Bypass 模式驗證 `flit.qos == awqos`。Limiter 模式驗證超標時 request 被 stall（阻塞注入）。Regulator 模式驗證超標時 urgency 降低。

---

## 6. Performance

### 6.1 Injection Latency

| Path | Latency | 說明 |
|------|---------|------|
| AXI AW → noc_req_o | CUT_AX ? 2 : 1 cycle | Pack + optional spill register |
| AXI W → noc_req_o | 1 cycle | Pack（injection buffer drain） |
| noc_rsp_i → AXI B | CUT_RSP ? 2 : 1 cycle | Unpack + RoB release |

### 6.2 Injection Rate

| 指標 | 值 |
|------|----|
| Max injection rate | 1 flit/cycle per link |
| NMU backpressure | Injection buffer full 或 RoB full |
| NSU backpressure | Local memory AXI slave not ready |

### 6.3 Burst Efficiency

| Transaction | Flit Overhead | 說明 |
|-------------|--------------|------|
| Single write (awlen=0) | 2 flits (AW+W) | AW payload 68% padding |
| Burst write (awlen=15) | 17 flits (AW+16W) | W payload 100% utilization |
| Single read req | 1 flit (AR) | AR payload 68% padding |
| Burst read rsp (arlen=15) | 16 flits (R) | R payload 100% utilization |

---

## 7. Related Documents

- [System Overview](01_overview.md) — Mesh 拓撲與系統架構
- [Flit Format](02_flit.md) — Header/Payload bit-level 定義、ECC 設計
- [Router Specification](03_router.md) — Router port interface
- [QoS Design](06_qos.md) — QoS Generator 模式、CSR Memory Map
- [Simulation](08_simulation.md) — NoC System API、NocConfig、Channel\<T\>

---

## Change Log

| Version | Date | Description |
|---------|------|-------------|
| 1.0 | 2026-03-09 | Initial release |
