# Rectangle Coordinate-Range Multicast (RCR-Multicast)

## 硬體規格書

| 欄位 | 值 |
|------|----|
| 文件編號 | NOC-SPEC-RCR-001 |
| 版本 | 1.0 |
| 狀態 | Draft |
| 日期 | 2026-03-02 |
| 目標平台 | 2D Mesh NoC |

本文件中的 "MUST"、"SHALL"、"SHOULD"、"MAY" 依照 RFC 2119 解讀。

---

## 1 概述

RCR-Multicast 在 flit header 中以 `(x_min, x_max, y_min, y_max)` 編碼一個
軸對齊矩形，使單一 flit 傳送至矩形內所有節點。

**核心優勢**：與傳統 `(dst, mask)` 編碼使用相同的 bit 數，
但可表達任意矩形——不受 power-of-2 對齊限制。

### 1.1 與 (dst, mask) 的比較

`(dst, mask)` 的範圍為 `[dst & ~mask, dst | mask]`，寬度必為 power of 2
且對齊。`(min, max)` 無此限制。

**反例**：範圍 [1, 3]（寬度 3）——`(min, max)` 可直接表達；
`(dst, mask)` 無法在不多覆蓋或少覆蓋的情況下表達。

### 1.2 先前研究定位

| 方案 | Header Bits (16x16) | 可表達區域 | Router 硬體成本 |
|------|---------------------|-----------|----------------|
| Full Bitmask [1] | 256 | 任意子集 | N-bit demux |
| (dst, mask) [2] | 16 | Power-of-2 對齊矩形 | 4 comparators |
| SpiNNaker TCAM [3] | 8 | 任意（需 TCAM table） | TCAM lookup |
| **(min, max) RCR** | **16** | **任意矩形** | **8 comparators** |

---

## 2 Header 格式

### 2.1 Multicast Header

```systemverilog
typedef struct packed {
    logic [X_BITS-1:0] x_min;  // 矩形左邊界（含）
    logic [X_BITS-1:0] x_max;  // 矩形右邊界（含）
    logic [Y_BITS-1:0] y_min;  // 矩形下邊界（含）
    logic [Y_BITS-1:0] y_max;  // 矩形上邊界（含）
} multicast_hdr_t;             // 合計：2*X_BITS + 2*Y_BITS bits
```

此 struct 取代基礎 header 中的 `mask` 欄位。`src_x`、`src_y`、`commtype`、`last` 欄位不變。

### 2.2 退化情形

無需特殊硬體，以下模式自然成立：

| 模式 | 編碼方式 |
|------|---------|
| Unicast 至 (dx, dy) | x_min=x_max=dx, y_min=y_max=dy |
| 全網 Broadcast | (0, MAX_X, 0, MAX_Y) |
| 整行/整列 | 固定一軸，另一軸全展開 |

### 2.3 有效性約束

1. `x_min <= x_max <= MAX_X`
2. `y_min <= y_max <= MAX_Y`
3. Source SHOULD 位於矩形內或邊界上；若在外部，路徑仍正確但會經過額外節點。

---

## 3 路由演算法

### 3.1 核心邏輯

每個 router 根據自身座標 `(my_x, my_y)` 與 header 欄位，
以純組合邏輯計算 multi-hot 輸出埠選擇：

```systemverilog
wire x_in_range = (my_x >= x_min) && (my_x <= x_max);
wire y_in_range = (my_y >= y_min) && (my_y <= y_max);
wire on_src_row = (my_y == src_y);

assign route_sel[Eject] = x_in_range && y_in_range;

assign route_sel[East]  = on_src_row && (my_x >= src_x) && (my_x < x_max);
assign route_sel[West]  = on_src_row && (my_x <= src_x) && (my_x > x_min);
assign route_sel[North] = x_in_range && (my_y >= src_y) && (my_y < y_max);
assign route_sel[South] = x_in_range && (my_y <= src_y) && (my_y > y_min);
```

硬體成本：8 個 magnitude comparator + 1 個 equality comparator + 5 個 AND gate。
無 routing table、無序列邏輯。

### 3.2 兩階段傳播

**Phase 1 — X 軸擴散**：Flit 沿 source row 水平傳播至 `[x_min, x_max]`。
每個經過的 router 同時向 North/South 分叉（啟動 Phase 2）並在矩形內 eject。

**Phase 2 — Y 軸擴散**：從 Phase 1 覆蓋的每一列，flit 垂直傳播至
`[y_min, y_max]`，沿途 eject。

**關鍵不變量**：
- Router 不修改 flit header（所有 router 看到相同的 header）。
- Source NI 僅注入一個 flit。
- 嚴格 dimension ordering：進入 Y 維度後不回到 X。
- 矩形內每個節點恰好收到一份副本。

### 3.3 完整範例

8x8 mesh，Source = (0,1)，矩形 `[1,3] x [0,2]`（3x3 = 9 節點，寬度 3 非 power of 2）：

```
         x=0        x=1        x=2        x=3
        +----------+----------+----------+----------+
   y=2  |          | (1,2)  E | (2,2)  E | (3,2)  E |
        |          |    ^N    |    ^N    |    ^N    |
        +----------+----+-----+----+-----+----+-----+
   y=1  | (0,1)    | (1,1)  E | (2,1)  E | (3,1)  E |
        | SOURCE   |  E,N,S-->|  E,N,S-->|    N,S   |
        |    --E-->|    +     |    +     |    +     |
        +----------+----+-----+----+-----+----+-----+
   y=0  |          | (1,0)  E | (2,0)  E | (3,0)  E |
        |          |    vS    |    vS    |    vS    |
        +----------+----------+----------+----------+
  E = Eject
```

| Cycle | Router | 動作 |
|-------|--------|------|
| T0 | (0,1) | → East（source 在矩形外，不 eject） |
| T1 | (1,1) | Eject + East + North + South |
| T2 | (2,1), (1,2), (1,0) | 各自 eject；(2,1) 繼續分叉 |
| T3 | (3,1), (2,2), (2,0) | 各自 eject；(3,1) 繼續 N,S |
| T4 | (3,2), (3,0) | 各自 eject（終端） |

Source 注入 1 個 flit → 9 個節點各收到 1 份。複製分散於 source row 各 router。

---

## 4 Deadlock 分析

**Routing deadlock freedom**：路由函數強制 XY dimension ordering——
X 擴散先於 Y 擴散，各維度內單調傳播。滿足 Dally-Seitz 條件 [4]。

**Multicast tree deadlock**：多個並行 multicast 流 MAY 產生循環依賴。
SHOULD 以專用 VC (`EN_MCAST_VC`) 隔離 multicast 與 unicast 流量。

**Protocol-level deadlock**：透過 req/rsp 雙 channel 設計天然避免，無需額外機制。

---

## 5 Response 路徑

每個目標 NI 完成 AXI 交易後產生 response flit，以 unicast 路由回
`(src_x, src_y)`。Source NI 等待全部 N 個 response。

**可選硬體 reduction** (`EN_REDUCTION`)：Router 在收斂路徑上合併
response——任一 `RESP_SLVERR` 則合併結果為 `RESP_SLVERR`，
全部 `RESP_OKAY` 則為 `RESP_OKAY`。Source NI 僅收到 1 個合併 response。

---

## 6 可擴展性

| Mesh | 最大節點 | Header Bits | Router 額外面積 |
|------|---------|-------------|----------------|
| 4x4 | 16 | 8 | ~40 gates |
| 8x8 | 64 | 12 | ~64 gates |
| 16x16 | 256 | 16 | ~88 gates |
| 32x32 | 1024 | 20 | ~112 gates |

面積開銷相對於 router 的 FIFO buffer 和 crossbar（數千 gates）可忽略。

**延遲**：`T_worst = T_src_to_rect + (W-1) + (H-1)` hops。

**頻寬**：總 flit 傳輸次數 = `T_src_to_rect + W*H - 1`（最佳：
每個目標恰好 1 次傳送，僅 source 到矩形邊界的路徑為額外開銷）。

---

## 7 限制

- 每個封包僅能表達**軸對齊矩形**（L 形、對角線、稀疏子集不可）。
- 非矩形目的地需由軟體分解為多個矩形封包或 unicast。
- Source 在矩形外時，source row 上的額外 hop 消耗頻寬。
- 多 flit wormhole packet 期間會 hold 多個輸出埠，MAY 造成 head-of-line blocking。

---

## 8 設計參數

| 參數 | 預設值 | 說明 |
|------|-------|------|
| `EN_MULTICAST` | 0 | 啟用 multicast。未設定時邏輯全部被 synthesis 移除。 |
| `X_BITS` / `Y_BITS` | 4 | 座標位寬，MUST 與 `id_t` 一致。 |
| `EN_MCAST_VC` | 0 | 專用 multicast VC（需 `NumVirtChannels >= 2`）。 |
| `EN_REDUCTION` | 0 | 網路內 response reduction（需 `EN_MULTICAST = 1`）。 |

---

## 9 Router 整合

**Header 映射**：`hdr.dst_id` 欄位重新解讀為 `multicast_hdr_t`（`{x_min, x_max}` + `{y_min, y_max}`）。
`hdr.src_id`、`hdr.commtype` 不變。

**Router 整合**：以 `route_rcr` 取代或並行於 `route_xy`，
介面相容（`channel_i`, `xy_id_i` → `route_sel_o`）。

**NI 整合**：Source NI 從 AXI `user` 欄位或 SAM lookup 填充矩形座標。
具體映射方式為 IMPLEMENTATION DEFINED。

---

## 參考文獻

[1] L. Wang et al., "Recursive Partitioning Multicast," *Proc. IEEE NOCS*, 2009.
[2] A. Madhavan et al., "mppSoC," *Proc. IEEE ASAP*, 2012.
[3] S. B. Furber et al., "The SpiNNaker Project," *Proc. IEEE*, vol. 102, no. 5, 2014.
[4] W. J. Dally and C. L. Seitz, "Deadlock-Free Message Routing," *IEEE Trans. Computers*, vol. C-36, no. 5, 1987.
