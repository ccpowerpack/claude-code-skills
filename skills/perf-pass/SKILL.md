---
name: perf-pass
description: Measurement-first performance work — profile, rank hotspots by evidence, optimize only what the numbers justify, prove the win. Use when the user says something is slow, asks to optimize, reduce latency/memory, or investigate performance.
---

# Perf Pass

The iron law: **no optimization without a measurement, no merged optimization without a before/after.** Intuition about performance is wrong often enough that acting on it unmeasured is negative-expected-value.

## Phase 1 — Define the target (gate)

Get concrete or stop:

- What operation is slow? (endpoint, query, job, render, build)
- Slow compared to what? Current number → target number.
- Which environment matters? Dev-machine timing on a 100-row table says nothing about prod with 10M rows.

If the user only says "it's slow", instrument first and bring back numbers before proposing anything.

## Phase 2 — Measure

Choose the coarsest tool that answers the question, then descend:

1. **End-to-end timing** of the reproduction (wall clock, N≥5 runs, note variance).
2. **Layer split**: where does the time go — DB, network, CPU, I/O wait? (APM traces, simple timers at boundaries, `EXPLAIN ANALYZE` for queries.)
3. **Profiler** on the hot layer — pick from the matrix below.
4. Record a baseline artifact (flamegraph, timing table) you can diff after the fix.

### Profiler matrix

| Stack | CPU profile | Allocation profile | Off-CPU / wait | One-liner |
|---|---|---|---|---|
| Python | `py-spy` (sampling, no code change) | `memray` | `py-spy record --idle` includes blocked threads | `py-spy record -o flame.svg --pid <pid> --duration 30` |
| Node | `--cpu-prof` (V8), `0x` for flamegraphs | `node --heap-prof`, DevTools allocation sampling | `clinic bubbleprof` (async wait), event-loop lag via `perf_hooks.monitorEventLoopDelay` | `node --cpu-prof --cpu-prof-dir=./prof server.js` then load in `chrome://inspect` |
| Go | built-in pprof | `pprof` heap/allocs endpoints | `/debug/pprof/block` + `/debug/pprof/mutex`; `fgprof` for wall-clock (CPU+wait combined) | `go tool pprof -http=:8081 http://localhost:6060/debug/pprof/profile?seconds=30` |
| JVM | async-profiler (no safepoint bias) | async-profiler `-e alloc` | async-profiler `-e wall`; JFR for lock contention | `asprof -d 30 -e cpu -f cpu.html <pid>` |
| Rust | `cargo flamegraph` (perf-based) | `heaptrack`, `dhat-rs` | `perf record --call-graph dwarf` + `offcputime-bpfcc` | `cargo flamegraph --bin server -- --bench-input big.json` |
| Postgres | `EXPLAIN (ANALYZE, BUFFERS)` per query | n/a (watch `temp_blks_written` for spills) | `pg_stat_activity.wait_event`, lock waits in `pg_locks` | `SELECT query, mean_exec_time, calls FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;` |
| Browser | DevTools Performance panel (record → flame chart) | Memory panel → allocation sampling | Performance panel long-task lanes; network waterfall for wait | `npx lighthouse https://app.example.com --only-categories=performance` |

Rule of thumb for the third column: if wall time ≫ CPU time in your end-to-end measurement, a CPU profiler will show you nothing — go straight to the off-CPU/wait tool.

## Phase 3 — Rank and pick

List hotspots as: `<location> — <% of total> — <hypothesis> — <expected win>`.

Attack in order of `% of total`, not ease. A 3% hotspot optimized to zero buys 3%. Amdahl's law is the budget.

### Usual suspects — symptom → cause → 60-second confirmation

| Symptom pattern | Likely cause | 60-second probe |
|---|---|---|
| Latency grows linearly with result-set size; DB layer dominated by many fast identical queries | N+1 query | Log SQL for one request, count statements: `grep -c 'SELECT .* FROM order_items' app.log` — 1 request producing dozens of near-identical queries confirms it |
| One query slow at prod scale, fine in dev | Missing index | `EXPLAIN (ANALYZE, BUFFERS) <query>` — `Seq Scan` on a large table with a selective filter confirms it |
| Wall time ≫ CPU time; throughput low but CPU idle | Sync I/O in a loop | `strace -c -p <pid>` for 10s — syscall time dominated by `read`/`write`/`connect` with high call counts; or compare `time` real vs user+sys |
| Runtime quadruples when input doubles | Accidental O(n²) — `in`/`.includes` on a list, nested scan | Time at two input sizes (n, 2n); >3x growth confirms. Then grep the hot function for membership tests on lists/arrays |
| Latency ∝ network RTT count, not payload size; fast locally, slow across regions | Chatty RPC / sequential round-trips | Count round-trips per operation in a trace, or `tcpdump` + count request/response pairs for one logical call |
| CPU profile hot in `json.dumps`/`JSON.stringify`/pickle/proto encode | Serialization overhead — encoding more or more often than needed | Flamegraph: serialization frames >15% of CPU confirms; check payload size vs what the consumer reads |
| Sawtooth latency; periodic pauses; p99 ≫ p50 with smooth p50 | GC pressure from allocation churn | GC logs / runtime counters: Go `GODEBUG=gctrace=1`, JVM `-Xlog:gc`, Node `--trace-gc` — pause frequency correlating with latency spikes confirms |

Check these before anything clever. Most "mysterious" slowness is on this table.

### When the profile is flat

No frame over ~5%? Then there is no hotspot to attack — the cost is architectural, paid per-item in a loop that shouldn't be per-item. Look for: a fixed overhead (query, RPC, lock acquisition, serializer setup) multiplied by N where batching would pay it once; or work that shouldn't happen at all (recomputing an invariant, loading columns nobody reads). A flat profile is an argument for restructuring one level up, never for micro-optimizing twenty 2% frames.

## Phase 4 — Optimize

- One change at a time; re-measure after each. Two stacked changes with one measurement = you don't know which worked.
- Preserve behavior: run the test suite after each change.
- Complexity is a cost: a 5% win that doubles the code's difficulty is usually a loss. Say so when it applies.

## Phase 5 — Prove

```
BASELINE: <number, conditions>
AFTER:    <number, same conditions>
DELTA:    <x% / xms>
```

Same environment, same data shape, same N of runs. If the win doesn't reproduce, revert and say so — a reverted non-win is a better outcome than merged noise.

### Measurement pitfalls checklist — walk it before trusting any number

- [ ] **Cold vs warm cache**: first run pays cache fills (page cache, DB buffer pool, JIT, connection pools). Either discard warm-up runs on both sides, or explicitly measure cold-start if that's the target. Never compare a cold baseline to a warm after.
- [ ] **Variance and N**: ≥5 runs; report p50 and p95, not the mean of a noisy handful. If baseline stddev overlaps the claimed delta, you measured noise.
- [ ] **Dev vs prod data shape**: row counts, cardinality, and cache-hit rates change which plan/branch executes. Postgres will happily Seq Scan a 500-row dev table and be right. Test against a prod-sized snapshot or a synthetic dataset matched in size and distribution.
- [ ] **Coordinated omission**: a closed-loop load generator (send → wait → send) slows down when the server does, hiding the worst latencies. Use a fixed-rate open-loop tool (`wrk2 -R`, `hey -q`, vegeta) when measuring p95+.
- [ ] **Load generator co-located with the server**: they compete for CPU; run the generator elsewhere or at least pin them apart (`taskset`).
- [ ] **Profiler overhead in the numbers**: measure with the profiler *off*; profile in a separate run.

## Worked example — endpoint p95 hunt

Target: `GET /api/orders?user_id=` p95 is 1400ms in prod; SLO is 300ms.

**Layer split.** Add boundary timers (middleware + DB wrapper), replay realistic traffic at fixed rate:

```
$ hey -z 60s -q 20 -c 10 "https://staging.example.com/api/orders?user_id=8841"
  p50: 620ms   p95: 1380ms
```

Per-request layer breakdown from the timers (median of the slow bucket):

```
total=1350ms  db=1190ms  serialize=85ms  auth=25ms  other=50ms
```

DB is 88% of the budget. Nothing else is worth touching yet (Amdahl).

**Descend into the DB layer.** One request, SQL log on:

```
$ grep 'req-7f3a' app.log | grep -c 'SELECT'
49
$ grep 'req-7f3a' app.log | grep 'SELECT' | sort | uniq -c | sort -rn | head -2
  48 SELECT * FROM order_items WHERE order_id = $1
   1 SELECT * FROM orders WHERE user_id = $1 ORDER BY created_at DESC
```

Classic N+1: one query for orders, one per order for items. Each item query is ~24ms (connection acquisition + RTT dominates, the query itself is indexed and sub-millisecond) × 48 = ~1150ms. Matches the 1190ms db figure — hypothesis confirmed, no speculative second cause needed.

**One fix.** Batch the child load (`selectinload(Order.items)` in SQLAlchemy — emits one `WHERE order_id IN (...)` query). Test suite passes; response payload byte-identical on a sampled request (`diff <(curl -s .../orders?user_id=8841) before.json`).

**Prove.** Same tool, same rate, same duration, same staging dataset:

```
$ hey -z 60s -q 20 -c 10 "https://staging.example.com/api/orders?user_id=8841"
  p50: 95ms   p95: 240ms
```

```
BASELINE: p95 1380ms (hey, 60s @ 20 rps, staging w/ prod-sized snapshot)
AFTER:    p95 240ms  (identical conditions)
DELTA:    -83% / -1140ms — under the 300ms SLO
```

Serialization (85ms) is now the largest remaining slice; noted for a future pass, not touched in this one.

## Output contract

Finish with: the target definition, the baseline artifact, the ranked hotspot list, the BASELINE/AFTER/DELTA block for each merged change, and any hotspot you identified but deliberately left (with the number that justified leaving it).

## Anti-patterns (hard no)

- Micro-optimizing (caching, memoizing, rewriting loops) code that no profile identified.
- Benchmarking against cold caches on one side and warm on the other.
- Claiming a win from a single noisy run.
- Adding a cache without an invalidation story.
- Reporting mean latency for a system with SLOs on percentiles.
- Fixing two suspects in one commit and attributing the win to your favorite.
