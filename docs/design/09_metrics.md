# 效能指標

本文件定義效能指標收集機制。

---

## 1. 效能指標

效能指標透過各元件的 Stats 類別收集。

### 1.1 元件層級統計

| 元件 | Stats 類別 | 主要指標 |
|------|-----------|---------|
| Buffer | `BufferStats` | 讀寫次數、平均/最大佔用率 |
| Router | `RouterStats` | flit 數量、延遲、阻塞事件 |
| NI | `NIStats` | AXI 交易數、flit 數量 |
| Mesh | `MeshStats` | 總週期、總 flit 數 |
| Memory | `MemoryStats` | 讀寫次數、bytes 傳輸量 |
| AXI Master | `AXIMasterStats` | 交易數、延遲、完成率 |

### 1.2 BufferStats

```python
@dataclass
class BufferStats:
    total_writes: int = 0
    total_reads: int = 0
    max_occupancy: int = 0
    total_occupancy_samples: int = 0
    cumulative_occupancy: int = 0

    @property
    def avg_occupancy(self) -> float:
        """平均 buffer 佔用率。"""
```

### 1.3 RouterStats

```python
@dataclass
class RouterStats:
    flits_received: int = 0
    flits_forwarded: int = 0
    flits_dropped: int = 0
    total_latency: int = 0
    arbitration_cycles: int = 0
    buffer_full_events: int = 0
    port_utilization: Dict[Direction, int]
```

### 1.4 AXIMasterStats

```python
@dataclass
class AXIMasterStats:
    # Write
    total_transactions: int = 0
    total_bytes: int = 0
    completed_transactions: int = 0
    completed_bytes: int = 0
    total_latency: int = 0

    # Read
    read_transactions: int = 0
    read_bytes_requested: int = 0
    read_completed: int = 0
    read_bytes_received: int = 0
    read_latency: int = 0
    read_matches: int = 0
```

---

## 2. MetricsCollector

`MetricsCollector` 提供統一的效能指標收集介面，支援時間序列快照與 Monitor-Based 延遲追蹤。

### 2.1 基本使用

```python
from src.core import V1System, NoCSystem
from src.visualization import MetricsCollector

# Host-to-NoC
system = V1System(mesh_cols=5, mesh_rows=4)
collector = MetricsCollector(system, capture_interval=1)

# 模擬迴圈中捕捉快照
for cycle in range(1000):
    system.process_cycle()
    collector.capture()

# 取得時間序列資料
cycles, occupancies = collector.get_total_buffer_occupancy_over_time()
cycles, throughputs = collector.get_throughput_over_time()
```

### 2.2 Monitor-Based 延遲追蹤

使用 `record_injection` / `record_ejection` 追蹤端到端延遲，無需修改 Core 程式碼：

```python
# Host-to-NoC: 追蹤 AXI 交易延遲
axi_id = 1
system.submit_write(addr, data, axi_id)
collector.record_injection(axi_id, cycle)  # 記錄注入時間

# ... 模擬執行 ...

resp = system.nsu.get_b_response()
if resp:
    collector.record_ejection(resp.bid, cycle)  # 記錄彈出時間，自動計算延遲

# NoC-to-NoC: 追蹤節點傳輸延遲
for node_id in system.node_controllers.keys():
    collector.record_injection(node_id, cycle=0)  # 所有節點同時開始

# ... 模擬執行 ...

if controller.is_transfer_complete:
    collector.record_ejection(node_id, cycles)  # 節點完成時記錄
```

### 2.3 統計方法

| 方法 | 回傳值 | 說明 |
|------|--------|------|
| `get_throughput(start_cycle)` | `float` | 整體吞吐量 (bytes/cycle) |
| `get_latency_stats()` | `dict` | 延遲統計 {min, max, avg, std, samples} |
| `get_buffer_stats(total_capacity)` | `dict` | Buffer 統計 {peak, avg, utilization} |
| `get_all_latencies()` | `List[int]` | 所有延遲樣本 |
| `get_throughput_over_time(window)` | `(cycles, values)` | 時間序列吞吐量 |
| `get_buffer_occupancy_over_time()` | `(cycles, matrix)` | 時間序列 buffer 佔用 |

### 2.4 計算公式

**Throughput**:
```
throughput = bytes_transferred / total_cycles
```

其中 `bytes_transferred` 以實際 AXI data 計量。每個 W/R flit 攜帶 AXI_DATA_WIDTH = 256 bits = 32 bytes 的有效資料。Flit 總寬度 FLIT_WIDTH = 408 bits，但 throughput 以有效資料量（payload data）計算，不包含 header 與 padding。

**Link Throughput（理論上限）：**
```
link_throughput = FLIT_WIDTH / cycle = 408 bits/cycle = 51 bytes/cycle (per direction)
```

**Latency**:
```
latency = ejection_cycle - injection_cycle
```

**Buffer Utilization**:
```
utilization = avg_occupancy / total_capacity
total_capacity = num_routers × buffer_depth × num_ports  (預設 400)
```

### 2.5 完整範例

```python
from src.visualization import MetricsCollector

collector = MetricsCollector(system, capture_interval=1)

# 模擬結束後取得統計
throughput = collector.get_throughput()
latency_stats = collector.get_latency_stats()
buffer_stats = collector.get_buffer_stats(total_capacity=400)

print(f"Throughput: {throughput:.2f} B/cycle")
print(f"Latency: min={latency_stats['min']}, max={latency_stats['max']}, "
      f"avg={latency_stats['avg']:.1f}, samples={latency_stats['samples']}")
print(f"Buffer: peak={buffer_stats['peak']}, avg={buffer_stats['avg']:.1f}, "
      f"util={buffer_stats['utilization']:.2%}")
```

---

## 3. 元件層級統計收集

統計資料透過各元件的 `.stats` 屬性存取：

```cpp
// 執行模擬後...
noc::NocSystem system(config);
system.run(1000);

// Mesh 統計
auto& mesh_stats = system.mesh().stats();

// Router 統計
for (auto& [pos, router] : system.mesh().routers()) {
    printf("Router (%d,%d): %lu flits\n", pos.x, pos.y,
           router->stats().flits_forwarded);
}

// NI 統計
for (auto& [pos, ni] : system.mesh().nis()) {
    printf("NI (%d,%d): %lu req flits\n", pos.x, pos.y,
           ni->stats().req_flits_sent);
}
```

---

## 相關文件

- [模擬參數](08_simulation.md)
- [Flit 格式](04_flit.md)
