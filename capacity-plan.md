# Capacity Plan — Vision Moderation Service

## 1. Latency Budget Breakdown

Target: p95 ≤ 250 ms, synchronous endpoint, GPU-backed replicas.

| Stage | Budget (ms) | Notes |
|---|---|---|
| Network in | 5 | Half of the 10 ms combined round-trip estimate |
| Auth + routing | 5 | API gateway JWT verification + upstream routing |
| Payload parse | 8 | Image decode, content-type validation |
| Feature lookup (Redis) | 12 | p95 measured at 8 ms; +4 ms buffer for tail |
| Pre-processing | 12 | Resize, normalize (CPU-side, from 15 ms combined) |
| Model inference | 80 | T4 GPU; median 22 ms; 3–4× for p95 under queue wait |
| Post-processing | 8 | Score thresholding, label mapping (from 15 ms combined) |
| Serialization | 5 | JSON encode response body |
| Network out | 5 | Half of the 10 ms combined round-trip estimate |
| **Headroom** | **110** | 44% of budget; absorbs GC pauses, retries, jitter spikes |
| **Total** | **250** | |

**Sum check:** 5 + 5 + 8 + 12 + 12 + 80 + 8 + 5 + 5 + 110 = **250 ms** ✓

The 110 ms headroom is intentionally large because the 80 ms inference budget already bakes in a 3–4× p50→p95 multiplier for the GPU. Under a well-tuned replica the realized p95 will sit closer to 140–160 ms total, leaving ~90 ms of observable headroom in production traces.

---

## 2. CPU vs GPU Decision

**Decision: T4 GPU (g4dn.xlarge on AWS)**

| Metric | CPU (c5.2xlarge, 8 vCPU) | GPU (g4dn.xlarge, 1× T4) |
|---|---|---|
| Median inference | 75 ms | 22 ms |
| Estimated p95 inference | ~150 ms | ~80 ms |
| Remaining budget after inference | ~100 ms | ~170 ms |
| Per-replica throughput target | ~55 req/s | ~80 req/s |
| On-demand price | ~$0.34/hr | ~$0.526/hr |
| Monthly cost per replica | ~$245 | ~$379 |

**Justification:** The CPU p95 inference of ~150 ms (2× median, typical tail under contention) leaves only 100 ms for all other stages combined. That headroom is fragile — a Redis hiccup or a GC pause would breach the 250 ms SLO. The T4 GPU's p95 of ~80 ms leaves 170 ms of slack, making the target robustly achievable. The GPU is ~55% more expensive per replica but we need fewer replicas to serve the same throughput (see section 3), so total fleet cost is lower with GPU than CPU. Pricing sourced from AWS EC2 on-demand pricing page (us-east-1, June 2025); within 30% of spot/committed rates.

---

## 3. Replica Sizing

**Per-replica throughput (GPU):** 80 req/s  
**Max target utilization (30% headroom):** 70% → effective capacity 56 req/s per replica

| Scenario | Target RPS | Raw replicas needed | Replicas with 30% headroom | Monthly cost |
|---|---|---|---|---|
| Sustained peak | 300 | 3.75 → 4 | ceil(300/56) = **6** | 6 × $379 = **$2,274** |
| 5-min spike | 500 | 6.25 → 7 | ceil(500/56) = **9** | 9 × $379 = **$3,411** |

**Both scenarios are within the $4,000/month compute budget.**

**Spike strategy: autoscaling with a warm floor.** GPU instances take 2–3 minutes to become ready (CUDA runtime + model load). A 5-minute spike window is too narrow to rely on cold autoscaling alone. The configuration is:
- Minimum replica floor: **7** (overprovisions slightly above sustained need)
- Autoscale ceiling: **9** on CPU utilization or queue-depth signal
- Horizontal Pod Autoscaler target: 65% GPU utilization

This gives ~90 seconds to scale from 7→9 before the spike fully loads, covering the gap with the warm floor.

---

## 4. Batching Decision

**Synchronous endpoint (90% of traffic): no dynamic batching.**  
The synchronous path serves latency-sensitive calls with a 250 ms p95 budget. Any batching max-wait window (even 10 ms) directly adds to every request's tail latency and is visible to partners. With 80 req/s per GPU replica, the average inter-arrival time is ~12.5 ms, so a 10 ms wait window would yield average batch sizes of only 1–2 — an insignificant throughput gain that does not justify the latency cost.

**Batch endpoint (10% of traffic): enable dynamic batching.** Partner aggregation calls are already acknowledged as higher-latency operations with no sub-250 ms SLA. Configure the ONNX Runtime serving layer with `max_batch_size: 16` and `max_wait_ms: 50`. At the batch endpoint's 30 req/s arrival rate across 6 replicas (5 req/s per replica), a 50 ms window yields an expected batch size of ~2.5, providing a meaningful GPU utilization boost without violating any latency contract. If partner SLA for the batch path is defined later, the wait window should be tuned to keep p95 batch latency under that threshold.
