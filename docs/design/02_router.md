# Router 規格

NoC Mesh 內的封包轉發單元。每個 Router 負責接收、路由決策、仲裁、crossbar switching 與轉發 flit。本文件定義 Router 的完整行為，包含 pipeline timing、state machine、仲裁策略，可直接用於 C++ cycle-accurate 實作。

---

## 1. Overview

### 1.1 Router 在 Mesh 中的角色

```
           N (to y+1)
           |
   W ---- [Router] ---- E
  (x-1)   |    |       (x+1)
           S    L
         (y-1) (Local NI)
```

每個 Router 為邏輯單元，內部由兩個獨立子 Router 組成 — **ReqRouter** 處理 Request network（AW/W/AR），**RspRouter** 處理 Response network（B/R）。兩者結構對稱，共用相同 routing / arbitration / crossbar 邏輯，僅處理的 flit 類型不同。

### 1.2 Router Port 架構

每個 Router 具備 **4 個 mesh 方向 port**（N, S, E, W）與 **0~4 個 LOCAL port**（由 `port_id` 識別）：

| Port 類別 | Port | 連接 | 識別方式 |
|-----------|------|------|---------|
| Mesh | N, S, E, W | 相鄰 Router | Direction |
| Local | L0, L1, L2, L3 | NI（各類周邊） | `port_id` [1:0] |

LOCAL port 數量為配置參數 `NUM_LOCAL_PORTS`（預設 1），每個 LOCAL port 獨立擁有 input buffer、output buffer、credit counter。Mesh 邊界 Router 不存在的方向 port 不連接。

Multi-port LOCAL 架構詳見 Section 11。

---

## 2. Design Parameters

所有參數與 [Flit Format](04_flit.md) 一致，採固定值設計。

| Parameter | Value | Source | Description |
|-----------|-------|--------|-------------|
| `FLIT_WIDTH` | 408 | 04_flit.md | Flit 寬度 (bits) |
| `HEADER_WIDTH` | 56 | 04_flit.md | Header 寬度 (bits) |
| `PAYLOAD_WIDTH` | 352 | 04_flit.md | Payload 寬度 (bits) |
| `NODE_ID_WIDTH` | 8 | 04_flit.md | Node ID 寬度 (X_WIDTH + Y_WIDTH) |
| `X_WIDTH` | 4 | 04_flit.md | X coordinate width (max 16 columns) |
| `Y_WIDTH` | 4 | 04_flit.md | Y coordinate width (max 16 rows) |
| `QOS_WIDTH` | 4 | 04_flit.md | QoS priority width (16 levels) |
| `LINK_WIDTH` | 410 | 04_flit.md | Physical link width (valid + ready + flit) |
| `INPUT_BUFFER_DEPTH` | 4 | Router config | Input buffer 深度 (flits) |
| `OUTPUT_BUFFER_DEPTH` | 2 | Router config | Output buffer 深度 (flits, 0 = bypass) |
| `NUM_MESH_PORTS` | 4 | Router config | Mesh direction ports (N/S/E/W) |
| `NUM_LOCAL_PORTS` | 1 | Router config | LOCAL ports per Router (0~4, port_id 識別) |
| `MAX_LOCAL_PORTS` | 4 | 04_flit.md | port_id 2 bits → 最多 4 個 LOCAL ports |

---

## 3. Port Architecture

### 3.1 Port 定義

| Port | Direction | Connection | dst_id Condition |
|------|-----------|------------|------------------|
| N (North) | 北 | (x, y+1) | dst_y > cur_y |
| S (South) | 南 | (x, y-1) | dst_y < cur_y |
| E (East) | 東 | (x+1, y) | dst_x > cur_x |
| W (West) | 西 | (x-1, y) | dst_x < cur_x |
| L (Local) | 本地 | NI | dst_x == cur_x && dst_y == cur_y |

Mesh 邊界的 Router，不存在的方向之 port 不連接（例如 y=0 的 Router 無 South neighbor）。

### 3.2 Internal Port Structure

Router 內部每個 port 的信號結構（CppRouter 實作細節，外部合約為 [Router_Interface](05_internal_interface.md) Section 3）：

```cpp
struct RouterPort {
    // --- Internal State ---
    FlitBuffer   input_buffer;      // depth = INPUT_BUFFER_DEPTH
    FlitBuffer   output_buffer;     // depth = OUTPUT_BUFFER_DEPTH (optional)
    CreditFlowControl output_credit; // tracks downstream buffer space

    // --- Ingress Interface (receiving from upstream) ---
    bool         in_valid;    // set by upstream
    Flit         in_flit;     // set by upstream (408 bits)
    bool         out_ready;   // set by this port (= input_buffer has space)

    // --- Egress Interface (sending to downstream) ---
    bool         out_valid;   // set by this port
    Flit         out_flit;    // set by this port (408 bits)
    bool         in_ready;    // set by downstream

    // --- Methods ---
    void update_ready();              // out_ready = !input_buffer.full()
    bool sample_input();              // if (in_valid && out_ready) → push to buffer
    bool set_output(Flit flit);       // push to output_buffer or set out_valid/out_flit
    bool clear_output_if_accepted();  // if (out_valid && in_ready) → pop & clear
};
```

### 3.3 Physical Link

每條 physical link 寬度 410 bits（Version A）或 409 + NUM_VC bits（Version B）：

```
  Version A: 410-bit Physical Link (per VC, per direction, per network):
  ┌───────┬───────┬──────────────────────────────┐
  │ valid │ ready │        flit (408 bits)        │
  │  1b   │  1b   │   header(56) + payload(352)   │
  └───────┴───────┴──────────────────────────────┘

  Version B: Shared forward + per-VC credit return:
  ┌───────┬──────────────────────────────┐  ┌──────────────────┐
  │ valid │        flit (408 bits)        │  │ credit[NUM_VC]   │
  │  1b   │   header(56) + payload(352)   │  │ (per-VC pulse)   │
  └───────┴──────────────────────────────┘  └──────────────────┘
```

Router 間的連接透過 abstract interface 的 `wire_all()` 實現（見 [Internal Interface](05_internal_interface.md) Section 3）。Simulation Driver 每 cycle 讀取各 Router 的 `get_output()` 並寫入對端的 `set_input()`，等效 zero-cycle wire propagation。

每個 Router-to-Router link 包含 Req + Rsp 兩組 channel，每組雙向。因此兩個相鄰 Router 間共有 4 條 physical link。

---

## 4. XY Routing Algorithm

### 4.1 基本規則

XY Routing 為確定性路由演算法，保證 deadlock-free（在無 VC 的 wormhole network 中）：

1. 先沿 X 軸移動（EAST / WEST）
2. 再沿 Y 軸移動（NORTH / SOUTH）
3. 到達目標座標後轉送至 LOCAL

### 4.2 Y-to-X 轉向禁止 (Deadlock Prevention)

一旦封包進入 Y 軸方向，禁止轉回 X 軸。這確保路由路徑形成 partial order，消除 cyclic dependency：

```
禁止的轉向 (Y→X):
  NORTH → EAST  ✗
  NORTH → WEST  ✗
  SOUTH → EAST  ✗
  SOUTH → WEST  ✗

允許的轉向:
  任何 → NORTH/SOUTH  (進入 Y 軸)
  EAST/WEST → 任何     (仍在 X 軸或轉入 Y 軸)
  LOCAL → 任何         (注入)
```

### 4.3 Routing Function

Router 從 flit header 的 `dst_id[7:0]` 提取目標座標：`dst_x = dst_id[3:0]`, `dst_y = dst_id[7:4]`。

```cpp
// Returns output port direction for a given flit
Direction compute_output_port(const Flit& flit, Direction input_dir) {
    uint8_t dst_x = flit.header.dst_id & 0x0F;        // dst_id[3:0]
    uint8_t dst_y = (flit.header.dst_id >> 4) & 0x0F;  // dst_id[7:4]

    Direction desired;

    // Step 1: X-first routing
    if (dst_x > cur_x)
        desired = EAST;
    else if (dst_x < cur_x)
        desired = WEST;
    // Step 2: then Y routing
    else if (dst_y > cur_y)
        desired = NORTH;
    else if (dst_y < cur_y)
        desired = SOUTH;
    // Step 3: arrived
    else
        return LOCAL;

    // Y→X turn prohibition check (safety assertion)
    if (input_dir == NORTH || input_dir == SOUTH) {
        assert(desired != EAST && desired != WEST);
        // XY routing guarantees this never happens;
        // if triggered, indicates corrupted dst_id
    }

    return desired;
}
```

> **Note:** XY routing 的數學特性天然保證不會出現 Y→X 轉向（X 方向偏差必在 Y 移動前歸零）。Y→X 檢查僅為 assertion / debug aid。

---

## 5. Pipeline Architecture

### 5.1 8-Phase Cycle

與 [Internal Interface](05_internal_interface.md) Section 7 一致，每個 cycle 分為 8 個 phase，所有 Router 在同一 phase 內同步執行：

| Phase | Name | Operation | Description |
|-------|------|-----------|-------------|
| 1 | Sample | `sample inputs` | 取樣上一 cycle wire_all() 設定的 input 信號，寫入 input buffer |
| 2 | Clear Inputs | `clear input signals` | 清除 input 信號（避免重複取樣） |
| 3 | Update Ready | `update ready/credit` | 根據 input buffer 空間更新 output.ready (VR) 或 output.credit (Credit) |
| 4 | Route & Forward | `route & forward` | Routing 決策 + QoS-Aware Arbitration + Crossbar switching |
| 5 | Wire All | `wire_all()` | 透過 abstract interface 交換所有元件的 output → 對端 input |
| 6 | Clear Accepted | `clear accepted` | 根據 input 中對端的 ready/credit 確認 handshake，清除 output |
| 7 | Credit Release | `credit release` | 更新 credit counter（Version B only） |
| 8 | NI Process | `ni.tick()` | NI 處理 AXI ↔ flit conversion |

### 5.2 Pipeline Timing Diagram

```
Cycle N-1                              Cycle N                              Cycle N+1
──────────────                         ──────────────────────────────────   ──────────
  Phase 5: wire_all() ────────────┐
                                 │
                                 ▼
                          Phase 1: sample inputs
                            取樣上一 cycle wire_all() 設定的 input
                          Phase 2: clear input signals
                          Phase 3: update ready/credit
                            out_ready = !input_buffer.full()
                          Phase 4: route_and_forward()         ← 核心邏輯
                            4a. 從 input buffer peek head flit
                            4b. compute_output_port() 計算輸出方向
                            4c. QoS-Aware arbitration（HEAD flit）
                            4d. Crossbar switch（winner → output）
                            4e. Wormhole path lock / unlock
                          Phase 5: wire_all() ───────────────────┐
                          Phase 6: clear_accepted_outputs()      │
                          Phase 7: credit_release()              │
                          Phase 8: NI process                    ▼
                                                           Phase 1 (N+1)...
```

### 5.3 Phase 4 Detail: Route & Forward

Phase 4 為 Router 的核心邏輯，包含以下子步驟（在同一 phase 內順序執行）：

```cpp
void route_and_forward(int current_time) {
    // 4a. Collect requests: peek head flit from each input buffer
    //     Routing function 回傳 output port set（unicast=1 個，multicast=多個）
    struct RouteRequest {
        Direction in_dir;
        std::set<Direction> out_dirs;  // multi-hot for multicast
        Flit flit;
    };

    std::map<Direction, RouteRequest> requests;
    for (auto& [dir, port] : ports) {
        if (!port.input_buffer.empty()) {
            Flit flit = port.input_buffer.peek();
            std::set<Direction> out_dirs = compute_output_ports(flit, dir);
            requests[dir] = { dir, out_dirs, flit };
        }
    }

    // 4b. Separate locked paths and new requests
    std::vector<RouteRequest> locked_grants;
    std::vector<RouteRequest> new_requests;
    for (auto& [in_dir, req] : requests) {
        if (path_locks[in_dir].is_locked()) {
            // Wormhole: body/tail follows locked path(s), no arbitration
            req.out_dirs = path_locks[in_dir].get_outputs();  // may be multi-hot
            locked_grants.push_back(req);
        } else if (is_head_flit(req.flit)) {
            new_requests.push_back(req);
        }
    }

    // 4c. Process locked paths (guaranteed, no arbitration)
    //     Multicast: independent per-port handshake (Section 6.6)
    for (auto& grant : locked_grants) {
        auto& tracker = mc_trackers[grant.in_dir];
        for (Direction out : grant.out_dirs) {
            if (tracker.is_completed(out)) continue;  // 已完成，跳過
            if (can_switch_to_output(out)) {
                crossbar_switch(grant.in_dir, out, grant.flit);
                tracker.current_handshakes.insert(out);
            }
        }
        // 所有 target output 都完成 → pop input buffer
        if (tracker.all_done()) {
            ports[grant.in_dir].input_buffer.pop();
            if (is_tail_flit(grant.flit)) {
                path_locks[grant.in_dir].unlock();  // last=1 releases all locks
            }
            tracker.commit_and_reset();  // past_handshakes 歸零
        } else {
            tracker.commit();  // current → past，下 cycle 繼續
        }
    }

    // 4d. QoS-Aware Round-Robin arbitration for new HEAD flits
    auto winners = qos_aware_arbitrate(new_requests, current_time);

    // 4e. Process arbitration winners (independent per-port handshake)
    for (auto& grant : winners) {
        auto& tracker = mc_trackers[grant.in_dir];
        tracker.expected = grant.out_dirs;

        for (Direction out : grant.out_dirs) {
            if (can_switch_to_output(out) && is_output_available(out)) {
                crossbar_switch(grant.in_dir, out, grant.flit);
                tracker.current_handshakes.insert(out);
            }
        }

        if (tracker.all_done()) {
            // 所有 target output 同 cycle 完成（常見於 unicast 或低負載）
            ports[grant.in_dir].input_buffer.pop();
            if (!is_tail_flit(grant.flit)) {
                path_locks[grant.in_dir].lock(grant.out_dirs);
            }
            // Single-flit (last=1): 不建立 lock
            tracker.commit_and_reset();
        } else if (!tracker.current_handshakes.empty()) {
            // 部分完成 → 建立 wormhole lock，下 cycle 繼續其餘 output
            path_locks[grant.in_dir].lock(grant.out_dirs);
            tracker.commit();  // current → past
        }
        // else: 全部 output 不可用 → HEAD stays in buffer, retry next cycle
    }
}

// Routing function: returns multi-hot output port set
// Unicast: returns single-element set
// Multicast: returns multiple outputs per RCR routing algorithm
std::set<Direction> compute_output_ports(const Flit& flit, Direction input_dir) {
    if (flit.header.commtype == 0) {
        // Unicast: delegate to standard XY routing
        return { compute_output_port(flit, input_dir) };
    }
    // Multicast (commtype=1): coordinate-range routing
    // See rectangle_multicast_spec_zh-TW.md Section 3.1
    return compute_rcr_outputs(flit, input_dir);
}
```

> **Independent per-port handshake**: Multicast flit 不要求所有 target output 同時可用。已完成 handshake 的 output 立即釋放（透過 `is_completed()` 壓掉 valid），未完成的 output 下 cycle 繼續嘗試。Input buffer 在 `all_done()` 時才 pop。詳見 Section 6.6。

---

## 6. Wormhole Switching

### 6.1 概念

Wormhole switching 確保封包完整性：

- **HEAD flit**（packet 首個 flit）需經過仲裁，獲勝後鎖定 input→output 路徑
- **BODY flit** 沿鎖定路徑直接轉發，無需重新仲裁
- **TAIL flit**（header.last=1）轉發後釋放路徑鎖定

### 6.2 Flit 類型判定

利用 header 的 `last` 欄位（bit [25]）與 path lock 狀態判定 flit 類型：

| Condition | Flit Type | Behavior |
|-----------|-----------|----------|
| 無 lock 且 last=0 | HEAD | 需仲裁，獲勝後建立 path lock |
| 無 lock 且 last=1 | HEAD+TAIL (single-flit) | 仲裁後立即轉發並釋放（AW/AR/B） |
| 有 lock 且 last=0 | BODY | 直接轉發，維持 lock |
| 有 lock 且 last=1 | TAIL | 轉發後釋放 lock |

> **Single-flit packets:** AW、AR、B channel 的 packet 均為單一 flit（恆為 last=1）。W 與 R channel 為 multi-flit packet（awlen+1 / arlen+1 flits），僅末尾 flit 設 last=1。

### 6.3 Path Lock State Machine

每個 input port 維護一個獨立的 path lock FSM：

```
                    ┌──────────────────────────────────────┐
                    │                                      │
              HEAD wins arbitration                  TAIL flit transmitted
              (output not locked by others)          (last=1 && handshake OK)
                    │                                      │
                    ▼                                      │
    ┌───────────┐         ┌───────────┐         ┌─────────┴──┐
    │   IDLE    │────────►│  LOCKED   │────────►│  RELEASE   │
    │(可參與    │  HEAD   │ (body/tail│  TAIL   │ (釋放 lock,│
    │ 仲裁)    │  wins   │  直接通過)│  sent   │  回到 IDLE)│
    └───────────┘         └─────┬─────┘         └────────────┘
         ▲                      │                      │
         │                      │ BODY flit            │
         │                      └──────┘               │
         │                 (stay LOCKED)               │
         └─────────────────────────────────────────────┘
                         (immediate transition)
```

**C++ State Machine:**

```cpp
enum class PathLockState { IDLE, LOCKED };

struct PathLock {
    PathLockState          state = PathLockState::IDLE;
    std::set<Direction>    locked_outputs;   // 支援 multicast 多 output

    bool is_locked() const { return state == PathLockState::LOCKED; }

    // Unicast lock: 單一 output
    void lock(Direction out_dir) {
        assert(state == PathLockState::IDLE);
        state = PathLockState::LOCKED;
        locked_outputs = { out_dir };
    }

    // Multicast lock: 多個 output（synchronized multicast）
    void lock(const std::set<Direction>& out_dirs) {
        assert(state == PathLockState::IDLE);
        assert(!out_dirs.empty());
        state = PathLockState::LOCKED;
        locked_outputs = out_dirs;
    }

    void unlock() {
        assert(state == PathLockState::LOCKED);
        state = PathLockState::IDLE;
        locked_outputs.clear();
    }

    // 取得已鎖定的 output set（Phase 4 使用）
    const std::set<Direction>& get_outputs() const {
        return locked_outputs;
    }

    // 檢查某 output 是否被此 lock 佔用
    bool holds_output(Direction out_dir) const {
        return locked_outputs.count(out_dir) > 0;
    }
};
```

> **Unicast vs Multicast Lock**: Unicast packet 鎖定單一 output（`locked_outputs` 大小為 1）。Multicast packet 鎖定多個 output（大小 ≥ 2）。兩者共用相同的 FSM 狀態轉移，差異僅在 `locked_outputs` 集合大小。BODY/TAIL flit 沿用 HEAD 建立的完整 output set。

### 6.4 Output Port Contention

一個 output port 在同一 cycle 只能被一個 input port 使用（crossbar 限制）。已被 wormhole lock 佔用的 output port 不可被其他 HEAD flit 分配：

```cpp
bool is_output_available(Direction out_dir) const {
    // Check: no existing path lock targets this output
    for (auto& [in_dir, lock] : path_locks) {
        if (lock.is_locked() && lock.holds_output(out_dir))
            return false;
    }
    return true;
}
```

> **Multicast 影響**: 一個 multicast wormhole lock 可能同時佔用多個 output port。例如 input WEST 鎖定 {NORTH, EAST, LOCAL}，則這三個 output 對其他 HEAD flit 均不可用。

### 6.6 Multicast Handshake: Independent Per-Port Completion

Multicast flit 的多 output 轉發採用 **independent per-port handshake** 機制：各 target output port 可在不同 cycle 獨立完成 handshake，已完成的 output port 立即釋放給其他 traffic，input buffer 在**所有** target output 都完成後才 pop。

#### 6.6.1 Past Handshake Tracking

每個 input port 維護一組 `past_handshakes` register，記錄當前 flit 已完成 handshake 的 output port 集合：

```cpp
struct MulticastTracker {
    // per input port, per output direction: 此 output 是否已完成 handshake
    std::set<Direction> past_handshakes;

    // 當前 cycle 完成的 handshake（output arbiter grant + ready）
    std::set<Direction> current_handshakes;

    // 本 flit 預期的 target output 集合（由 routing function 決定）
    std::set<Direction> expected;

    // 已完成 handshake 的 output → 壓掉 valid，釋放該 output arbiter
    bool is_completed(Direction out_dir) const {
        return past_handshakes.count(out_dir) > 0;
    }

    // 所有 target output 都已完成 → 可以 pop input buffer
    bool all_done() const {
        for (Direction d : expected) {
            if (past_handshakes.count(d) == 0 &&
                current_handshakes.count(d) == 0)
                return false;
        }
        return true;
    }

    // Cycle 結束：current_handshakes 併入 past_handshakes（下 cycle 繼續）
    void commit() {
        past_handshakes.insert(current_handshakes.begin(),
                               current_handshakes.end());
        current_handshakes.clear();
    }

    // 一個 flit 全部完成：commit 後 reset（準備下一個 flit）
    void commit_and_reset() {
        past_handshakes.clear();
        current_handshakes.clear();
        expected.clear();
    }
};
```

#### 6.6.2 Handshake 流程

**Valid 信號控制**: Multicast flit 的 valid 對每個 target output 獨立控制。已完成 handshake 的 output 被壓掉 valid（`~past_handshakes`），使該 output arbiter 立即可服務其他 input：

```cpp
// 對每個 output port，計算 masked_valid
bool get_masked_valid(Direction in_dir, Direction out_dir) const {
    const auto& tracker = mc_trackers[in_dir];
    bool route_hit = tracker.expected.count(out_dir) > 0;
    bool already_done = tracker.is_completed(out_dir);
    bool in_valid = !ports[in_dir].input_buffer.empty();

    return in_valid && route_hit && !already_done;
}
```

**Ready 信號控制**: Input port 的 ready（可 pop buffer）僅在所有 expected output 都完成 handshake 時才 assert：

```cpp
// Input buffer pop 條件
bool can_pop_input(Direction in_dir) const {
    return mc_trackers[in_dir].all_done();
}
```

#### 6.6.3 時序範例

```
Multicast flit 從 WEST input 進入，target outputs = {NORTH, EAST, LOCAL}

Cycle 1: EAST arbiter grant ✓（空閒）
         NORTH arbiter busy ✗（被其他 wormhole lock 佔用）
         LOCAL arbiter busy ✗
         → past_handshakes = {EAST}
         → EAST output 立即釋放，可服務其他 traffic
         → input buffer 不 pop（未全部完成）

Cycle 2: NORTH arbiter grant ✓（前一筆 TAIL 到達，lock 釋放）
         LOCAL arbiter busy ✗
         → past_handshakes = {EAST, NORTH}
         → NORTH output 釋放

Cycle 3: LOCAL arbiter grant ✓
         → past_handshakes = {EAST, NORTH, LOCAL} == expected
         → all_done() = true → input buffer pop
         → past_handshakes reset → 準備下一個 flit
```

#### 6.6.4 Multi-Flit Wormhole Packet

對 multi-flit multicast packet（W burst, R burst），每個 flit 獨立執行 per-port handshake：

```
W burst (awlen=2, 3 flits): target = {NORTH, EAST}

HEAD flit:
  Cycle 1: EAST accept ✓, NORTH busy → past = {EAST}
  Cycle 2: NORTH accept ✓ → past = {EAST, NORTH} → all_done → pop
           → 各 output arbiter 建立 wormhole lock (LockIn)

BODY flit:
  Cycle 3: EAST accept ✓, NORTH accept ✓ → all_done → pop
           （兩個 output 都已被 wormhole lock 鎖定到此 input，
             通常同 cycle 就能全部 accept）

TAIL flit (last=1):
  Cycle 4: EAST accept ✓, NORTH accept ✓ → all_done → pop
           → 各 output arbiter 釋放 wormhole lock
```

> **HEAD 後的 BODY/TAIL**: HEAD 完成 handshake 後，各 target output arbiter 已建立 wormhole lock（LockIn），鎖定到此 input。後續 BODY/TAIL flit 通常能在同一 cycle 被所有 target output 接受，因為 wormhole lock 保證了 output 的獨佔性。

#### 6.6.5 Unicast 退化

Unicast packet（`expected` 大小為 1）時，per-port handshake 退化為普通的 single-output handshake：output accept 即 `all_done()`，行為與無 multicast 支援的 router 完全一致。

#### 6.6.6 設計特性

| 特性 | 說明 |
|------|------|
| **Output 早期釋放** | 已完成 handshake 的 output 立即可服務其他 traffic，不等其餘 output |
| **無 collateral blocking** | 不 de-assert 任何 input port 的 ready，不影響無關 traffic |
| **Arbiter 無感知** | Output arbiter 為標準 wormhole arbiter，不需 multicast 特殊邏輯 |
| **Input HoL blocking** | Flit 在 input buffer 中停留直到所有 output 完成，此期間該 input port blocked |
| **Liveness 保證** | 每個 target output 的 wormhole lock 有限（TAIL 必到），故所有 output 必定完成 |

### 6.5 AXI Channel → Wormhole Packet 邊界定義

AXI 的 5 個 channel（AW/W/AR/B/R）映射為**獨立的 wormhole packet**。Router 不需要維護 AW 與 W 之間的關聯邏輯 — ordering 完全由 NMU 保證。

#### 6.5.1 Flit Type 對照表

| AXI Channel | Network | Flit Count | Flit Types | Wormhole Behavior |
|-------------|---------|-----------|------------|-------------------|
| AW | REQ | 1 | HEAD_ONLY (last=1) | 單 flit，不鎖路 |
| W (awlen+1 beats) | REQ | awlen+1 | HEAD (last=0) + (awlen-1)×BODY (last=0) + TAIL (last=1) | 鎖路直到 TAIL |
| AR | REQ | 1 | HEAD_ONLY (last=1) | 單 flit，不鎖路 |
| B | RSP | 1 | HEAD_ONLY (last=1) | 單 flit，不鎖路 |
| R (arlen+1 beats) | RSP | arlen+1 | HEAD (last=0) + (arlen-1)×BODY (last=0) + TAIL (last=1) | 鎖路直到 TAIL |

> **Single-beat W/R（awlen=0 / arlen=0）：** 產生 1 個 flit，last=1，行為等同 HEAD_ONLY（單 flit，不鎖路）。

#### 6.5.2 AW 與 W 的獨立性

AW 與 W 為**兩個獨立的 wormhole packet**，關鍵規則：

1. **AW flit** 恆為 single-flit packet（last=1），經 arbitration 後立即轉發並釋放路徑，不建立 wormhole lock
2. **W burst** 為獨立的 multi-flit packet（HEAD + BODY×(N-2) + TAIL），建立獨立的 wormhole lock
3. **AW→W ordering** 由 NMU 保證：NMU 先注入 AW flit，再注入 W burst 的所有 flit。由於兩者從同一 NMU 的同一 output port 注入 Req Router，FIFO 順序天然保證 AW 先於 W 到達 NSU
4. **Router 無需 AW/W 關聯邏輯**：Router 視每個 flit 為獨立 packet 或已鎖定 path 的一部分，不感知 AXI channel 語義

#### 6.5.3 Wormhole Lock 生命週期範例

```
Write Transaction (awlen=3, 4 beats):

  AW flit:  [HEAD_ONLY, last=1] → arbitrate → forward → no lock (immediate release)
  W flit 0: [HEAD,      last=0] → arbitrate → forward → LOCK path
  W flit 1: [BODY,      last=0] → locked path → forward → maintain lock
  W flit 2: [BODY,      last=0] → locked path → forward → maintain lock
  W flit 3: [TAIL,      last=1] → locked path → forward → UNLOCK path

Single Write (awlen=0, 1 beat):

  AW flit:  [HEAD_ONLY, last=1] → arbitrate → forward → no lock
  W flit 0: [HEAD_ONLY, last=1] → arbitrate → forward → no lock
```

---

## 7. QoS-Aware Round-Robin Arbitration

仲裁策略與 [QoS Design](10_qos.md) Section 5 一致：**qos 為第一優先級，同 qos 才使用 Round-Robin**。

### 7.1 Arbitration Policy

```
Priority order:
  1. Higher qos value wins (qos=15 highest, qos=0 lowest)
  2. Same qos: Round-Robin among same-priority HEAD flits
  3. Wormhole: Body/Tail flits follow locked path (no preemption)
```

### 7.2 Per-Output Arbitration

仲裁以 **output port** 為單位進行。若多個 input port 的 HEAD flit 競爭同一 output port，進行 QoS-Aware Round-Robin：

```cpp
struct QoSAwareArbiter {
    // Per-output-port RR pointer: 上次 grant 的 input direction 的下一個
    // Direction 順序固定為: NORTH=0, SOUTH=1, EAST=2, WEST=3, LOCAL=4
    std::map<Direction, int> rr_pointer;  // 初始值 = 0 (NORTH)

    // Direction → integer mapping（固定順序，用於 Round-Robin）
    static constexpr Direction dir_order[] = {
        NORTH, SOUTH, EAST, WEST, LOCAL
    };
    static constexpr int NUM_DIRS = 5;

    static int dir_to_idx(Direction d) {
        for (int i = 0; i < NUM_DIRS; i++)
            if (dir_order[i] == d) return i;
        return -1;
    }

    // Round-Robin 選擇：從 rr_pointer 指向的 direction 開始，
    // 按固定順序掃描，選擇第一個在 candidates 中的 input direction
    RouteRequest select_rr(const std::vector<RouteRequest>& candidates, int rr) {
        for (int i = 0; i < NUM_DIRS; i++) {
            int idx = (rr + i) % NUM_DIRS;
            Direction scan_dir = dir_order[idx];
            for (auto& c : candidates) {
                if (c.in_dir == scan_dir) return c;
            }
        }
        return candidates[0];  // fallback (should not reach)
    }

    // RR pointer 更新：指向 winner 的下一個 direction
    int next_rr(Direction winner_dir) {
        return (dir_to_idx(winner_dir) + 1) % NUM_DIRS;
    }

    std::vector<Grant> arbitrate(
        const std::vector<RouteRequest>& new_requests
    ) {
        // Group requests by target output port
        std::map<Direction, std::vector<RouteRequest>> per_output;
        for (auto& req : new_requests) {
            if (is_output_available(req.out_dir)) {
                per_output[req.out_dir].push_back(req);
            }
        }

        std::vector<Grant> winners;
        std::set<Direction> used_outputs;
        std::set<Direction> used_inputs;

        for (auto& [out_dir, candidates] : per_output) {
            if (candidates.empty()) continue;

            // Step 1: Find max qos among candidates
            uint8_t max_qos = 0;
            for (auto& c : candidates) {
                uint8_t qos = c.flit.header.qos;
                if (qos > max_qos) max_qos = qos;
            }

            // Step 2: Filter to highest-qos candidates only
            std::vector<RouteRequest> top_candidates;
            for (auto& c : candidates) {
                if (c.flit.header.qos == max_qos &&
                    used_inputs.find(c.in_dir) == used_inputs.end()) {
                    top_candidates.push_back(c);
                }
            }

            // Step 3: Round-Robin among top candidates
            if (!top_candidates.empty()) {
                int rr = rr_pointer[out_dir];
                RouteRequest winner = select_rr(top_candidates, rr);
                winners.push_back({ winner.in_dir, out_dir, winner.flit });
                used_outputs.insert(out_dir);
                used_inputs.insert(winner.in_dir);
                rr_pointer[out_dir] = next_rr(winner.in_dir);
            }
        }

        return winners;
    }
};
```

### 7.3 QoS 與 Wormhole 的交互

QoS arbitration **僅影響 HEAD flit**。已鎖定路徑的 BODY/TAIL flit 不參與仲裁，直接通過 crossbar：

```
Packet A (qos=15): [HEAD] → [BODY] → [BODY] → [TAIL]
Packet B (qos=3):  [HEAD] → [BODY] → [TAIL]

Cycle 1: A.HEAD vs B.HEAD 競爭 output → A wins (qos=15 > 3)
         A.HEAD 獲得 path lock
Cycle 2: A.BODY 直接通過（locked path, 不仲裁）
         B.HEAD 仍在等待
Cycle 3: A.BODY 通過
Cycle 4: A.TAIL 通過，lock 釋放
Cycle 5: B.HEAD 獲得 output（此時無競爭）
```

### 7.4 Priority Inversion

低 QoS 的長 packet 鎖定路徑時，高 QoS 的 HEAD flit 必須等待。這是 wormhole switching 的固有特性，本設計接受此 priority inversion 並依賴以下機制緩解：

| 緩解機制 | 說明 |
|----------|------|
| AXI burst length 限制 | 限制 packet 最大 flit 數，減少 blocking 時間 |
| Age-based promotion (optional) | 等待過久的 flit 自動提升 effective qos |
| Per-QoS Virtual Channel (future) | 不同 QoS 使用不同 VC，物理隔離 |

### 7.5 Anti-Starvation

Round-Robin pointer 確保同 qos 的 HEAD flit 獲得公平服務。低 qos flit 不會被永久阻塞：當高 qos 的 packets 全部完成傳輸後，低 qos 的 HEAD flit 自然獲得仲裁機會。

---

## 8. Crossbar Structure

### 8.1 Overview

Crossbar 為 Router 的核心 switching fabric，實現任意 input port 到任意 output port 的 non-blocking 連接（同一 output 每 cycle 僅一個 input）。

```
                        Output Ports
                   N     S     E     W     L
                   |     |     |     |     |
  Input   N ─── [ MUX ] [ MUX ] [ MUX ] [ MUX ] [ MUX ]
  Ports   S ─── [     ] [     ] [     ] [     ] [     ]
          E ─── [     ] [     ] [     ] [     ] [     ]
          W ─── [     ] [     ] [     ] [     ] [     ]
          L ─── [     ] [     ] [     ] [     ] [     ]
                   |     |     |     |     |
                  out   out   out   out   out
```

### 8.2 Switching Constraints

| Constraint | Description |
|------------|-------------|
| No U-turn | Input N 不能 output N（不可原路返回） |
| No self-loop | Input L 不能 output L（LOCAL→LOCAL 無意義） |
| One winner per output | 每個 output port 每 cycle 僅接受一個 input |
| One output per input (Unicast) | Unicast 模式下，每個 input port 每 cycle 僅送往一個 output |
| Multi-output per input (Multicast) | Multicast 模式下，一個 input port 可送往多個 output（independent per-port handshake，見 Section 6.6） |

### 8.3 Internal Implementation

#### 8.3.1 Unicast Switching

每個 output port 由一個 (NUM_MESH_PORTS + NUM_LOCAL_PORTS):1 MUX 控制。Arbitration 結果決定 MUX select signal：

```cpp
void crossbar_switch(Direction in_dir, Direction out_dir, const Flit& flit) {
    // 注意：不在此處 pop input buffer。
    // Pop 由 Phase 4 的 MulticastTracker.all_done() 控制（Section 6.6）。
    // Unicast 時 all_done() 在同 cycle 即為 true，等效於立即 pop。

    // Route through crossbar to output
    if (ports[out_dir].output_buffer_enabled()) {
        ports[out_dir].output_buffer.push(flit);
    } else {
        // Bypass: directly set output signals
        ports[out_dir].out_valid = true;
        ports[out_dir].out_flit  = flit;
    }
}
```

#### 8.3.2 Multicast Switching（Independent Per-Port）

Multicast 時，routing function 產生 multi-hot output port set（見 [RCR-Multicast Spec](rectangle_multicast_spec_zh-TW.md) Section 3.1）。Crossbar 採用 **independent per-port handshake** 策略：各 target output 可在不同 cycle 獨立接受 flit。

**核心規則：每個可用的 target output 獨立完成 handshake。已完成的 output 立即釋放。Input buffer 在所有 target output 都完成後才 pop。**

Multicast 的 crossbar switch 複用 unicast 的 `crossbar_switch()` 函數，由 Phase 4 逐 output port 呼叫：

```cpp
// Phase 4 中對每個 target output 獨立呼叫（見 Section 5 Phase 4 pseudocode）
for (Direction out : grant.out_dirs) {
    if (tracker.is_completed(out)) continue;   // 已完成，壓掉 valid
    if (can_switch_to_output(out)) {
        crossbar_switch(grant.in_dir, out, grant.flit);  // 複用 unicast switch
        tracker.current_handshakes.insert(out);
    }
}
// Input buffer pop 由 tracker.all_done() 控制，不在 crossbar_switch 內
```

**Independent per-port handshake 的特性：**

| 特性 | 說明 |
|------|------|
| Output 早期釋放 | 已完成 handshake 的 output 立即可服務其他 traffic |
| 無 collateral blocking | 不阻擋任何 input port，不影響無關 traffic |
| Input HoL blocking | Flit 在 input buffer 停留至所有 output 完成，此期間該 input blocked |
| Flit 複製 | 同一 flit data 經 crossbar wire fan-out 寫入多個 output（每次 switch 一份） |
| Arbiter 無感知 | Output arbiter 為標準 wormhole arbiter，不需 multicast 特殊邏輯 |

### 8.4 Output Buffer 與 Wire-Through 模式

Output buffer 為可選配置，位於 crossbar 與 physical link 之間。**預設 `OUTPUT_BUFFER_DEPTH=2`**。

```
Data Path (OUTPUT_BUFFER_DEPTH >= 1):
  InputBuffer → Crossbar → OutputBuffer → Physical Link
                            (depth=N)

Data Path (OUTPUT_BUFFER_DEPTH = 0, wire-through):
  InputBuffer → Crossbar ─────────────→ Physical Link
                          (direct wire)
```

#### OUTPUT_BUFFER_DEPTH >= 1（啟用 Output Buffer）

1. Crossbar 輸出寫入 output buffer（若 buffer 有空間）
2. Output buffer head flit 驅動 `out_valid` / `out_flit`
3. Downstream 接受後（`in_ready=1`），pop buffer head
4. Crossbar 可寫入的條件：`!output_buffer.full()`
5. Input buffer 在 crossbar switch 時立即 pop（flit 已轉移至 output buffer）

#### OUTPUT_BUFFER_DEPTH = 0（Wire-Through 模式）

此模式下 **不存在 output buffer**，crossbar output 直接連接 physical link：

1. Crossbar 直接驅動 `out_valid` / `out_flit`（combinational path）
2. Crossbar 可寫入的條件：output port 當前 `out_valid == false`（上一 cycle 已被下游接受）**或** downstream `in_ready == 1`（可即時接受）
3. 若 downstream 未接受（`in_ready=0` 且 `out_valid` 已設），flit **保留在 input buffer 不 pop**，下一 cycle 重新嘗試
4. Input buffer 是唯一的 buffering point — 所有 flow control 壓力由 input buffer 承擔

```cpp
// Crossbar switch 判斷 output 是否可寫
bool can_switch_to_output(Direction out_dir) const {
    if (ports[out_dir].output_buffer_enabled()) {
        return !ports[out_dir].output_buffer.full();
    } else {
        // Wire-through: 只有在 output 空閒時才可 switch
        return !ports[out_dir].out_valid;
    }
}
```

> **Note:** Wire-through 模式下 Router 的 throughput 受限於 downstream 的接收速度。啟用 output buffer 可解耦 crossbar 與 downstream timing，提升 throughput。本設計預設 `OUTPUT_BUFFER_DEPTH=2`。

---

## 9. Wormhole Stall & Backpressure

### 9.1 Stall 觸發條件

當 output port 無法接受新 flit 時，產生 backpressure：

| Condition | Cause | Effect |
|-----------|-------|--------|
| Output buffer full | `output_buffer.full() == true` | Crossbar 無法 switch 到此 output |
| Downstream not ready | `in_ready == 0`（downstream input buffer 滿） | Output buffer 無法 drain |
| Output port locked | 另一個 input 的 wormhole lock 佔用此 output | HEAD flit 無法獲得此 output |

### 9.2 Backpressure 傳播

Backpressure 沿 wormhole path 反向傳播，逐 hop 影響上游 Router：

```
Router A           Router B           Router C
┌──────────┐      ┌──────────┐      ┌──────────┐
│ input buf│──────│ input buf│──────│ input buf│ ← FULL
│          │      │          │      │  (stall) │
│ out_valid│─────►│ in_valid │      │          │
│          │◄─────│ out_ready│=0    │          │ ← backpressure
│ (stall)  │      │ (stall)  │◄─────│ out_ready│=0
└──────────┘      └──────────┘      └──────────┘

Timeline:
  Cycle T:   C's input buffer full → C.out_ready = 0
  Cycle T+1: B sees C.out_ready=0 → B cannot drain → B.out_ready = 0
  Cycle T+2: A sees B.out_ready=0 → A cannot drain → A stalls
```

### 9.3 Stall 對 Wormhole Path 的影響

當 wormhole locked path 上某 hop 發生 stall，**整條 path 上所有 flits 暫停**：

- BODY/TAIL flit 停留在上游 Router 的 input buffer
- Input buffer 佔用期間，該 input port 的 `out_ready=0`，阻止更上游寫入
- **Path lock 不釋放** — 直到 stall 解除，TAIL flit 最終通過並釋放 lock

### 9.4 Head-of-Line (HoL) Blocking

Wormhole switching 存在 HoL blocking：一個被 stall 的 wormhole path 會阻塞共用 input buffer 中其他 packet 的 HEAD flit。

```
Input buffer (input port E):
  [Position 0] BODY flit of Packet A → locked to output N (stalled)
  [Position 1] HEAD flit of Packet B → wants output S (available)
               但 Packet B 被 Packet A 阻塞，無法被仲裁
```

此為 wormhole switching 的已知限制。Virtual Channel (VC) 可緩解 HoL blocking（每個 VC 有獨立 buffer，不同 VC 的 packet 互不阻塞），但 VC 增加硬體複雜度。

---

## 10. Req/Rsp 物理分離

### 10.1 Architecture

每個邏輯 Router 由兩個完全獨立的子 Router 組成，分別處理 Request 與 Response network：

```cpp
struct Router {
    Coord       coord;      // (x, y)
    ReqRouter   req_router; // Request network: AW, W, AR
    RspRouter   rsp_router; // Response network: B, R
};

struct ReqRouter : public XYRouter {
    // 處理 axi_ch = {0 (AW), 1 (W), 2 (AR)}
};

struct RspRouter : public XYRouter {
    // 處理 axi_ch = {3 (B), 4 (R)}
    // 額外支援 In-Network Reduction（commtype = ParallelReduction）
    ReductionSync   reduction_sync;    // per-output port
    ReductionArbiter reduction_arbiter; // per-output port
};
```

### 10.2 分離理由

Request 與 Response 使用獨立 physical link（各 410 bits/direction），消除 request-response circular dependency。這是 protocol-level deadlock avoidance 的主要機制：

```
Router A  ════════ Req Link (410b) ════════►  Router B
          ◄════════ Req Link (410b) ════════
          ════════ Rsp Link (410b) ════════►
          ◄════════ Rsp Link (410b) ════════
```

Req Router 與 Rsp Router 共用相同的 routing algorithm、arbitration policy、crossbar structure，僅 flit 所屬 `axi_ch` 不同。**Rsp Router 額外支援 In-Network Reduction**（見 Section 10.3）。NI 根據 flit header 的 `axi_ch` 欄位分流至對應 network。

### 10.3 In-Network Reduction（Response Router Only）

Multicast write 的 B response 在 Response Router 中逐 hop 合併，Source NMU 僅收到 **1 個** 已合併的 B response。

#### 10.3.1 概述

```
Forward (Multicast):     NMU → Req Router → fan-out → NSU₀, NSU₁, ..., NSUₙ₋₁
Backward (Reduction):    NSU₀, ..., NSUₙ₋₁ → Rsp Router → fan-in → NMU (1 merged B)
                                              ↑ 每個 hop 逐步合併
```

**commtype 轉換**：Dest NI 收到 Multicast AW → 產生的 B flit 標記為 `commtype = ParallelReduction (2)`。Router 據此區分 unicast response 與 reduction response。

#### 10.3.2 Backward In-Route Mask

Reduction 的關鍵在於：每個 router 需知道有幾個方向會送來 response。

**核心洞察：backward `in_route_mask` = forward `route_sel`。** 因為 forward multicast fan-out 到的每個方向，backward 都會有 response 從該方向回來。

```cpp
// Reduction sync — 使用與 forward RCR 完全相同的公式
// 計算此 router 應等待哪些 input 方向的 response
void compute_in_route_mask(
    Coord my_pos, Coord src,
    uint8_t x_min, uint8_t x_max, uint8_t y_min, uint8_t y_max,
    std::set<Direction>& in_route_mask)
{
    bool x_in_range = (my_pos.x >= x_min) && (my_pos.x <= x_max);
    bool y_in_range = (my_pos.y >= y_min) && (my_pos.y <= y_max);
    bool on_src_row = (my_pos.y == src.y);

    in_route_mask.clear();
    if (x_in_range && y_in_range)
        in_route_mask.insert(Direction::LOCAL);
    if (on_src_row && (my_pos.x >= src.x) && (my_pos.x < x_max))
        in_route_mask.insert(Direction::EAST);
    if (on_src_row && (my_pos.x <= src.x) && (my_pos.x > x_min))
        in_route_mask.insert(Direction::WEST);
    if (x_in_range && (my_pos.y >= src.y) && (my_pos.y < y_max))
        in_route_mask.insert(Direction::NORTH);
    if (x_in_range && (my_pos.y <= src.y) && (my_pos.y > y_min))
        in_route_mask.insert(Direction::SOUTH);
}
```

#### 10.3.3 ReductionSync

同步等待所有 expected inputs，確保合併前所有 response 已到達。

```cpp
struct ReductionSync {
    std::set<Direction> in_route_mask;  // 預期 response 來源方向
    std::set<Direction> arrived;        // 已到達的方向

    // 從 ParallelReduction flit 的 multicast_mask + src_id 計算 in_route_mask
    void configure(const FlitHeader& hdr, Coord my_pos);

    // 檢查某方向的 ParallelReduction flit 是否屬於此 reduction group
    // 比對 multicast_mask 和 src_id（同一 multicast transaction 的所有 B flit 攜帶相同值）
    bool matches(const FlitHeader& hdr) const;

    // 記錄到達
    void mark_arrived(Direction dir) { arrived.insert(dir); }

    // 所有 expected inputs 都到達？
    bool all_arrived() const { return arrived == in_route_mask; }

    // Reset for next reduction
    void reset() { in_route_mask.clear(); arrived.clear(); }
};
```

**同步條件**：所有 `in_route_mask` 中的方向都有 valid 的 ParallelReduction flit，且它們的 `multicast_mask` 和 `src_id` 相同（屬於同一 multicast transaction）。

#### 10.3.4 ReductionArbiter

合併多個 B response 為單一 B flit。

```cpp
struct ReductionArbiter {
    // 合併規則：
    // - bresp: 任一 SLVERR → 最終 SLVERR；全部 OKAY → OKAY
    // - ecc_fail: 任一 true → 最終 true
    // - multicast_status: 不使用（由最終合併結果的 bresp 推導）
    Flit reduce(const std::map<Direction, Flit>& inputs,
                const std::set<Direction>& in_route_mask)
    {
        Flit merged = inputs.begin()->second;  // 取第一個作為 base
        bool any_error = false;
        bool any_ecc_fail = false;

        for (auto dir : in_route_mask) {
            auto& b = inputs.at(dir);
            uint8_t bresp = extract_bresp(b.payload);
            bool ecc = extract_ecc_fail(b.payload);
            if (bresp != 0) any_error = true;   // Non-OKAY
            if (ecc) any_ecc_fail = true;
        }

        set_bresp(merged.payload, any_error ? 2 : 0);  // SLVERR or OKAY
        set_ecc_fail(merged.payload, any_ecc_fail);
        // commtype 維持 ParallelReduction（下一 hop 繼續 reduction）
        return merged;
    }
};
```

#### 10.3.5 Reduction Output Routing

合併後的 B flit 以 **unicast XY routing** 向 `src_id`（原始 multicast 的 source）方向輸出：

```cpp
// 合併完成後，計算 output direction（標準 unicast toward src_id）
Direction compute_reduction_output(Coord my_pos, Coord src) {
    if (my_pos.x != src.x)
        return (my_pos.x > src.x) ? Direction::WEST : Direction::EAST;
    if (my_pos.y != src.y)
        return (my_pos.y > src.y) ? Direction::SOUTH : Direction::NORTH;
    return Direction::LOCAL;  // 已到達 source
}
```

合併後的 flit 保持 `commtype = ParallelReduction`，使下一 hop 的 Response Router 繼續等待其他方向的 response 並執行 reduction。**Source Router 的最後一次 reduction 完成後，output direction = LOCAL → 送入 Source NMU。**

#### 10.3.6 Reduction 時序範例

Source=(0,1), rect=[1,3]×[0,2] (9 nodes):

```
Cycle T:    NSU(1,0), NSU(1,2), NSU(3,0), NSU(3,2) 產生 B flit
Cycle T+1:  Router(1,1) 收到 Local + N + S → 等待 East（尚未到達）
            Router(3,1) 收到 Local + N + S → 合併 → 送 West
Cycle T+2:  Router(2,1) 收到 Local + N + S + East(from 3,1) → 合併 → 送 West
Cycle T+3:  Router(1,1) 收到 East(from 2,1) → 全部到齊 → 合併 → 送 West
Cycle T+4:  Router(0,1) 收到 East(from 1,1) → 合併 → LOCAL → Source NMU
```

Source NMU 在 T+4 收到 **1 個** 已合併的 B response。

#### 10.3.7 Unicast Response 不受影響

Reduction 僅處理 `commtype = ParallelReduction` 的 flit。Unicast response（`commtype = 0`）和 R channel response 走標準 arbiter 路徑，不受 reduction 邏輯影響。Response Router 內部以 `commtype` 分流：

```cpp
if (flit.header.commtype == CommType::ParallelReduction) {
    // → ReductionSync + ReductionArbiter path
} else {
    // → 標準 wormhole arbiter path（unicast）
}
```

> **Note**: R channel（Read response）為 multi-flit wormhole packet，不支援 in-network reduction，一律走 unicast path。

---

## 11. Multi-Port LOCAL 架構

### 11.1 概述

每個 Router 可配置 0~4 個 LOCAL port，由 header 的 `port_id`（2 bits）識別，允許同一 Router 連接多種不同類型的 NI（如 CPU、DMA、Memory Controller、Accelerator）。

```
                  N
                  │
         W ── [Router] ── E
                  │
                  S
                  │
         ┌───┬───┼───┬───┐
         L0  L1  L2  L3       ← port_id = 0, 1, 2, 3
         │   │   │   │
        NI  NI  NI  NI        ← 各自獨立的 Network Interface
```

`NUM_LOCAL_PORTS = 0` 的 Router 無 LOCAL port，僅作為純轉發節點。

### 11.2 LOCAL Port 結構

每個 LOCAL port 為完整獨立的 RouterPort 實例：

| 屬性 | 說明 |
|------|------|
| Input Buffer | 獨立，depth = `INPUT_BUFFER_DEPTH` |
| Output Buffer | 獨立，depth = `OUTPUT_BUFFER_DEPTH` |
| Credit Counter | 獨立，追蹤對端 NI 的 buffer space |
| Abstract Interface | 獨立，透過 `wire_all()` 連接對應 NI |

### 11.3 Routing 與 port_id 的關係

XY routing 僅決定方向（N/S/E/W/**LOCAL**），不涉及 `port_id`。當 routing 輸出為 LOCAL 時，`port_id` 決定送往哪個 LOCAL port：

```cpp
Direction compute_output_direction(const Flit& flit, Direction input_dir) {
    uint8_t dst_x = flit.header.dst_id & 0x0F;
    uint8_t dst_y = (flit.header.dst_id >> 4) & 0x0F;

    if (dst_x != coord.x)
        return (dst_x > coord.x) ? EAST : WEST;
    if (dst_y != coord.y)
        return (dst_y > coord.y) ? NORTH : SOUTH;
    return LOCAL;  // dst matches this router → deliver to local port
}

// LOCAL 內部 port 選擇
int select_local_port(const Flit& flit) {
    assert(flit.header.port_id < num_local_ports);
    return flit.header.port_id;  // 0~3
}
```

### 11.4 Crossbar 擴展

Crossbar 大小為 `(NUM_MESH_PORTS + NUM_LOCAL_PORTS) × (NUM_MESH_PORTS + NUM_LOCAL_PORTS)`。典型配置：

| 配置 | Crossbar 大小 | 說明 |
|------|--------------|------|
| 1 local port | 5×5 | 標準配置（4 mesh + 1 local） |
| 2 local ports | 6×6 | CPU + DMA 共享 Router |
| 0 local ports | 4×4 | 純轉發節點 |
| 4 local ports | 8×8 | 最大配置 |

### 11.5 No-UTurn 擴展

No-UTurn 規則在 multi-port LOCAL 下的行為：

- Mesh port 間的 no-UTurn 規則不變（不允許同方向返回）
- LOCAL port 之間：LOCAL port `i` 輸入的 flit 不允許轉發至同一 LOCAL port `i`（self-loop），但可以轉發至同一 Router 的不同 LOCAL port `j`（inter-port 直連通信）
- Mesh→LOCAL、LOCAL→Mesh 方向不受 no-UTurn 限制

```cpp
bool is_uturn(Direction in_dir, int in_port_id, Direction out_dir, int out_port_id) {
    if (in_dir == out_dir) {
        if (in_dir != LOCAL) return true;           // mesh 同方向 UTurn
        if (in_port_id == out_port_id) return true; // 同一 local port UTurn
    }
    return false;
}
```

---

## 12. Router 完整結構圖

```
                          Physical Links (410b each)
                               N port
                          in ◄──┤├──► out
                                │
           W port          ┌────┴────┐          E port
      in ◄──┤├──► out ────►│         │◄──── in ◄──┤├──► out
                           │ Crossbar │
      out ◄──┤├──► in ◄────│  (5x5)  │────► out ◄──┤├──► in
                           │         │
                           └────┬────┘
                                │
                          out ◄──┤├──► in
                               S port
                                │
                          in ◄──┤├──► out
                               L0 port (LOCAL, port_id=0)
                                │
                              ┌─┴─┐
                              │NI │  ← NUM_LOCAL_PORTS 可為 0~4
                              └───┘     port_id 識別各 local port

  Each port:
  ┌──────────────────────────────────────────────────────────┐
  │  [Ingress]          [Input     ]   [Crossbar]   [Output  ]   [Egress]    │
  │  in_valid ──────►  Buffer      ──► Switch    ──► Buffer   ──► out_valid  │
  │  in_flit  ──────►  (depth=4)   ──► (MUX)    ──► (depth=2*) ─► out_flit  │
  │  out_ready ◄──────  !full()                      !full()  ◄── in_ready   │
  │                                                                          │
  │  * OUTPUT_BUFFER_DEPTH=2 (預設). depth=0 時 output buffer 不存在,         │
  │    crossbar 直接驅動 out_valid/out_flit (wire-through mode)               │
  └──────────────────────────────────────────────────────────┘
```

---

## 13. C++ Implementation Summary

### 13.1 Key Data Structures

```cpp
// Flit header (56 bits) - matching 04_flit.md
struct FlitHeader {
    uint8_t  qos;              // [3:0]   4 bits
    uint8_t  axi_ch;           // [6:4]   3 bits
    uint8_t  src_id;           // [14:7]  8 bits
    uint8_t  dst_id;           // [22:15] 8 bits
    uint8_t  port_id;          // [24:23] 2 bits
    bool     last;             // [25]    1 bit
    bool     rob_req;          // [26]    1 bit
    uint8_t  rob_idx;          // [31:27] 5 bits
    uint8_t  commtype;         // [33:32] 2 bits
    uint16_t multicast_mask;   // [49:34] 16 bits {x_min, x_max, y_min, y_max}
    uint8_t  vc_id;            // [52:50] 3 bits (Version B)
    uint8_t  rsvd;             // [55:53] 3 bits
};

// Flit (408 bits)
struct Flit {
    FlitHeader header;              // 56 bits
    uint8_t    payload[44];         // 352 bits (44 bytes)
};

// Direction enumeration
enum class Direction {
    NORTH, SOUTH, EAST, WEST, LOCAL, NONE
};

// Router configuration
struct RouterConfig {
    int  input_buffer_depth  = 4;
    int  output_buffer_depth = 2;   // 0 = no output buffer
    int  num_local_ports     = 1;   // 0~4, identified by port_id
    // All flit parameters are fixed (not configurable)
};
```

### 13.2 Router Class Skeleton

```cpp
class XYRouter {
public:
    Coord coord;
    int   num_local_ports;                          // 0~4
    std::map<Direction, RouterPort> mesh_ports;      // N, S, E, W
    std::vector<RouterPort>        local_ports;      // index = port_id (0~3)
    std::map<Direction, PathLock>   path_locks;
    QoSAwareArbiter arbiter;

    // Phase 1
    void sample_all_inputs();
    // Phase 2
    void clear_all_input_signals();
    // Phase 3
    void update_all_ready();
    // Phase 4
    void route_and_forward(int current_time);
    // Phase 6
    void clear_accepted_outputs();

private:
    Direction compute_output_direction(const Flit& flit, Direction input_dir);
    int       select_local_port(const Flit& flit);  // port_id → local_ports index
    std::set<Direction> compute_output_ports(const Flit& flit, Direction input_dir);
    std::set<Direction> compute_rcr_outputs(const Flit& flit, Direction input_dir);
    void crossbar_switch(Direction in_dir, Direction out_dir, const Flit& flit);
    bool is_output_available(Direction out_dir) const;
    bool can_switch_to_output(Direction out_dir) const;

    // Multicast per-port handshake tracking (Section 6.6)
    std::map<Direction, MulticastTracker> mc_trackers;
};

class Router {
public:
    Coord       coord;
    XYRouter    req_router;   // Request network
    XYRouter    rsp_router;   // Response network
};
```

---

## 14. CppRouter 內部 Pipeline Stages

本 section 定義 `CppRouter` 的內部 pipeline stage 分解，將 pipeline 各階段映射至本設計的 8-phase cycle model。此為 Phase 4 (Route & Forward) 的內部子步驟展開。

### 14.1 Pipeline Stage 對應

| Pipeline Stage | CppRouter Function | Phase 對應 | 說明 |
|------------------------|--------------------|-----------|------|
| Input Queuing | `_InputQueuing()` | Phase 1 | Sample inputs → push to input buffer |
| Route Evaluate | `_RouteEvaluate()` | Phase 4a | XY route computation → output port determination |
| VC Alloc Evaluate | `_VCAllocEvaluate()` | Phase 4b | VC allocation (Version B，pluggable Allocator) |
| SW Alloc Evaluate | `_SWAllocEvaluate()` | Phase 4c | Switch arbitration (QoS-aware RR / iSLIP) |
| Switch Traversal | `_SwitchTraverse()` | Phase 4d | Crossbar traversal |
| Output Queuing | `_OutputQueuing()` | Phase 4e | Write to output buffer |

### 14.2 CppRouter Class Skeleton（含 Pipeline Stages）

```cpp
template<FlowControlMode Mode>
class CppRouter : public Router_Interface<Mode> {
public:
    // === Router_Interface 合約 ===
    int num_ports() const override;
    const PortOutput<Mode>& get_output(int port, Channel ch) const override;
    void set_input(int port, Channel ch, const PortOutput<Mode>& in) override;
    void tick() override;       // Phase 1-4
    void post_wire() override;  // Phase 6
    void process_credit_returns() override;  // Phase 7

    // === 內部 pipeline stages ===
private:
    void _InputQueuing();       // Phase 1: Sample inputs → push to buffer
    void _RouteEvaluate();      // Phase 4a: XY route computation
    void _VCAllocEvaluate();    // Phase 4b: VC allocation (Version B)
    void _SWAllocEvaluate();    // Phase 4c: Switch arbitration (QoS-aware RR)
    void _SwitchTraverse();     // Phase 4d: Crossbar traversal
    void _OutputQueuing();      // Phase 4e: Write to output buffer

    // Wormhole FSM
    void _UpdatePathLocks();

    // === tick() 內部實作 ===
    // void tick() {
    //     _InputQueuing();        // Phase 1
    //     clear_all_inputs();     // Phase 2
    //     update_all_ready();     // Phase 3
    //     _RouteEvaluate();       // Phase 4a
    //     _VCAllocEvaluate();     // Phase 4b
    //     _SWAllocEvaluate();     // Phase 4c
    //     _SwitchTraverse();      // Phase 4d
    //     _OutputQueuing();       // Phase 4e
    // }

    // === 內部元件 ===
    std::vector<InputBuffer>    input_buffers_;    // per-port, per-VC
    std::vector<BufferState>    next_buf_states_;  // per-output: downstream credit
    std::vector<FlitBuffer>     output_buffers_;   // per-port (optional)
    std::vector<PathLock>       path_locks_;       // per-input: wormhole lock
    std::vector<MulticastTracker> mc_trackers_;    // per-input: multicast tracking

    std::unique_ptr<Allocator>  vc_allocator_;     // pluggable
    std::unique_ptr<Allocator>  sw_allocator_;     // pluggable
    RouteCompute                route_compute_;    // XY routing + multicast RCR
    Coord                       coord_;
    RouterStats                 stats_;

    // Phase 4 intermediate results
    struct RouteResult {
        int input_port;
        int output_port;
        int output_vc;
        Flit flit;
    };
    std::vector<RouteResult> route_results_;       // _RouteEvaluate 產出
    std::vector<RouteResult> vc_alloc_results_;    // _VCAllocEvaluate 產出
    std::vector<RouteResult> sw_alloc_results_;    // _SWAllocEvaluate 產出
};
```

### 14.3 Pipeline Stage 詳細說明

#### 14.3.1 _InputQueuing() — Phase 1

從 `set_input()` 設定的信號讀取 flit，push 到 per-VC input buffer：

```cpp
void _InputQueuing() {
    for (int p = 0; p < num_ports(); p++) {
        auto& in = inputs_[p];
        if constexpr (Mode == FlowControlMode::VALID_READY) {
            for (int vc = 0; vc < num_vc_; vc++) {
                if (in.valid[vc] && outputs_[p].ready[vc]) {
                    input_buffers_[p].Push(vc, in.flit[vc]);
                }
            }
        } else {
            if (in.valid) {
                int vc = in.flit.header.vc_id;
                input_buffers_[p].Push(vc, in.flit);
            }
        }
    }
}
```

#### 14.3.2 _RouteEvaluate() — Phase 4a

對每個 input buffer 的 HEAD flit 計算 output port：

```cpp
void _RouteEvaluate() {
    route_results_.clear();
    for (int p = 0; p < num_ports(); p++) {
        for (int vc = 0; vc < num_vc_; vc++) {
            if (input_buffers_[p].Empty(vc)) continue;
            const Flit& flit = input_buffers_[p].Peek(vc);

            // Wormhole locked path: 沿用已鎖定的 output
            if (path_locks_[p].is_locked()) {
                for (int out : path_locks_[p].get_outputs()) {
                    route_results_.push_back({p, out, -1, flit});
                }
            } else if (is_head_flit(flit)) {
                // HEAD flit: compute routing
                auto out_ports = route_compute_.compute(flit, p, coord_);
                for (int out : out_ports) {
                    route_results_.push_back({p, out, -1, flit});
                }
            }
        }
    }
}
```

#### 14.3.3 _VCAllocEvaluate() — Phase 4b

Version B only — 為 HEAD flit 分配目標 VC：

```cpp
void _VCAllocEvaluate() {
    vc_alloc_results_ = route_results_;  // Pass-through for Version A

    if constexpr (Mode == FlowControlMode::CREDIT) {
        vc_allocator_->Clear();
        for (auto& r : route_results_) {
            if (!path_locks_[r.input_port].is_locked()) {
                // Request: input VC → output VC (with priority from QoS)
                for (int ovc = 0; ovc < num_vc_; ovc++) {
                    if (next_buf_states_[r.output_port].IsFree(ovc)) {
                        vc_allocator_->AddRequest(
                            r.input_port * num_vc_ + 0,  // input id
                            r.output_port * num_vc_ + ovc,  // output id
                            r.flit.header.qos);
                    }
                }
            }
        }
        vc_allocator_->Allocate();
        // Update results with allocated VCs
        for (auto& r : vc_alloc_results_) {
            int grant = vc_allocator_->OutputAssignment(
                r.input_port * num_vc_ + 0);
            if (grant >= 0) {
                r.output_vc = grant % num_vc_;
            }
        }
    }
}
```

#### 14.3.4 _SWAllocEvaluate() — Phase 4c

Switch allocation — 決定每個 output port 的 winner：

```cpp
void _SWAllocEvaluate() {
    sw_allocator_->Clear();
    for (auto& r : vc_alloc_results_) {
        bool can_send = true;
        if constexpr (Mode == FlowControlMode::CREDIT) {
            can_send = next_buf_states_[r.output_port].HasCredit(r.output_vc);
        }
        if (can_send) {
            sw_allocator_->AddRequest(r.input_port, r.output_port,
                                      r.flit.header.qos);
        }
    }
    sw_allocator_->Allocate();

    sw_alloc_results_.clear();
    for (auto& r : vc_alloc_results_) {
        if (sw_allocator_->OutputAssignment(r.input_port) == r.output_port) {
            sw_alloc_results_.push_back(r);
        }
    }
}
```

#### 14.3.5 _SwitchTraverse() — Phase 4d

執行 crossbar switching：

```cpp
void _SwitchTraverse() {
    for (auto& r : sw_alloc_results_) {
        crossbar_switch(r.input_port, r.output_port, r.flit);
        // Update wormhole path lock
        _UpdatePathLocks();
        // Update multicast tracker
        mc_trackers_[r.input_port].current_handshakes.insert(r.output_port);
        // Credit consume (Version B)
        if constexpr (Mode == FlowControlMode::CREDIT) {
            next_buf_states_[r.output_port].ConsumeCredit(r.output_vc);
        }
        stats_.flits_forwarded++;
    }
}
```

#### 14.3.6 _OutputQueuing() — Phase 4e

Push crossbar output to output buffer（若啟用），或直接設定 output signals：

```cpp
void _OutputQueuing() {
    for (int p = 0; p < num_ports(); p++) {
        if (output_buffers_[p].depth() > 0) {
            // Output buffer enabled: head → output signals
            if (!output_buffers_[p].empty()) {
                set_output_signals(p, output_buffers_[p].peek());
            }
        }
        // Wire-through mode: output signals already set by _SwitchTraverse
    }
}
```

### 14.4 Pipeline Delay 支援

NocConfig 的 `routing_delay` / `vc_alloc_delay` / `sw_alloc_delay` 參數可為各 stage 加入額外 cycle。當 delay > 0 時，對應 stage 的結果需延遲指定 cycle 才送入下一 stage（使用 internal staging register）：

```cpp
// 例: routing_delay = 1 時，_RouteEvaluate 的結果延遲 1 cycle
// 才傳給 _VCAllocEvaluate
if (config_.routing_delay > 0) {
    route_result_pipeline_.push_back(route_results_);
    if (route_result_pipeline_.size() > config_.routing_delay) {
        route_results_ = route_result_pipeline_.front();
        route_result_pipeline_.pop_front();
    } else {
        route_results_.clear();  // 延遲期間無輸出
    }
}
```

---

## 15. Error Handling

本 section 定義 Router 層面的 error 偵測與處理策略。模擬目標為 **快速偵測設計 bug**（assert 中止），而非 runtime error recovery。

### 15.1 Error 分類與處理

| Error | 偵測方式 | 處理方式 | 嚴重度 |
|-------|---------|---------|--------|
| Credit underflow | `credit[vc] < 0` | `assert()` 失敗，simulation 中止 | FATAL |
| Buffer overflow | `input_buffer.push()` when full | `assert()` 失敗，simulation 中止 | FATAL |
| Invalid destination | `dst_id` 座標超出 mesh 範圍 | Drop flit + log warning | ERROR |
| Y→X turn violation | XY routing assertion | `assert()` 失敗（不應發生） | FATAL |
| ECC failure | 偵測於 destination NI | 標記 `ecc_fail` flag 在 response flit（見 [NI Spec](03_network_interface.md) Section 8） | WARNING |
| Wormhole deadlock | 不偵測 | 本設計不模擬 deadlock recovery | N/A |

### 15.2 Credit Underflow

Credit underflow 表示 sender 在 credit=0 時仍發送 flit — 這是 flow control 邏輯 bug：

```cpp
void CreditCounter::consume(int vc) {
    assert(credits[vc] > 0 && "Credit underflow: sender sent without available credit");
    credits[vc]--;
}
```

Credit-based flow control 的正確實作保證 credit 永不 underflow。觸發此 assert 表示 credit return 邏輯或 send 條件有 bug。

### 15.3 Buffer Overflow

Buffer overflow 表示 upstream 在 downstream buffer 已滿時仍寫入 — credit 或 ready 機制應阻止此情況：

```cpp
bool FlitBuffer::push(const Flit& flit) {
    assert(!full() && "Buffer overflow: pushed to full buffer, flow control bug");
    buffer.push_back(flit);
    return true;
}
```

### 15.4 Invalid Destination

若 flit 的 `dst_id` 指向不存在的 mesh 座標，Router 無法轉發：

```cpp
Direction compute_output_port(const Flit& flit, Direction input_dir) {
    uint8_t dst_x = flit.header.dst_id & 0x0F;
    uint8_t dst_y = (flit.header.dst_id >> 4) & 0x0F;

    if (dst_x >= mesh_cols || dst_y >= mesh_rows) {
        // Invalid destination: drop flit and log
        log_warning("Flit dropped: invalid dst_id=0x%02X (dst_x=%d, dst_y=%d) "
                    "at router (%d,%d)", flit.header.dst_id, dst_x, dst_y,
                    coord.x, coord.y);
        stats.dropped_flits++;
        return Direction::NONE;  // NONE = do not forward
    }
    // ... normal routing logic
}
```

### 15.5 Wormhole Deadlock

XY routing 在無 VC 的 wormhole network 中是 **deadlock-free**（已證明）。因此本設計：

- **不實作 deadlock detection**（XY routing 數學保證不發生）
- **不實作 deadlock recovery**（無 timeout 釋放 path lock、無 packet drain/reroute）
- 若使用非 XY 的 routing algorithm（future），需重新評估 deadlock freedom

> **已知限制：** 本模擬假設 XY routing 的 deadlock freedom。若未來引入 adaptive routing 或 irregular topology，需加入 deadlock detection/avoidance 機制。

### 15.6 ECC Error 處理

Router **不檢查、不修改 ECC** — Router 為 flit 的透通轉發器。ECC 保護的 end-to-end scope：

```
Source NI (ECC generate) → Router chain (pass-through) → Destination NI (ECC check)
```

ECC 錯誤處理完全在 NI 層面，詳見 [Network Interface Specification](03_network_interface.md) Section 8。

---

## Related Documents

- [System Overview](01_overview.md) — Mesh 拓撲與系統架構
- [Network Interface Specification](03_network_interface.md) — NI 與 LOCAL port 連接、CppNI 內部 functions
- [Flit Format](04_flit.md) — Flit 結構、header 欄位定義、physical link format
- [Internal Interface](05_internal_interface.md) — Abstract Interface、Channel\<T\>、TrafficManager、Allocator、8-phase cycle 定義
- [模擬規格](08_simulation.md) — NocSystem API (6 組)、I/O Pattern、NocConfig、Replaceable Components
- [QoS Design](10_qos.md) — QoS-Aware arbitration policy、QoS Generator
