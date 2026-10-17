# Low-Latency Order Book: Comprehensive Benchmark & Profiling Report

| Student Name | Roll No |
| :--- | :--- |
| Sourabh Srivastva | 24BCS10157 |
| Daksh Shah | 24BCS10092 |

---

## 1. Executive Summary

This report documents the performance evaluation and microarchitectural analysis of three resting order book implementations for a single financial instrument, adhering strictly to the capstone requirements:
- **Workload**: 1,000,000 operations (approx. 70% Add, 20% Cancel, 10% BBO query, depth query every 100 operations).
- **Random Seed**: Fixed repeatable seed `42` (`std::mt19937`).
- **Repetitions**: 5 independent runs per implementation to determine the median result.
- **Environment**: Linux (Ubuntu 26.04 LTS on WSL2 / x86_64, Linux 6.18 kernel), compiled with GCC 15.2.0 (`-O2 -g -Wall -Wextra -Wpedantic -std=c++20`).
- **Profiling**: Linux `perf stat` across 9 hardware performance counters and `perf record` / `perf report` callgraph hotspot analysis.

---

## 2. Correctness Verification

Prior to benchmarking, correctness was verified by running identical operation streams across all three implementations. Every operation verified:
- **BBO agreement**: Exact match of `has_bid`, `has_ask`, `bid_price`, `bid_qty`, `ask_price`, and `ask_qty`.
- **Depth level agreement**: Exact match of prices and quantities across top 10 bid and ask price levels every 100 operations.
- **Cancellation agreement**: Boolean return status matches identically on valid and invalid cancellation attempts.

### Verification Terminal Output (100,000 Operations, Seed 42):
```bash
./build/bench --verify --ops 100000 --seed 42
```
```text
Running correctness test across 100000 operations (seed 42)...

Implementation A: PASS
Implementation B: PASS
Implementation C: PASS

All implementations produce identical results.
```
*Process exit code: `0` (clean pass)*

Unit test validation (`./build/verify --verify --ops 100000`) also completed with zero errors and exit code `0`.

---

## 3. Benchmark Results (1,000,000 Operations)

The benchmark was executed using the shared harness [`bench.cpp`](bench.cpp) with 1,000,000 operations and 5 runs each (seed 42).

### 3.1 Primary Benchmark Results (1,000,000 Operations, Seed 42, 5 Runs)

| Metric | Implementation A (`std::map`) | Implementation B (`priority_queue` + Lazy) | Implementation C (`std::list`) |
| :--- | :---: | :---: | :---: |
| **Workload Operations** | **1,000,000** | **1,000,000** | **1,000,000** |
| **Status** | **Completed** | **Completed** | **UNAVAILABLE (Timed Out / Infeasible)** |
| **Median Total Time** | **241.67 ms** | **182.57 ms** | **UNAVAILABLE** (Ran >2m, Est. ~2.0 hrs/run) |
| **Average ns / `add_order`** | **186.0 ns** | **114.5 ns** | **UNAVAILABLE** |
| **Average ns / `cancel_order`** | **432.9 ns** | **323.1 ns** | **UNAVAILABLE** |
| **Average ns / `get_bbo`** | **21.4 ns** | **20.3 ns** | **UNAVAILABLE** |
| **Median Page Faults** | **1** | **1,402** | **UNAVAILABLE** |
| **Max RSS (KB)** | **65,452 KB** | **65,716 KB** | **UNAVAILABLE** |
| **Final Best Bid** | `99.99 x 2722` | `99.99 x 2722` | **UNAVAILABLE** |
| **Final Best Ask** | `100.01 x 2525` | `100.01 x 2525` | **UNAVAILABLE** |
| **Final Checksum** | **4,495,793,605** | **4,495,793,605** | **UNAVAILABLE** |

*Both Implementation A and B produce identical final BBOs (`Bid: 99.99 x 2722 | Ask: 100.01 x 2525`) and identical checksums (`4495793605`), proving consistent state across all 1,000,000 operations.*

> [!WARNING]
> ### 3.2 Implementation C: 1,000,000-Operation Run Attempt & Infeasibility Report
> In accordance with the assignment requirements, we attempted to benchmark Implementation C on the required 1,000,000-operation workload using:
> ```bash
> /usr/bin/time -v ./build/bench --impl C --ops 1000000 --runs 1 --seed 42
> ```
> **Observations & Why it Could Not Complete**:
> 1. The process ran at 100% CPU utilization for over 2 minutes without completing even an initial fraction of the first run.
> 2. Live hardware counter sampling via `perf stat -p <pid> -- sleep 5` revealed:
>    - Core frequency: ~4.7 GHz (~23.37 billion CPU cycles in 5 seconds).
>    - Instructions executed: only 366 million, yielding an IPC of **0.015** (over 98% of all CPU cycles are stalled on DRAM memory access).
>    - Traversal rate: ~12 million linked-list node hops per second.
> 3. **Mathematical Infeasibility**: A 1,000,000-operation workload contains 700,000 `add_order` operations. With resting orders accumulating to ~500,000, the average list length during adds is ~250,000 nodes. Finding the insertion position requires an average of $\frac{1}{2} \times 250,000 = 125,000$ pointer hops per add. Total list node hops across the run equals $700,000 \times 125,000 \approx \mathbf{87.5\text{ billion hops}}$. At 12 million hops/sec, a single run takes $\approx \mathbf{7,290\text{ seconds (~2.02 hours)}}$, and 5 runs would require $\approx \mathbf{10.1\text{ hours}}$.
> 4. Therefore, completing the required 5 runs of 1,000,000 operations for Implementation C is computationally impractical. Per the assignment instructions, this result is explicitly reported as **UNAVAILABLE**, rather than substituted or fabricated. **Consequently, the project cannot be marked fully compliant with the 1,000,000-operation requirement for all three implementations.**

### 3.3 Implementation C: Auxiliary Measured Workload (100,000 Operations, 5 Runs)
To characterize Implementation C's empirical behavior without substituting it into the primary 1M benchmark, it was independently evaluated on a smaller workload of 100,000 operations:
- **Workload**: 100,000 operations, 5 runs, seed `42` (`./build/bench --impl C --ops 100000 --runs 5 --seed 42`).
- **Median Total Time**: **4,483.51 ms** (~4.48 s)
- **Average ns / `add_order`**: **63,865.6 ns**
- **Average ns / `cancel_order`**: **194.8 ns**
- **Average ns / `get_bbo`**: **64.3 ns**
- **Median Page Faults**: 1
- **Max RSS**: 13,676 KB
- **Checksum**: 404,245,593
- **Quadratic Scaling Evidence**: A 200,000-operation test ran in **37,104.91 ms** (~37.1 s) with add latency jumping to **264,760.6 ns**—an 8.3x runtime increase for a 2x workload increase, confirming the $O(N^2)$ algorithmic explosion.

<details>
<summary>View detailed per-run terminal logs for Implementation A & B (1,000,000 ops)</summary>

#### Implementation A (1,000,000 ops, seed 42):
```text
Run 1/5: 272.62 ms | add: 225.1 ns | cancel: 447.1 ns | BBO: 21.7 ns | faults: 7392
Run 2/5: 241.67 ms | add: 186.0 ns | cancel: 432.9 ns | BBO: 22.0 ns | faults: 1392
Run 3/5: 235.04 ms | add: 181.8 ns | cancel: 414.2 ns | BBO: 21.4 ns | faults: 0
Run 4/5: 252.29 ms | add: 198.0 ns | cancel: 443.8 ns | BBO: 21.3 ns | faults: 1
Run 5/5: 233.66 ms | add: 179.9 ns | cancel: 414.4 ns | BBO: 21.1 ns | faults: 0

MEDIAN RESULTS (5 runs):
  Total execution time : 241.67 ms
  Average ns / add     : 186.0 ns
  Average ns / cancel  : 432.9 ns
  Average ns / get_bbo : 21.4 ns
  Page faults (median) : 1
  Max RSS              : 65452 KB
```

#### Implementation B (1,000,000 ops, seed 42):
```text
Run 1/5: 195.74 ms | add: 140.9 ns | cancel: 308.3 ns | BBO: 20.2 ns | faults: 7435
Run 2/5: 180.31 ms | add: 114.5 ns | cancel: 323.1 ns | BBO: 20.4 ns | faults: 1402
Run 3/5: 175.53 ms | add: 113.4 ns | cancel: 305.7 ns | BBO: 20.2 ns | faults: 2724
Run 4/5: 201.79 ms | add: 126.1 ns | cancel: 380.9 ns | BBO: 20.5 ns | faults: 0
Run 5/5: 182.57 ms | add: 107.9 ns | cancel: 354.0 ns | BBO: 20.3 ns | faults: 0

MEDIAN RESULTS (5 runs):
  Total execution time : 182.57 ms
  Average ns / add     : 114.5 ns
  Average ns / cancel  : 323.1 ns
  Average ns / get_bbo : 20.3 ns
  Page faults (median) : 1402
  Max RSS              : 65716 KB
```

</details>

---

## 4. Hardware Counter Profiling (`perf stat`)

Hardware counter data was collected using Linux `perf stat -r 5` targeting user-space events across the exact 9 counters specified in `assignment.md`:
```bash
perf stat -r 5 \
  -e task-clock,cycles,instructions,branches,branch-misses,cache-misses,L1-dcache-load-misses,dTLB-load-misses,page-faults \
  -- ./build/bench --impl <A|B|C> --ops <N> --runs 1 --seed 42
```

### Hardware Counter Results Table

| Counter | Implementation A (1M ops) | Implementation B (1M ops) | Implementation C (100k ops) |
| :--- | :---: | :---: | :---: |
| **`task-clock`** | 306.26 ms | 269.36 ms | 5,803.28 ms |
| **`cycles`** | 1,286,794,063 | 1,130,166,826 | 27,311,647,039 |
| **`instructions`** | 925,770,251 | 940,498,653 | 2,782,460,504 |
| **`IPC` (Instructions / Cycle)** | **0.72** | **0.83** | **0.10** |
| **`branches`** | 174,119,532 | 163,378,275 | 907,516,107 |
| **`branch-misses`** | 8,423,303 | 1,747,128 | 455,007 |
| **Branch Miss Rate (%)** | **4.84%** | **1.07%** | **0.05%** |
| **`cache-misses`** | 5,207,221 | 4,692,474 | 727,661,741 |
| **`L1-dcache-load-misses`** | 17,772,418 | 10,642,232 | 992,892,492 |
| **`dTLB-load-misses`** | 527,856 | 520,503 | 1,108,836 |
| **`page-faults`** | 17,430 | 17,473 | 2,667 |
| **`perf` Elapsed Time** | 310.93 ms | 274.64 ms | 5,808.60 ms |

### Counter Availability & Environment Notes:
- **All 9 requested counters were successfully collected** on our Linux WSL2 test system using the virtualized Intel PMU.
- Because 9 concurrent hardware events exceed the 4–8 physical hardware performance registers available on consumer x86_64 CPUs, the Linux kernel multiplexed events with ~85% counter residency. Perf scaled all counter values across all 5 repetitions.

---

## 5. Hotspot Identification (`perf record` & `perf report`)

Callgraph profiles (`perf record -g`) were collected to inspect execution hotspots for each implementation:

### 5.1 Implementation A Hotspots
1. **`OrderBookA::add_order` (24.57% self, 26.65% children)**:
   - Tree navigation: `std::map::operator[]` calls `std::_Rb_tree::_M_lower_bound` (10.49% bids, 10.33% asks).
   - Inlined tree node walking: 7.53% in `_S_right` and `_S_left` pointer dereferences while balancing binary search nodes.
   - Hash table insertion: 4.11% in `m_orders.operator[]` for node allocation (`operator new`).
2. **`OrderBookA::cancel_order` (24.57% self)**:
   - Hash lookup: `m_orders.find(id)`.
   - Tree lookup & node erasure: `m_bids.find(price)` followed by `m_bids.erase(map_it)` which triggers tree rebalancing rotations.
3. **`__vdso_clock_gettime` (12.94%)**: Nanosecond timing capture for per-operation latency metrics.

### 5.2 Implementation B Hotspots
1. **`OrderBookB::add_order` (27.26%)**:
   - Hash table bucket indexing: 14.61% in `m_orders[id]` insertion, dominated by `_M_rehash` (8.09%) as the active order table expands.
   - Price level lookup: 12.83% in `m_priceOrderMap.find(tick_price)`.
2. **`OrderBookB::cancel_order` (19.26%)**:
   - `m_orders.find(id)` hash lookup and node erase.
   - **Heap operations (`pop`/`push`) account for only ~3.6% combined**: because of lazy deletion, cancellations never touch the binary heap.

### 5.3 Implementation C Hotspots
1. **`OrderBookC::add_order` (99.46% of all execution cycles)**:
   - The loop: `while (it != m_bids.end() && (*it)->price > tick_price) ++it;`
   - Split: ~49.98% on bids traversal and ~49.48% on asks traversal.
   - Every node hop dereferences a heap-allocated `std::_List_node` pointer, stalling the CPU on main memory access.

---

## 6. Comprehensive Performance Analysis (Assignment Section 11)

### 11.1 Which implementation is fastest?
- **Fastest for `add_order`**: **Implementation B (114.5 ns)** vs 186.0 ns for A and 63,865+ ns for C. In realistic trading distributions where prices cluster around 10,000 discrete ticks, incoming orders frequently land on existing price levels. Implementation B finds existing levels in $O(1)$ time via `m_priceOrderMap`, avoiding tree walks.
- **Fastest for `cancel_order`**: **Implementation B (323.1 ns)** vs 432.9 ns for A. Implementation B employs lazy deletion: cancellation simply subtracts quantity via a direct pointer and erases the ID from the hash map, requiring zero heap heapify operations. Implementation A must perform an $O(\log P)$ tree search and red-black tree rebalancing whenever a level is emptied.
- **Fastest for `get_bbo`**: **Implementation B (20.3 ns)** and **Implementation A (21.4 ns)** perform almost identically. Implementation A reads the root-adjacent `begin()` pointer in $O(1)$ time. Implementation B reads `top()` of the priority queue, which is $O(1)$ when the top order has non-zero quantity.

### 11.2 Branch Prediction Analysis
- **Most Branch Misses**: **Implementation A** had **8,423,303 branch misses (4.84% miss rate)**, compared to **1.07% for B** and **0.05% for C**.
- **Explanation**: In `std::map`, searching the red-black tree requires comparing the incoming tick price to each tree node (`tick < node->price`). Because order prices are randomly distributed across the 5000 valid ticks for each side, the direction of branch traversal (left child vs. right child) is virtually random (close to 50/50 probability). Modern branch predictors rely on pattern recognition and history tables; random comparisons cause frequent mispredictions and pipeline flushes.
- In contrast, Implementation C's branch miss rate is remarkably low (0.05%) because its insertion loop tests `(*it)->price > tick_price`. In a long linked list, this loop condition evaluates to `true` hundreds or thousands of times consecutively before terminating once, allowing the hardware branch predictor to predict "branch taken" with near 100% accuracy.

### 11.3 Cache Behavior Analysis
- **Most Cache Misses**: **Implementation C** generated an astonishing **727,661,741 cache misses** and **992,892,492 L1-dcache misses** on only 100,000 operations. By comparison, Implementation A and B recorded only 5.2M and 4.7M cache misses across 1,000,000 operations.
- **Explanation**: 
  - Implementation C allocates every single order as a separate heap object (`new Order`) wrapped in a doubly-linked `std::_List_node`. These nodes are scattered across the heap with poor spatial locality.
  - Adding an order requires sequentially dereferencing `node->next` pointers. The CPU cannot effectively prefetch these pointers because their memory addresses are not contiguous. The CPU pipeline stalls waiting for DRAM lines to arrive, driving its IPC down to a catastrophic **0.10**.
  - Implementation B stores price level pointers inside `std::vector` contiguous memory buffers within the binary heap. Binary heap traversals access adjacent memory blocks, resulting in **10.64M L1-dcache misses**—nearly 40% fewer than Implementation A's 17.77M misses.

### 11.4 Page Fault Analysis
- Implementation A generated 17,430 page faults and Implementation B generated 17,473 page faults during the 1,000,000-operation benchmark.
- **Explanation**: With 70% adds and 20% cancels, the book retains ~500,000 resting orders at steady state. Storing 500,000 orders in `std::unordered_map` requires tens of megabytes of bucket arrays and node allocations (~65.5 MB peak RSS). As the process dynamically allocates memory from the runtime heap, the Linux virtual memory manager allocates physical frames, generating minor page faults. Implementation C used fewer page faults at 100k ops (2,667 faults, 13.6 MB RSS) simply because the resident set was smaller.

### 11.5 Production Choice: Which Implementation Would You Deploy?
If restricted to deploying only one of these three implementations to a real trading engine, we would choose **Implementation A (with architectural extensions)**.
- **Why not Implementation B?** Although Implementation B demonstrated the highest raw average throughput (182 ms vs 241 ms), its **tail latency distribution** is dangerous for HFT systems. Lazy deletion defers garbage collection to query operations. When an aggressive market move sweeps multiple adjacent price levels, a subsequent `get_bbo()` or depth query will encounter a burst of stale nodes, forcing repeated heap pops and node deallocations right on the latency-sensitive quote path.
- **Why Implementation A?** Implementation A offers **deterministic, bounded worst-case latency**. Its red-black tree maintains an invariant where price levels are always pre-aggregated and strictly ordered. `get_bbo()` is guaranteed $O(1)$ and `get_bids(10)` is strictly $O(K)$, never stalling for deferred maintenance.
- **Why not Implementation C?** Implementation C is disqualified due to its $O(N^2)$ algorithmic complexity and catastrophic cache behavior.

### 11.6 What Would You Try Next?
To achieve sub-microsecond, production-grade HFT performance:
1. **Direct Price-Indexed Array (`std::array<Level, 10000>`)**:
   Since the price universe is strictly bounded to 10,000 discrete ticks ($50.00 to $150.00), dynamic search structures can be replaced with a contiguous 10,000-element array. Level lookup is a single array index: `size_t idx = tick - 5000;`. Adding and cancelling order quantities becomes a single $O(1)$ cache-line memory read/write.
2. **Bitmaps for Best-Price Tracking**:
   Maintain active price levels in two 160-word 64-bit integer bitmaps (`uint64_t bitmap[160]`). Best bid and ask discovery requires finding the first set bit using a single hardware assembly instruction (`__builtin_clzll` / `__builtin_ctzll`), executing in **1 clock cycle** with zero branch mispredictions.
3. **Pre-allocated Fixed Object Pool**:
   Replace `std::unordered_map` and dynamic heap allocations (`new`/`delete`) with a contiguous pre-allocated array of orders. This eliminates runtime `malloc` overhead and eliminates minor page faults during trading hours.

---

## 7. Reproducibility Commands

To build and reproduce all results on Linux:

```bash
# 1. Clean build in Release mode (-O2 -g)
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j"$(nproc)"

# 2. Correctness verification (checks 100,000 operations across A, B, and C)
./build/bench --verify --ops 100000 --seed 42
./build/verify --verify --ops 100000

# 3. Benchmark runs (5 iterations each)
./build/bench --impl A --ops 1000000 --runs 5 --seed 42
./build/bench --impl B --ops 1000000 --runs 5 --seed 42
./build/bench --impl C --ops 100000 --runs 5 --seed 42

# 4. Hardware counter profiling
perf stat -r 5 \
  -e task-clock,cycles,instructions,branches,branch-misses,cache-misses,L1-dcache-load-misses,dTLB-load-misses,page-faults \
  -- ./build/bench --impl A --ops 1000000 --runs 1 --seed 42

perf stat -r 5 \
  -e task-clock,cycles,instructions,branches,branch-misses,cache-misses,L1-dcache-load-misses,dTLB-load-misses,page-faults \
  -- ./build/bench --impl B --ops 1000000 --runs 1 --seed 42

perf stat -r 5 \
  -e task-clock,cycles,instructions,branches,branch-misses,cache-misses,L1-dcache-load-misses,dTLB-load-misses,page-faults \
  -- ./build/bench --impl C --ops 100000 --runs 1 --seed 42

# 5. Hotspot callgraph profiling
perf record -g -o /var/tmp/perf_a.data -- ./build/bench --impl A --ops 1000000 --runs 1 --seed 42
perf report --stdio -i /var/tmp/perf_a.data

perf record -g -o /var/tmp/perf_b.data -- ./build/bench --impl B --ops 1000000 --runs 1 --seed 42
perf report --stdio -i /var/tmp/perf_b.data

perf record -g -o /var/tmp/perf_c.data -- ./build/bench --impl C --ops 100000 --runs 1 --seed 42
perf report --stdio -i /var/tmp/perf_c.data
```

---

## 8. Submission Compliance & Known Limitations

| Capstone Requirement | Status | Notes |
| :--- | :---: | :--- |
| Single-threaded resting order book (no matching) | **COMPLIANT** | All 3 implementations adhere strictly to resting order book semantics. |
| Three distinct data-structure designs | **COMPLIANT** | Implementation A (`std::map`), B (`priority_queue` + lazy deletion), C (`std::list` + direct iterators). |
| Shared benchmark program with CLI flags | **COMPLIANT** | One unified binary `bench` supports `--impl <A\|B\|C>`, `--verify`, `--ops`, `--runs`, `--seed`. |
| Automated correctness verification | **COMPLIANT** | `bench --verify` checks exact requested ops, validates state agreement, and exits nonzero on failure. |
| 1,000,000-operation benchmark for A and B | **COMPLIANT** | Fully executed (5 runs each, median recorded, identical BBO and checksum). |
| 1,000,000-operation benchmark for C | **UNMET / KNOWN LIMITATION** | Implementation C cannot finish 1M ops due to $O(N^2)$ algorithmic scaling (~2.0 hrs/run, ~10.1 hrs for 5 runs). Explicitly reported as **UNAVAILABLE**. |
| Linux `perf stat` across 9 hardware counters | **COMPLIANT** | All 9 requested counters measured and analyzed. |
| Linux `perf record` & `perf report` hotspot profiling | **COMPLIANT** | Hotspots identified and correlated directly with code-level data structures. |
| Analysis report answering Section 11 questions | **COMPLIANT** | Comprehensive discussion on runtime, branch prediction, cache behavior, page faults, and production choices. |

