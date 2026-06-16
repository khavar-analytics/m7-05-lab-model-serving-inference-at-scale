\# Load Test Plan — Vision Moderation Service



\## Section 1: Tool Choice



\*\*Tool:\*\* k6

\*\*Justification:\*\* it handles multiple endpoints with different ratios easily and outputs p95/p99 metrics automatically. This tool provides built-in support for ramping traffic up and down.



\## Section 2: Test Phases



| Phase | Duration | Target RPS | Notes |

|---|---|---|---|

| Warmup | 2 minutes | 30 |Send very low traffic first to warm up the service  |

| Ramp | 5 minutes | 300 |Gradually increase traffic from 0 to full sustained load.  |

| Sustained Peak | 20 minutes| 300 |Hold at full normal load long enough to confirm stability |

| Spike | 5 minutes | 500 |Suddenly jump to spike load |

| Soak | 60 minutes | 300 | Run at normal load for a long time to catch slow memory leaks|



\## Section 3: Traffic Shape



| Property | Value | Notes |

|---|---|---|

| Sync vs Batch ratio | 90% vs 10% | |

| Payload size | 100-500 KB | Small 100 KB, Medium 300 KB, Large 500 KB|

| Concurrency model | 18 VUs | 300 RPS × 0.060s response time = 18 concurrent virtual users |



\## Section 4: Pass/Fail Criteria



| Metric | Threshold | Action if breached |

|---|---|---|

| p95 latency | 250 ms | fail the test |

| p99 latency | 500 ms | fail the test |

| Error rate | 0.3% | fail the test |

| Availability | 0.997 | fail the test |



\## Section 5: Bottleneck Checklist



\- \[ ] CPU usage per replica — watch for sustained >80%

\- \[ ] GPU utilization per replica — watch for sustained >85%

\- \[ ] GPU memory — watch for >80% VRAM usage

\- \[ ] Redis latency — must stay under 8 ms during peak

\- \[ ] Redis connections — watch for connection pool exhaustion

\- \[ ] Model inference time — must stay at \~22 ms per batch

\- \[ ] Request queue depth — watch for growing backlog

\- \[ ] Error rate per endpoint — sync and batch measured separately

