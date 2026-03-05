# Memory Copy 操作

本文件說明 Host Memory 到 Local Memory 的複製機制。

---

## 1. 架構概述

### 1.1 簡化架構

```
┌─────────────┐
│ Source       │  (AXI Master)
└──────┬──────┘
       │
       ▼ AXI Write
┌──────────────┐
│     NMU      │  ← Protocol Conversion (AXI → Flit)
└──────┬───────┘
       │
       ▼ Req Flit (via Router LOCAL port)
┌──────────────┐
│  NoC Mesh    │  ← XY Routing
└──────┬───────┘
       │
   ┌───┴───┐
   ▼       ▼
┌─────┐ ┌─────┐
│Node0│ │Node1│ ... (N nodes)
│ LM  │ │ LM  │     Local Memory (經 NSU)
└─────┘ └─────┘
```

### 1.2 完整 Memory Copy 架構

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           SOURCE SIDE                                        │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                       Source Memory                                     │ │
│  │   ┌──────────────────────────────────────────────────────────────┐     │ │
│  │   │  Source Data: [block0][block1][block2]...[blockN]             │     │ │
│  │   │  (copy_size bytes，分割成 block_size 區塊)                     │     │ │
│  │   └──────────────────────────────────────────────────────────────┘     │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                         │
│                                    ▼  AXI Write Request (AW + W)             │
│  ┌─────────────────────────────────┴───────────────────────────────────────┐ │
│  │                    NMU (AXI Slave Interface)                            │ │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌───────────────────────┐  │ │
│  │  │  Address         │  │  Flit Packer     │  │  Outstanding Tracker  │  │ │
│  │  │  Translation     │  │  (AXI→Flit)      │  │  (max_outstanding)    │  │ │
│  │  │  NodeID→(x,y)    │  │                  │  │                       │  │ │
│  │  └──────────────────┘  └──────────────────┘  └───────────────────────┘  │ │
│  └─────────────────────────────────┬───────────────────────────────────────┘ │
└────────────────────────────────────┼─────────────────────────────────────────┘
                                     │ via Router LOCAL port (port_id)
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  NOC MESH (5×4)                                                              │
│                                                                              │
│  (0,3)────(1,3)───(2,3)───(3,3)───(4,3)                                    │
│    │  R    │  R     │  R    │  R    │  R                                    │
│   NI      NI       NI      NI      NI                                       │
│  (0,2)────(1,2)───(2,2)───(3,2)───(4,2)                                    │
│    │  R    │  R     │  R    │  R    │  R                                    │
│   NI      NI       NI      NI      NI                                       │
│  (0,1)────(1,1)───(2,1)───(3,1)───(4,1)                                    │
│    │  R    │  R     │  R    │  R    │  R                                    │
│   NI      NI       NI      NI      NI                                       │
│  (0,0)────(1,0)───(2,0)───(3,0)───(4,0)                                    │
│    │  R    │  R     │  R    │  R    │  R                                    │
│   NI      NI       NI      NI      NI                                       │
│                                                                              │
│  R = Router, NI = Network Interface (含 Local Memory)                        │
└─────────────────────────────────────────────────────────────────────────────┘

資料流向 (Write Request):
    Source → NMU → Router (LOCAL) → NoC Mesh (Req) → Router (LOCAL) → NSU → Local Memory

Response 流向 (B Response):
    Local Memory → NSU → Router (LOCAL) → NoC Mesh (Rsp) → Router (LOCAL) → NMU → Source
```

### 1.3 交錯傳輸時序

(`parallel_nodes=4`):
```
Cycle │ Node0    Node1    Node2    Node3    Node4   ...
──────┼──────────────────────────────────────────────────
  0   │ blk0 ──►
  1   │          blk0 ──►
  2   │                   blk0 ──►
  3   │                            blk0 ──►
  4   │ blk1 ──►                            (等待)
  5   │          blk1 ──►
  6   │                   blk1 ──►
  7   │                            blk1 ──►
 ...  │                                     (Node0-3 完成後，
      │                                      Node4-7 開始)
```

---

## 2. 架構特性

Source NMU 透過 Router 的 LOCAL port 注入 flit 至 mesh，經 XY routing 到達 destination Router，再由 LOCAL port 送至 NSU。

**效能影響因素**:
- NMU 數量與位置決定注入頻寬
- `max_outstanding` 決定 pipeline 深度
- Mesh 拓撲影響路由 hop 數與 contention

---

## 3. 交錯 vs 順序傳輸

由於 V1 架構限制，"平行" 傳輸實為**交錯** (Interleaved) 傳輸:

**順序** (`parallel_nodes=1`):
```
Node0.blk0 → Node0.blk1 → Node0.blk2 → ... → Node0 完成
Node1.blk0 → Node1.blk1 → Node1.blk2 → ... → Node1 完成
...
```

**交錯** (`parallel_nodes=4`):
```
Node0.blk0 → Node1.blk0 → Node2.blk0 → Node3.blk0 →
Node0.blk1 → Node1.blk1 → Node2.blk1 → Node3.blk1 →
...
```

---

## 4. 交錯的效能優勢

| 模式 | Cycles | Throughput | 原因 |
|------|--------|------------|------|
| 順序 | 168 | 9.58 B/cycle | 單一路徑，等待 Response |
| 交錯 (4) | 87 | 18.60 B/cycle | 路徑分散，減少等待 |

**效能提升來源**:
1. **路徑分散**: 不同目的地使用不同 mesh 路徑
2. **減少 Head-of-line Blocking**: 不卡在單一目的地
3. **Pipeline 效果**: 多筆未完成交易同時在網路中

---

## 5. Flit 大小與傳輸計算

以下參數與 [Flit 格式](02_flit.md) 一致：

| 參數 | 值 | 說明 |
|------|-----|------|
| FLIT_WIDTH | 408 bits | Header + Payload |
| Header | 56 bits (7 bytes) | 路由與控制資訊 |
| Payload | 352 bits (44 bytes) | 最大 payload（W/R channel） |
| AXI_DATA_WIDTH | 256 bits (32 bytes) | 每 W/R flit 攜帶的資料量 |

每個 W flit 攜帶 256 bits (32 bytes) 的 wdata。一次 AXI burst write（awlen=15，即 16 beats）需要 1 個 AW flit + 16 個 W flit + 1 個 B flit = 18 flits。

**傳輸計算範例：**

```
transfer_size = 4096 bytes (4 KB)
AXI_DATA_WIDTH = 256 bits = 32 bytes

W flits = 4096 / 32 = 128 flits
AW flits = 128 / 16 = 8 flits (每次 burst 16 beats)
Total request flits = 128 + 8 = 136 flits
```

---

## 6. 真正硬體並行 (V2)

真正硬體並行需 V2 架構:
- 多個 AXI Masters (多 DMA/CPU)
- 多個 NMUs (多入口點)
- Smart Crossbar (並行路由)

---

## 相關文件

- [系統概述](01_overview.md)
- [Flit 格式](02_flit.md)
