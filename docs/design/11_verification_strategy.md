# 驗證策略

本文件定義 NoC C++ Model 的驗證方法論，確保 spec 與 implementation 一致。涵蓋單元測試、整合測試、golden verification、performance validation 與 RTL co-simulation。

---

## 1. 驗證層次

```
                    ┌─────────────────────────────┐
                    │  RTL Co-simulation (DPI-C)  │  ← 最高層：C++ vs RTL 一致性
                    ├─────────────────────────────┤
                    │  System Integration Tests    │  ← 完整系統 end-to-end
                    ├─────────────────────────────┤
                    │  Scenario Integration Tests  │  ← 多元件交互場景
                    ├─────────────────────────────┤
                    │  Component Unit Tests        │  ← 單一元件隔離測試
                    └─────────────────────────────┘
```

---

## 2. 單元測試策略

每個核心元件需獨立測試，使用 GoogleTest 框架。

### 2.1 元件測試矩陣

| 元件 | 測試重點 | 最低 Test Case 數 |
|------|---------|:----------------:|
| `FlitBuffer` | push/pop/full/empty/peek, depth boundary | 8 |
| `Flit` (Header) | pack/unpack round-trip, bit-field accuracy | 6 |
| `XYRouter` | routing decision for all 5 output ports, edge cases | 12 |
| `PathLock` | lock/unlock FSM, state transitions | 6 |
| `QoSAwareArbiter` | QoS priority, Round-Robin fairness, multi-contender | 10 |
| `Crossbar` | N×N switching, no-uturn, no-self-loop | 8 |
| `CreditCounter` | init/consume/release, underflow assert | 6 |
| `NMU` | flit packing (AW/W/AR), RoB allocate/release, injection buffer | 12 |
| `NSU` | flit unpacking, W reassembly, B/R response packing | 10 |
| `ECC` | generate/check round-trip, single-bit correction, double-bit detection | 6 |
| `ReductionSync/Arbiter` | in-network B reduction, partial/all-ok/all-fail | 6 |
| `AddressTranslation` | 64b/32b mode, node_id extraction | 4 |

### 2.2 單元測試規範

```cpp
// 範例：FlitBuffer 測試
TEST(FlitBufferTest, PushPopRoundTrip) {
    FlitBuffer buf(4);  // depth=4
    Flit f = make_test_flit(/*dst_id=*/0x11, /*axi_ch=*/0);

    EXPECT_TRUE(buf.empty());
    buf.push(f);
    EXPECT_FALSE(buf.empty());
    EXPECT_EQ(buf.size(), 1);

    Flit out = buf.pop();
    EXPECT_EQ(out.header.dst_id, 0x11);
    EXPECT_TRUE(buf.empty());
}

TEST(FlitBufferTest, OverflowAssert) {
    FlitBuffer buf(2);
    buf.push(make_test_flit());
    buf.push(make_test_flit());
    EXPECT_DEATH(buf.push(make_test_flit()), "Buffer overflow");
}
```

---

## 3. 整合測試矩陣

### 3.1 關鍵場景

| ID | 場景 | 涉及元件 | 驗證重點 |
|----|------|---------|---------|
| IT-01 | 單次 32B Write (1 hop) | NMU→Router→NSU | 基本 end-to-end data integrity |
| IT-02 | 單次 64B Write (3 hop) | NMU→Router×3→NSU | Multi-hop routing correctness |
| IT-03 | 256B Burst Write (awlen=7) | NMU→Mesh→NSU | Wormhole lock/unlock, multi-flit |
| IT-04 | 單次 Read (AR→R) | NMU→Mesh→NSU→Mesh→NMU | 雙向 req/rsp 路徑 |
| IT-05 | Burst Read (arlen=15) | NMU→Mesh→NSU→Mesh→NMU | Multi-flit R response |
| IT-06 | 交叉流量衝突 | 2 NMU→同一 Router port | QoS arbitration, backpressure |
| IT-07 | Wormhole stall 傳播 | 長 burst + buffer 滿 | Backpressure propagation |
| IT-08 | Multicast Write | NMU→4 NSU | B response aggregation |
| IT-09 | Read-after-Write | Write 完成後 Read 同地址 | Data consistency |
| IT-10 | Max outstanding | 32 pending transactions | RoB full backpressure |
| IT-11 | Multi-ID reorder | 不同 axi_id 的 response OoO | RoB per-ID ordering |
| IT-12 | Credit exhaustion | 持續注入至 credit=0 | Flow control correctness |
| IT-13 | ECC single-bit error | 注入 1-bit error | ECC correction |
| IT-14 | ECC double-bit error | 注入 2-bit error | ECC detection, SLVERR |
| IT-15 | Multicast timeout | 一個 NSU 不回應 | Timeout mechanism |

### 3.2 整合測試範例

```cpp
// IT-01: 單次 32B Write, 1 hop
TEST(IntegrationTest, SingleWriteOneHop) {
    NocConfig cfg;
    cfg.mesh_cols = 3;
    cfg.mesh_rows = 2;
    NocSystem system(cfg);

    // Write 32B to node (1,0) addr 0x1000
    std::vector<uint8_t> data(32, 0xAB);
    uint64_t addr = (0x01ULL << 32) | 0x1000;  // node_id=0x01 → (1,0)
    ASSERT_TRUE(system.submit_write(addr, data, 0));

    int cycles = system.run_until_complete(10000);
    EXPECT_GT(cycles, 0);

    // Verify data arrived at node (1,0) local memory
    auto report = system.verify();
    EXPECT_TRUE(report.passed);
}
```

---

## 4. Golden Verification

與 [Golden 驗證機制](07_golden_verification.md) 交叉引用。

### 4.1 驗證點

| 驗證點 | 位置 | 比較對象 |
|--------|------|---------|
| Write data integrity | NSU local memory | NMU 注入的原始資料 |
| Read data integrity | NMU 收到的 R response | NSU local memory 的實際資料 |
| Response ordering | NMU RoB release 順序 | Per-AXI-ID FIFO 順序 |
| Multicast completeness | Aggregator | 所有 target node 的 B response |
| ECC pass-through | Destination NI | Source NI 產生的 ECC 值 |

### 4.2 GoldenManager 驗證流程

```
1. Transaction 提交時：GoldenManager 記錄 expected outcome
2. Transaction 完成時：比對 actual vs expected
3. 模擬結束時：生成 VerificationReport
```

```cpp
struct VerificationReport {
    bool     passed;                    // 整體結果
    int      total_checks;             // 總驗證點數
    int      passed_checks;            // 通過數
    int      failed_checks;            // 失敗數
    std::vector<FailureDetail> failures; // 失敗詳情
};
```

---

## 5. Performance Validation

與 [效能指標](09_metrics.md) 交叉引用。

### 5.1 效能基準測試

| 測試 | 量測指標 | 預期範圍 |
|------|---------|---------|
| Single flit latency (1 hop) | cycle count | 3-5 cycles |
| Single flit latency (max hop) | cycle count | mesh_cols + mesh_rows + overhead |
| Burst throughput (no contention) | flits/cycle | ≈1.0 flit/cycle/port |
| Sustained throughput (uniform random) | flits/cycle/node | >0.5 |
| Buffer utilization (high load) | avg occupancy / depth | >0.7 |

### 5.2 Parameter Sweep

```cpp
// 效能參數掃描範例
for (int buf_depth : {2, 4, 8, 16}) {
    for (int mesh_size : {3, 5, 8}) {
        NocConfig cfg;
        cfg.mesh_cols = mesh_size;
        cfg.mesh_rows = mesh_size;
        cfg.input_buffer_depth = buf_depth;
        NocSystem system(cfg);

        // Run benchmark traffic
        auto metrics = run_benchmark(system, uniform_random_traffic);
        record_result(buf_depth, mesh_size, metrics);
    }
}
```

---

## 6. RTL Co-simulation

與 [模擬規格](08_simulation.md) Section 6 交叉引用。

### 6.1 Co-sim 架構

```
┌─────────────────┐       ┌─────────────────┐
│  SystemVerilog  │       │   C++ Model     │
│  RTL Testbench  │◄─────►│  (via DPI-C)    │
│                 │       │                 │
│  RTL Router ×N  │       │  NocSystem      │
│  RTL NI ×N     │       │                 │
└────────┬────────┘       └────────┬────────┘
         │                         │
         ▼                         ▼
    RTL Response              C++ Response
         │                         │
         └────────► Comparator ◄───┘
                        │
                   PASS / FAIL
```

### 6.2 測試策略

| 層級 | 測試方式 | 比較粒度 |
|------|---------|---------|
| Flit-level | 每 cycle 比對 Router output flit | Bit-exact |
| Transaction-level | 比對完整 AXI transaction 結果 | Data + response code |
| Latency-level | 比對 per-transaction latency | Cycle-exact |

### 6.3 Co-sim Test Plan

1. **Smoke test**: Single write/read, C++ vs RTL bit-exact match
2. **Stress test**: 100 random transactions, all responses match
3. **Corner cases**: Buffer full, credit exhaustion, multicast timeout
4. **Performance**: Latency/throughput within 5% tolerance

---

## 7. Coverage Targets

### 7.1 行為覆蓋率

| 類別 | 目標 | 量測方式 |
|------|------|---------|
| Code line coverage | ≥ 80% | gcov / llvm-cov |
| Branch coverage | ≥ 70% | gcov / llvm-cov |
| FSM state coverage | 100% | Custom: 所有 PathLock 狀態轉移 |
| AXI channel coverage | 100% | 所有 5 channel 至少 1 transaction |
| Routing direction coverage | 100% | 所有 5 output port direction 至少 1 routing |
| Error path coverage | 100% | ECC error, credit boundary, buffer boundary |

### 7.2 功能覆蓋矩陣

| Feature | 相關測試 ID | 狀態 |
|---------|-----------|------|
| XY Routing (all directions) | IT-01, IT-02 | — |
| Wormhole lock/unlock | IT-03, IT-07 | — |
| QoS arbitration | IT-06 | — |
| Credit flow control | IT-12 | — |
| Multicast aggregation | IT-08, IT-15 | — |
| RoB reorder | IT-10, IT-11 | — |
| ECC generate/check | IT-13, IT-14 | — |
| Backpressure propagation | IT-07, IT-12 | — |
| NMU injection buffer | IT-10 | — |
| NSU W-flit reassembly | IT-03 | — |

---

## 相關文件

- [Golden 驗證機制](07_golden_verification.md) — Golden data comparison
- [效能指標](09_metrics.md) — MetricsCollector, stats classes
- [模擬規格](08_simulation.md) — DPI-C bridge, system API
- [Router 規格](02_router.md) — Router behavior under test
- [Network Interface 規格](03_network_interface.md) — NI behavior under test
