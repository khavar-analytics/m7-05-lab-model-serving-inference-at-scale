# Load Test Plan — Vision Moderation Service

## 1. Tool Choice

**Tool: k6**

k6 scripts in JavaScript, ships a built-in HTTP client with per-request latency histograms, and outputs native Prometheus metrics — making it trivial to compare test results against the SLO thresholds defined in `slos.yaml` without any post-processing. It supports custom virtual-user lifecycle scripts, which are needed to model the 90/10 sync/batch traffic split with realistic payload distributions.

---

## 2. Test Phases

| Phase | Duration | Target RPS | Purpose |
|---|---|---|---|
| Warmup | 5 min | 30 RPS (10%) | Allow JIT compilation, Redis connection pools, and GPU caches to stabilize before measurement begins |
| Ramp | 10 min | 30 → 300 RPS | Linear ramp; surface resource saturation points before hitting peak |
| Sustained peak | 30 min | 300 RPS | Primary measurement window; all pass/fail criteria evaluated here |
| Spike | 5 min | 300 → 500 → 300 RPS | 2-min ramp up, 1-min at 500, 2-min ramp down; validates autoscale response and spike SLA |
| Ramp down | 5 min | 300 → 0 RPS | Graceful drain; check for connection leaks on teardown |
| Soak | 60 min | 200 RPS (67%) | Run at 2/3 of peak for an hour; catches memory leaks, thread-pool exhaustion, file descriptor leaks |

**Total test duration: ~115 minutes**

---

## 3. Traffic Shape

**Endpoint split:**
- 90% of virtual users target `POST /v1/moderate` (synchronous)
- 10% of virtual users target `POST /v1/moderate/batch` (batch aggregation)

**Payload distribution (synchronous):**

| Payload class | Share | Image size |
|---|---|---|
| Small (thumbnails) | 40% | 50–200 KB |
| Medium (standard uploads) | 45% | 200 KB–2 MB |
| Large (high-res) | 15% | 2–5 MB |

Payloads drawn from a corpus of pre-generated test images (JPEG, PNG mix). No synthetic all-zero images — they produce unrealistically fast inference.

**Payload distribution (batch):**
- 10–50 images per batch call; batch size sampled from a uniform distribution
- Batch payload size: 5–50 MB per request

**Concurrency model:**
- Synchronous VUs: 60 virtual users with a 0.2 s think time between requests (yields ~300 RPS at peak across all VUs)
- Batch VUs: 6 virtual users with a 2 s think time between requests (yields ~30 RPS batch calls)
- HTTP/1.1 keep-alive enabled; connection pool size = 10 per VU

---

## 4. Pass/Fail Criteria

All thresholds apply to the **sustained peak phase** (30 min at 300 RPS). Test fails if any threshold is breached.

| Metric | Threshold | Endpoint |
|---|---|---|
| p95 latency | ≤ 250 ms | Synchronous |
| p99 latency | ≤ 400 ms | Synchronous |
| p95 latency | ≤ 5,000 ms | Batch (informational only; no hard SLA defined yet) |
| Error rate (5xx + timeout) | ≤ 0.3% | Synchronous |
| Error rate (5xx + timeout) | ≤ 1.0% | Batch |
| Sustained throughput | ≥ 295 RPS (98% of target) | Synchronous |
| Max observed p95 during spike | ≤ 500 ms | Synchronous (partner SLA refund threshold) |
| GPU utilization (per replica) | < 80% | All replicas during sustained peak |

The spike phase has a **separate gate:** p95 latency must return to ≤ 250 ms within 60 seconds of the spike subsiding (validates autoscale cooldown).

---

## 5. Bottleneck Checklist

Inspect the following on each replica every 30 seconds during the test:

**Compute:**
- [ ] CPU utilization % (pre/post-processing threads; target < 70%)
- [ ] GPU utilization % (NVIDIA SMI `gpu_util`; target < 80% at sustained peak)
- [ ] GPU memory used / total (flag if > 85%; indicates batch size needs tuning)

**Memory:**
- [ ] Replica RSS memory (flag growth > 5% over soak phase — indicates leak)
- [ ] JVM / runtime heap (if applicable): GC pause duration and frequency

**Downstream dependencies:**
- [ ] Redis command latency p95 (target ≤ 12 ms as budgeted; alert if > 20 ms)
- [ ] Redis connection pool active / waiting (flag if waiting queue > 0 sustained)
- [ ] Redis error rate (any TIMEOUT or CONNECTION_REFUSED counts as critical)

**Network:**
- [ ] Ingress bytes/s per replica (compare to expected payload distribution)
- [ ] Egress bytes/s per replica
- [ ] Load balancer active connections (flag sustained > 80% of max)
- [ ] TCP retransmit rate on replica NICs

**Application-level:**
- [ ] Request queue depth inside each replica (ONNX server pending queue; target ≤ 5)
- [ ] Model inference latency histogram (separate from end-to-end; isolates GPU vs. overhead)
- [ ] Batch endpoint fill rate (average realized batch size vs. max_batch_size=16)
