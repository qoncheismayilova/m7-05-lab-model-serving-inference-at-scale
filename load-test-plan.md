# Load Test Plan

## Tool Choice

### k6

k6 is selected because it provides precise control over request rates, supports realistic traffic patterns, and generates detailed latency and error-rate metrics. It is widely used for API load testing and integrates well with modern monitoring platforms.

---

## Test Phases

### 1. Warmup Phase

* Duration: 5 minutes
* Traffic: 0 → 100 RPS

Purpose:

* Load model into memory
* Warm Redis cache
* Stabilize infrastructure

### 2. Ramp-Up Phase

* Duration: 10 minutes
* Traffic: 100 → 300 RPS

Purpose:

* Observe system behavior under increasing load
* Detect scaling issues

### 3. Sustained Peak Phase

* Duration: 20 minutes
* Traffic: 300 RPS

Purpose:

* Validate normal peak operating conditions

### 4. Spike Test Phase

* Duration: 5 minutes
* Traffic: 500 RPS

Purpose:

* Verify handling of short traffic bursts

### 5. Soak Test Phase

* Duration: 60 minutes
* Traffic: 200 RPS

Purpose:

* Detect memory leaks
* Detect resource exhaustion
* Validate long-term stability

---

## Traffic Shape

### Endpoint Distribution

* 90% synchronous endpoint
* 10% batch endpoint

### Payload Size Distribution

* Small (1–5 KB): 70%
* Medium (5–20 KB): 25%
* Large (20–50 KB): 5%

### Concurrency Model

Virtual users will be configured to maintain target RPS throughout the test. Requests will be distributed evenly across all active replicas.

---

## Pass/Fail Criteria

The load test passes only if all conditions below are satisfied:

| Metric             | Threshold       |
| ------------------ | --------------- |
| p95 latency        | ≤ 250 ms        |
| p99 latency        | ≤ 500 ms        |
| Error rate         | ≤ 0.5%          |
| Availability       | ≥ 99.9%         |
| Redis p95 latency  | ≤ 15 ms         |
| GPU utilization    | ≤ 90% sustained |
| CPU utilization    | ≤ 85% sustained |
| Memory utilization | ≤ 80% sustained |

Any threshold violation results in a failed test.

---

## Bottleneck Checklist

During testing, the following metrics will be monitored on every replica:

### Compute Resources

* CPU utilization (%)
* Memory utilization (%)
* GPU utilization (%)
* GPU memory usage

### Application Metrics

* Request latency (p50, p95, p99)
* Request throughput (RPS)
* Error rate
* Average batch size

### Redis Metrics

* Redis p95 latency
* Connection pool utilization
* Request failures

### Infrastructure Metrics

* Pod restarts
* Out-of-memory (OOM) events
* Network throughput
* Queue length
* Autoscaling events

The service is considered production-ready only if all pass criteria are met and no significant bottlenecks are observed during sustained or spike traffic tests.
