\# Capacity Plan — Vision Moderation Service



\## Section 1: Latency Budget Breakdown



| Stage | Budget (ms) | Notes |

|---|---|---|

| Network in | 5 |Half of the 10 ms combined round-trip estimate |

| Auth + routing | 2 |API gateway JWT verification + upstream routing |

| Payload parse | 1 |Image decode, content-type validation |

| Feature lookup (Redis) | 8 |p95 measured at 8 ms |

| Pre-processing | 5 |Resize, normalize (CPU-side, from 15 ms combined) |

| Model inference | 22 |T4 GPU |

| Post-processing | 10 |Score thresholding, label mapping (from 15 ms combined) |

| Serialization | 2 | JSON encode response body|

| Network out | 5 |Half of the 10 ms combined round-trip estimate |

| Headroom | 190 |190 ms remaining after all stages;

absorbs tail latency, GC pauses, and jitter |

| \*\*Total\*\* | \*\*250\*\* | |



Sum check: 5 + 2 + 1 + 8 + 5 + 22 + 10 + 2 + 5 + 190 = 250 ms ✓



\## Section 2: CPU vs GPU Decision



\*\*Choice:\*\* GPU



\*\*Justification:\*\*

GPU gives 3x faster inference, provides better latency headroom, needs fewer replicas to manage. And batching will bring the cost down significantly. Both options exceed the $4,000 budget — batching will reduce replica count and bring cost down.

Prices are sourced from AWS on-demand pricing (approximate)



| | CPU (c5.xlarge) | GPU (g4dn.xlarge) |

|---|---|---|

| Inference time | 75 ms | 22 ms |

| Total request time | 113 ms | 60 ms |

| Per-replica throughput | 9 RPS | 18 RPS |

| Replicas needed (spike) | 73 | 37 |

| Cost per replica/month | $120 | $380 |

| \*\*Total monthly cost\*\* | \*\*$8760\*\* | \*\*$14060\*\* |



\## Section 3: Replica Sizing



| Case | Target RPS | Per-Replica Throughput | Replicas (raw) | Replicas (+30% headroom) | Monthly Cost |

|---|---|---|---|---|---|

| Sustained | 300 | 18 RPS | 17 | 23 | $8740 |

| Spike | 500 | 18 RPS | 28 | 37 | $14060 |



\*\*Spike strategy:\*\* Autoscaling - spikes last only 5 minutes. Spinning up extra replicas temporarily is more cost-efficient than running 37 replicas 24/7.



\## Section 4: Batching Decision



\*\*Dynamic batching enabled:\*\* Yes, enabled



\*\*Settings:\*\*

| Setting | Value | Justification |

|---|---|---|

| Max batch size | 8 |\~5 requests arrive per 10 ms at 500 RPS peak; 8 absorbs small bursts |

| Max wait window | 10 ms |safe within 190 ms headroom; adds only 10 ms to total request time |



\*\*Justification:\*\*

Latency budget is 250 ms, total stages require 60 ms and headroom is 190 ms. Adding 10 ms wait still keeps total at 70 ms — well under 250 ms. That is why batching is safe.



Each replica now handles 8 requests at once instead of 1 and throughput per replica increases enabling batching. Running process needs fewer replicas, it brings monthly cost closer to $4,000 budget.

&#x20;







