# 系統概述

本文件說明 NoC Behavior Model 的整體系統架構。

---

## 1. 系統架構圖

```
  ┌─────────────────────────────────────────────────────────────┐
  │                        NoC Mesh                             │
  │                                                             │
  │  (0,3)────(1,3)───(2,3)───(3,3)───(4,3)                   │
  │    │  R    │  R     │  R    │  R    │  R                   │
  │   NI      NI       NI      NI      NI                      │
  │  (0,2)────(1,2)───(2,2)───(3,2)───(4,2)                   │
  │    │  R    │  R     │  R    │  R    │  R                   │
  │   NI      NI       NI      NI      NI                      │
  │  (0,1)────(1,1)───(2,1)───(3,1)───(4,1)                   │
  │    │  R    │  R     │  R    │  R    │  R                   │
  │   NI      NI       NI      NI      NI                      │
  │  (0,0)────(1,0)───(2,0)───(3,0)───(4,0)                   │
  │    │  R    │  R     │  R    │  R    │  R                   │
  │   NI      NI       NI      NI      NI                      │
  └─────────────────────────────────────────────────────────────┘

  R  = Router (每個 Router 可有 0~4 個 LOCAL port，由 port_id 識別)
  NI = Network Interface (連接 Router 的 LOCAL port)
  ── = Router 間 mesh link (Req/Rsp 各一條，雙向 full-duplex)
```

每個 Router 為統一結構，具備 4 個 mesh 方向 port（N, S, E, W）與 0~4 個 LOCAL port。NI 透過 Router 的 LOCAL port 連接，各 LOCAL port 由 header `port_id`（2 bits）識別。

外部 AXI Master/Slave 經由 NI 進行 Protocol Conversion（AXI ↔ Flit），NI 再經 Router 的 LOCAL port 注入/接收 mesh 網路流量。

---

## 2. 拓撲參數

| 參數 | 預設值 | 說明 |
|------|--------|------|
| `mesh_cols` | 5 | 欄數 |
| `mesh_rows` | 4 | 列數 |

### 2.1 Flit 與 Link 參數

以下為固定設計參數，與 [Flit 格式](02_flit.md) 一致：

| 參數 | 值 | 說明 |
|------|-----|------|
| `FLIT_WIDTH` | 408 bits | Flit 總寬度 (Header + Payload) |
| `HEADER_WIDTH` | 56 bits | Header 寬度 |
| `PAYLOAD_WIDTH` | 352 bits (44 bytes) | 最大 Payload 寬度 (W/R channel) |
| `AXI_DATA_WIDTH` | 256 bits (32 bytes) | AXI 資料寬度 |
| `AXI_ADDR_WIDTH` | 64 bits | AXI 位址寬度 |
| `AXI_ID_WIDTH` | 8 bits | AXI transaction ID 寬度 |
| `NODE_ID_WIDTH` | 8 bits | 節點 ID 寬度 ({y[3:0], x[3:0]}) |
| `QOS_WIDTH` | 4 bits | QoS 優先級寬度 |
| `ROB_IDX_WIDTH` | 5 bits | RoB index 寬度 (32 entries) |
| `ECC_WIDTH` | 32 bits | ECC 總寬度 (SECDED) |
| Physical Link | 410 bits | valid + ready + 408-bit flit |

---

## 3. 核心特性

| 特性 | 說明 |
|------|------|
| Uniform Router | 所有 Router 結構統一，支援 multi-port LOCAL（port_id 識別） |
| XY Routing | Deterministic routing，X-first then Y |
| Wormhole Switching | Packet-level path lock，`last` bit 釋放 |
| Separate Req/Rsp Channels | Request / Response 獨立 physical link，消除 protocol deadlock |
| Credit-Based Flow Control | Per-port credit tracking（Version B） |

---

## 4. 已知限制

- **Deadlock**: 依賴 XY routing 的 deadlock-free 特性，不實作 deadlock recovery
- **Topology**: 僅支援 2D mesh，不支援 torus 或 irregular topology

---

## 相關文件

- [Router 規格](03_router.md)
- [Network Interface 規格](04_network_interface.md)
- [Flit 格式](02_flit.md)
- [內部介面架構](05_internal_interface.md)
