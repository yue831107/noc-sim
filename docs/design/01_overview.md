---
document_id: NOC-SPEC-01
title: System Overview
version: 1.0
status: Draft
last_updated: 2026-03-09
prerequisite: [00_architecture.md]
---

# System Overview

本文件說明 NoC Behavior Model 的整體系統架構。

---

## 1. Overview

```
  ┌──────────────────────────────────────────────────┐
  │                     NoC Mesh                      │
  │                                                    │
  │  (0,3)────(1,3)────(2,3)────(3,3)                │
  │    │  R    │  R      │  R     │  R                │
  │   NI      NI        NI       NI                    │
  │  (0,2)────(1,2)────(2,2)────(3,2)                │
  │    │  R    │  R      │  R     │  R                │
  │   NI      NI        NI       NI                    │
  │  (0,1)────(1,1)────(2,1)────(3,1)                │
  │    │  R    │  R      │  R     │  R                │
  │   NI      NI        NI       NI                    │
  │  (0,0)────(1,0)────(2,0)────(3,0)                │
  │    │  R    │  R      │  R     │  R                │
  │   NI      NI        NI       NI                    │
  └──────────────────────────────────────────────────┘

  R  = Router (每個 Router 可有 0~4 個 LOCAL port，由 port_id 識別)
  NI = Network Interface (連接 Router 的 LOCAL port)
  ── = Router 間 mesh link (Req/Rsp 各一條，雙向 full-duplex)
```

每個 Router 為統一結構，具備 4 個 mesh 方向 port（N, S, E, W）與 0~4 個 LOCAL port。NI 透過 Router 的 LOCAL port 連接，各 LOCAL port 由 header `port_id`（2 bits）識別。

外部 AXI Master/Slave 經由 NI 進行 Protocol Conversion（AXI ↔ Flit），NI 再經 Router 的 LOCAL port 注入/接收 mesh 網路流量。

---

## 2. 拓撲參數

| Parameter | Default | Description |
|-----------|---------|-------------|
| `mesh_cols` | 4 | Number of columns |
| `mesh_rows` | 4 | Number of rows |

### 2.1 Flit 與 Link 參數

以下為固定設計參數，與 [Flit 格式](02_flit.md) 一致：

| Parameter | Value | Description |
|-----------|-------|-------------|
| `FLIT_WIDTH` | 408 bits | Total flit width (Header + Payload) |
| `HEADER_WIDTH` | 56 bits | Header width |
| `PAYLOAD_WIDTH` | 352 bits (44 bytes) | Maximum payload width (W/R channel) |
| `AXI_DATA_WIDTH` | 256 bits (32 bytes) | AXI data width |
| `AXI_ADDR_WIDTH` | 64 bits | AXI address width |
| `AXI_ID_WIDTH` | 8 bits | AXI transaction ID width |
| `NODE_ID_WIDTH` | 8 bits | Node ID width ([7:4]=y, [3:0]=x) |
| `QOS_WIDTH` | 4 bits | QoS priority width |
| `ROB_IDX_WIDTH` | 5 bits | RoB index width (32 entries) |
| `ECC_WIDTH` | 32 bits | Total ECC width (SECDED) |
| Physical Link | 410 bits | valid + ready + 408-bit flit |

---

## 3. 核心特性

| Feature | Description |
|---------|-------------|
| Uniform Router | All routers identical, multi-port LOCAL supported (port_id) |
| XY Routing | Deterministic routing, X-first then Y |
| Wormhole Switching | Packet-level path lock, `last` bit releases |
| Separate Req/Rsp Channels | Independent Req/Rsp physical links, eliminates protocol deadlock |
| Credit-Based Flow Control | Per-port credit tracking (Credit-Based mode) |

---

## 4. 已知限制

- **Deadlock**: 依賴 XY routing 的 deadlock-free 特性，不實作 deadlock recovery
- **Topology**: 僅支援 2D mesh，不支援 torus 或 irregular topology

---

## 相關文件

- [Router 規格](03_router.md)
- [Network Interface 規格](04_network_interface.md)
- [Flit 格式](02_flit.md)
- [Simulation Platform](08_simulation.md)

---

## Change Log

| Version | Date | Description |
|---------|------|-------------|
| 1.0 | 2026-03-09 | Initial release |
