# Numbers You Must Produce Without a Calculator (C6)

Candidate reports for SRE design rounds specifically warn: "be able to architect systems for different bandwidths / memory capacities without a calculator."

### C6.1 — Latency ladder (round, modern, and defensible)
| Operation                        | Time        |
| -------------------------------- | ----------- |
| L1 reference                     | ~1 ns       |
| Branch mispredict                | ~3 ns       |
| L2 reference                     | ~4 ns       |
| Mutex lock/unlock (uncontended)  | ~20 ns      |
| Main memory reference            | ~100 ns     |
| Syscall (post-mitigations)       | ~100–500 ns |
| Context switch                   | ~1–3 µs     |
| Read 1 MB sequentially from RAM  | ~50 µs      |
| NVMe SSD random read (4 KB)      | ~50–100 µs  |
| Round trip within a datacentre   | ~0.5 ms     |
| Read 1 MB sequentially from NVMe | ~100–200 µs |
| Disk seek (spinning)             | ~5–10 ms    |
| Round trip London ↔ US East      | ~70–80 ms   |
| Round trip London ↔ Singapore    | ~170 ms     |

### C6.2 — Capacity conversions worth memorising
- 1 day ≈ **86,400 s** ≈ 10⁵. 1 month ≈ 2.6 × 10⁶ s.
- **1 M requests/day ≈ 12 QPS.** 1 B/day ≈ 11,600 QPS. (Divide daily by 10⁵.)
- Peak ≈ **2–3× average** for consumer traffic; use 2× and say you're using 2×.
- 1 Gbps ≈ **125 MB/s**. 10 Gbps ≈ 1.25 GB/s.
- 1 KB × 1000 QPS = 1 MB/s = 8 Mbps.
- 1 TB/day ≈ **12 MB/s** sustained.
- A modern server: ~64–128 cores, 256–512 GB RAM, 10–25 Gbps NIC, a few TB NVMe at ~1 GB/s and ~500k IOPS.

### C6.3 — Estimation drill format (do this out loud, always in this order)
1. **Requirements → numbers.** Users, requests/user/day, payload size, retention.
2. **QPS**, average and peak. Read:write ratio.
3. **Bandwidth** = QPS × payload, both directions.
4. **Storage** = write QPS × payload × retention × replication factor. Then add index overhead.
5. **Memory** — working set. What fraction must be hot? Cache hit rate assumption, stated.
6. **Machine count**, derived separately from each of CPU, memory, disk, and network. **The max is your answer, and naming which resource binds is the whole point.**
7. **Sanity check.** Does the answer imply 3 machines or 30,000? Both are suspicious for different reasons.

### C6.4 — Powers of two, instantly
2¹⁰ = 1K, 2²⁰ = 1M, 2³⁰ = 1B, 2⁴⁰ = 1T. A 32-bit int caps at ~2.1 B (the classic overflow bug). A 64-bit ns timestamp overflows in ~584 years.

## One-line summary

**Numbers:** daily ÷ 10⁵ = QPS · 1 Gbps = 125 MB/s · peak = 2× · derive machines from all four resources and name the binding one.
