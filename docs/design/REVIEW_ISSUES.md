# Design Review Issues — 待解決的架構問題

> 來自 4 位不同視角 reviewer 的交叉審查（Hardware Architect, C++ Software Architect, Verification Engineer, Documentation Auditor）。
> 僅列出需要設計決策的問題，純文字/格式錯誤已在本次修正中處理完畢。

---

## CRITICAL — 需優先解決

### CR-1: FlowControlMode compile-time template vs runtime NocConfig field

**檔案**: `08_simulation.md` Section 1.1 + `05_internal_interface.md` Section 3
**問題**: `NocConfig` 把 `flow_control` 放在 runtime struct field，但 `Router_Interface<Mode>`, `Mesh<Mode>` 都用 compile-time template parameter。無法直接用 runtime 值選擇 template instantiation。
**選項**:
- (A) 移除 NocConfig 的 flow_control field，改用 `#define` 或 `constexpr`
- (B) 在 NocSystem 外層用 factory function 根據 JSON 做 if-else 雙份 instantiation + type-erased base class
- (C) 放棄 template，改用 runtime polymorphism（virtual function）

### CR-2: Credit return timing — Phase 7 生成 vs Phase 5 傳播

**檔案**: `05_internal_interface.md` Section 7.2.1
**問題**: Phase 7 生成 credit，但 Phase 5（在 Phase 7 之前）才 wire_all() 傳播。Phase 7 的 credit 無法在同一 cycle 的 Phase 5 傳播，需等到下一 cycle。這意味著 credit return latency 是 2 cycles（非文件暗示的 1 cycle）。
**選項**:
- (A) 接受 2-cycle credit latency，修正文件描述
- (B) 將 credit generation 移到 Phase 4 或更早，使同 cycle Phase 5 可傳播（1-cycle latency）
- (C) 在 Phase 7 後加一個 Phase 7.5 專門做 credit wire propagation

### CR-3: Channel ReadInputs/WriteOutputs 執行順序

**檔案**: `05_internal_interface.md` Section 8.3
**問題**: pseudocode 中 Send() 先於 ReadInputs()，可能導致 latency=1 的 channel 行為像 latency=0。
**建議**: 正確順序應為 ReadInputs → WriteOutputs → Deliver → Send

### CR-4: In-Network Reduction 潛在 deadlock

**檔案**: `02_router.md` Section 10.3
**問題**: ReductionSync 等待所有方向的 B response，可能形成 circular wait。XY routing 的 deadlock freedom 證明不涵蓋 reduction 的「等待多方向」行為。
**選項**:
- (A) 加入 timeout 機制（超時後用已到達的結果部分合併）
- (B) 形式化證明 RCR backward routing 不產生 circular dependency
- (C) 取消 in-network reduction，改為 source-side reduction

### CR-5: Phase 4 collapse — pipeline delay 參數如何生效未定義

**檔案**: `05_internal_interface.md` Section 7.2 + `08_simulation.md` Section 8.3
**問題**: RC/VA/SA/ST 全在一個 Phase 4 完成。NocConfig 有 routing_delay, vc_alloc_delay 等參數，但文件未描述這些參數如何與 8-phase model 互動。
**需要定義**: flit 在某 stage 等待 N cycles 時，input buffer 和後續 flit 如何處理。

### CR-6: "Cycle-accurate match" claim 需加限定條件

**檔案**: `08_simulation.md` Section 2.7
**問題**: 文件聲稱 C++ model 與 RTL 的 `complete_cycle` 完全相同，但前提是 pipeline delay 參數正確校準。Section 6.3 自己也承認 "單一 Phase 4 可能需多 cycle pipeline"。
**建議**: 改為 "Cycle-accurate match (requires pipeline delay parameter calibration)"

### CR-7: Golden "Last write wins" 缺乏確定性順序保證

**檔案**: `07_golden_verification.md`
**問題**: Golden capture 按 config 順序取「最後一筆」，但實際 memory 寫入順序取決於 NoC routing 延遲和 contention。
**選項**:
- (A) 禁止重疊 address 的 concurrent writes
- (B) 改為 post-simulation golden capture（simulation 完成後才 capture）
- (C) 對重疊 address 產生 warning

### CR-8: 無 deadlock detection 機制

**檔案**: `08_simulation.md` Section 6.1
**問題**: `run_until_idle()` 只有 max_cycles timeout，無法區分 deadlock 和 slow convergence。
**建議**: 在 MetricsCollector 加入 N cycles 無 forward progress 則 report deadlock warning。

---

## MEDIUM — 建議解決

### MR-1: TrafficManager 不是 template class 但引用 Mesh\<Mode\>
**檔案**: `05_internal_interface.md` Section 9.2

### MR-2: NocConfig 缺少 num_local_ports
**檔案**: `08_simulation.md` Section 1.1

### MR-3: Allocator interface 無法表達 multicast 1:N mapping
**檔案**: `05_internal_interface.md` Section 10.1

### MR-4: DPI-C bridge 缺少 signal-level API（hot-swap 無法運作）
**檔案**: `08_simulation.md` Section 5.1

### MR-5: TxnStatus 缺少 TIMEOUT state
**檔案**: `08_simulation.md` Section 4.1

### MR-6: NSU reassembly buffer 硬編碼 16，應與 max_burst_len 連動
**檔案**: `03_network_interface.md` Section 5.6

### MR-7: max_outstanding 三個文件不同預設值 (8/16/32) — 需釐清是否為不同參數
**檔案**: `08_simulation.md`, `03_network_interface.md`, `hardware_parameters_guide.md`

### MR-8: 缺少 memory access latency model
**檔案**: `06_memory_operations.md` + `08_simulation.md` Section 6.3

### MR-9: NUM_VC compile-time 常數 vs NocConfig.num_vc runtime field
**檔案**: `05_internal_interface.md` Section 2

### MR-10: Hot-swap 時 Channel pipeline 的 flush/drain 機制未定義
**檔案**: `05_internal_interface.md` Section 3.5

### MR-11: 07_golden_verification.md + 09_metrics.md 的 Python code 需改為 C++
**檔案**: `07_golden_verification.md`, `09_metrics.md`

### MR-12: 11_verification_strategy.md 缺少 multicast deadlock test 和 reset behavior test
**檔案**: `11_verification_strategy.md`

### MR-13: Width Converter 不在 verification plan 中
**檔案**: `11_verification_strategy.md` + `A3_width_converter.md`

---

## LOW — 未來可處理

- TxnHandle 用 uint32_t，長時間模擬可能 wrap → 改 uint64_t
- DPI-C void* handle 缺少 magic number validation
- std::set\<Direction\> 在 hot path 上的效能問題 → 改用 bitmask
- Throughput formula 未區分 goodput 和 raw throughput
- Flit conservation invariant 未定義
- NI injection 相對 Router-Router 多 1 cycle 延遲未納入 Channel\<T\> 模型
