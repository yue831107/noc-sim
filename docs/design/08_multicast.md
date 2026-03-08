# Multicast

本文件定義 RCR-Multicast（Rectangle Coordinate-Range Multicast）的 header 格式、路由行為、response 路徑與設計參數。

---

## 1. Overview

RCR-Multicast 在 flit header 中以 `(x_min, x_max, y_min, y_max)` 編碼一個
軸對齊矩形，使單一 flit 傳送至矩形內所有節點。

**核心優勢**：與傳統 `(dst, mask)` 編碼使用相同的 bit 數，
但可表達任意矩形——不受 power-of-2 對齊限制。

### 1.1. 與 (dst, mask) 的比較

`(dst, mask)` 的範圍為 `[dst & ~mask, dst | mask]`，寬度必為 power of 2
且對齊。`(min, max)` 無此限制。

**反例**：範圍 [1, 3]（寬度 3）——`(min, max)` 可直接表達；
`(dst, mask)` 無法在不多覆蓋或少覆蓋的情況下表達。

### 1.2. 方案比較

| 方案 | Header Bits (16x16) | 可表達區域 | 複雜度 |
|------|---------------------|-----------|--------|
| Full Bitmask | 256 | 任意子集 | 高 |
| (dst, mask) | 16 | Power-of-2 對齊矩形 | 低 |
| TCAM-based | 8 | 任意（需 lookup table） | 高 |
| **(min, max) RCR** | **16** | **任意矩形** | **低** |

---

## 2. Header 格式

### 2.1. Multicast Header

Multicast header 以 `(x_min, x_max, y_min, y_max)` 四欄位取代基礎 header 的 `dst_id` 欄位：

| Field | Width | Description |
|-------|-------|-------------|
| `x_min` | X_BITS | 矩形左邊界（含） |
| `x_max` | X_BITS | 矩形右邊界（含） |
| `y_min` | Y_BITS | 矩形下邊界（含） |
| `y_max` | Y_BITS | 矩形上邊界（含） |

合計：2×X_BITS + 2×Y_BITS bits。`src_x`、`src_y`、`commtype`、`last` 欄位不變。

### 2.2. 退化情形

無需特殊硬體，以下模式自然成立：

| 模式 | 編碼方式 |
|------|---------|
| Unicast 至 (dx, dy) | x_min=x_max=dx, y_min=y_max=dy |
| 全網 Broadcast | (0, MAX_X, 0, MAX_Y) |
| 整行/整列 | 固定一軸，另一軸全展開 |

### 2.3. 有效性約束

1. `x_min <= x_max <= MAX_X`
2. `y_min <= y_max <= MAX_Y`
3. Source SHOULD 位於矩形內或邊界上；若在外部，路徑仍正確但會經過額外節點。

---

## 3. 路由演算法

### 3.1. 核心邏輯

每個 router 根據自身座標 `(my_x, my_y)` 與 header 欄位，計算 multi-hot 輸出埠選擇：

```
x_in_range = (my_x >= x_min) AND (my_x <= x_max)
y_in_range = (my_y >= y_min) AND (my_y <= y_max)
on_src_row = (my_y == src_y)

route[Eject] = x_in_range AND y_in_range
route[East]  = on_src_row AND (my_x >= src_x) AND (my_x < x_max)
route[West]  = on_src_row AND (my_x <= src_x) AND (my_x > x_min)
route[North] = x_in_range AND (my_y >= src_y) AND (my_y < y_max)
route[South] = x_in_range AND (my_y <= src_y) AND (my_y > y_min)
```

無 routing table、無狀態。每個 router 獨立判斷。

### 3.2. 兩階段傳播

**Phase 1 — X 軸擴散**：Flit 沿 source row 水平傳播至 `[x_min, x_max]`。
每個經過的 router 同時向 North/South 分叉（啟動 Phase 2）並在矩形內 eject。

**Phase 2 — Y 軸擴散**：從 Phase 1 覆蓋的每一列，flit 垂直傳播至
`[y_min, y_max]`，沿途 eject。

**關鍵不變量**：
- Router 不修改 flit header（所有 router 看到相同的 header）。
- Source NI 僅注入一個 flit。
- 嚴格 dimension ordering：進入 Y 維度後不回到 X。
- 矩形內每個節點恰好收到一份副本。

### 3.3. 完整範例

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

## 4. Deadlock 分析

**Routing deadlock freedom**：路由函數強制 XY dimension ordering — X 擴散先於 Y 擴散，各維度內單調傳播。

**Multicast tree deadlock**：多個並行 multicast 流可能產生循環依賴。建議以專用 VC (`EN_MCAST_VC`) 隔離 multicast 與 unicast 流量。

**Protocol-level deadlock**：透過 req/rsp 雙 channel 設計天然避免，無需額外機制。

---

## 5. Response 路徑

每個目標 NI 完成 AXI 交易後產生 response flit，以 unicast 路由回 `(src_x, src_y)`。Source NI 等待全部 N 個 response。

**可選 reduction** (`EN_REDUCTION`)：Router 在收斂路徑上合併 response — 任一 `RESP_SLVERR` 則合併結果為 `RESP_SLVERR`，全部 `RESP_OKAY` 則為 `RESP_OKAY`。Source NI 僅收到 1 個合併 response。

---

## 6. 可擴展性

| Mesh | 最大節點 | Header Bits |
|------|---------|-------------|
| 4×4 | 16 | 8 |
| 8×8 | 64 | 12 |
| 16×16 | 256 | 16 |

**延遲**：`T_worst = T_src_to_rect + (W-1) + (H-1)` hops。

**頻寬**：總 flit 傳輸次數 = `T_src_to_rect + W×H - 1`（每個目標恰好 1 次傳送，僅 source 到矩形邊界的路徑為額外開銷）。

---

## 7. 限制

- 每個封包僅能表達**軸對齊矩形**（L 形、對角線、稀疏子集不可）。
- 非矩形目的地需由軟體分解為多個矩形封包或 unicast。
- Source 在矩形外時，source row 上的額外 hop 消耗頻寬。
- 多 flit wormhole packet 期間會 hold 多個輸出埠，可能造成 head-of-line blocking。

---

## 8. 設計參數

| 參數 | 預設值 | 說明 |
|------|-------|------|
| `EN_MULTICAST` | 0 | 啟用 multicast。未設定時相關邏輯不啟用。 |
| `X_BITS` / `Y_BITS` | 4 | 座標位寬，須與 `id_t` 一致。 |
| `EN_MCAST_VC` | 0 | 專用 multicast VC（需 `NumVirtChannels >= 2`）。 |
| `EN_REDUCTION` | 0 | 網路內 response reduction（需 `EN_MULTICAST = 1`）。 |

---

## 9. Router 整合

**Header 映射**：`hdr.dst_id` 欄位重新解讀為 `multicast_hdr_t`（`{x_min, x_max}` + `{y_min, y_max}`）。
`hdr.src_id`、`hdr.commtype` 不變。

**Router 整合**：以 `route_rcr` 取代或並行於 `route_xy`，
介面相容（`channel_i`, `xy_id_i` → `route_sel_o`）。

**NI 整合**：Source NI 從 AXI `user` 欄位或 SAM lookup 填充矩形座標。
具體映射方式為 IMPLEMENTATION DEFINED。

---

## Related Documents

- [Flit Format](02_flit.md) — Header 欄位定義（`dst_id` 重新解讀為 multicast 座標）
- [Router](03_router.md) — Router pipeline 與 routing 整合
- [Network Interface](04_network_interface.md) — NI 的 multicast response 處理
