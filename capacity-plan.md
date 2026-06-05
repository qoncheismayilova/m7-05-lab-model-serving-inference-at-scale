# Capacity Plan

## 1. Latency Budget Breakdown

The synchronous endpoint has a p95 latency target of **250 ms**. The latency budget is allocated as follows:

| Stage                  | Budget (ms) | Notes                               |
| ---------------------- | ----------- | ----------------------------------- |
| Network In             | 5           | Request ingress                     |
| Auth + Routing         | 2           | Authentication and routing          |
| Payload Parsing        | 5           | JSON validation and parsing         |
| Feature Lookup (Redis) | 8           | Given p95 latency                   |
| Pre-processing         | 10          | Image normalization and preparation |
| Model Inference        | 22          | NVIDIA T4 GPU inference             |
| Post-processing        | 8           | Thresholding and result formatting  |
| Serialization          | 5           | JSON response generation            |
| Network Out            | 5           | Response transmission               |
| Headroom               | 180         | Buffer for spikes and tail latency  |
| **Total**              | **250**     |                                     |

The latency budget balances exactly to 250 ms, leaving significant headroom for traffic bursts and infrastructure variability.

---

## 2. CPU vs GPU Decision

### Decision: GPU-Based Deployment

The service will use NVIDIA T4 GPUs for inference. The model requires approximately 75 ms median inference time on a CPU core but only 22 ms on a T4 GPU. This significantly reduces end-to-end latency and increases throughput per replica.

Assuming approximately 45 RPS per GPU replica, fewer replicas are required to meet the target load compared with CPU-based deployment. GPU acceleration also enables efficient dynamic batching, which improves utilization during traffic spikes.

Approximate cloud pricing:

* CPU instance (AWS c6i.large): ~$0.10/hour
* GPU instance (AWS g4dn.xlarge with T4): ~$0.53/hour

Although GPU instances are more expensive individually, the higher throughput reduces the total number of required replicas and improves latency consistency.

---

## 3. Replica Sizing

Assuming a conservative throughput of 45 requests per second per GPU replica:

### Sustained Traffic (300 RPS)

300 / 45 = 6.67 replicas

Rounded up: 7 replicas

With 30% headroom:

7 × 1.3 = 9.1 → 10 replicas

### Spike Traffic (500 RPS)

500 / 45 = 11.1 replicas

Rounded up: 12 replicas

With 30% headroom:

12 × 1.3 = 15.6 → 16 replicas

| Scenario       | Target RPS | Per-Replica Throughput | Required Replicas | With 30% Headroom |
| -------------- | ---------- | ---------------------- | ----------------- | ----------------- |
| Sustained Load | 300        | 45 RPS                 | 7                 | 10                |
| Peak Spike     | 500        | 45 RPS                 | 12                | 16                |

### Scaling Strategy

The service will maintain 10 warm replicas for normal traffic and use Kubernetes Horizontal Pod Autoscaling to scale up to 16 replicas during 5-minute traffic spikes.

---

## 4. Batching Decision

### Dynamic Batching: Enabled

Configuration:

* Maximum Batch Size: 8
* Maximum Wait Time: 10 ms

Dynamic batching improves GPU utilization by combining multiple requests into a single inference execution. This increases throughput and reduces infrastructure cost while maintaining efficient hardware usage.

The maximum waiting window is limited to 10 ms, which represents only a small portion of the 250 ms latency budget. Therefore, batching provides additional throughput capacity without significantly affecting end-to-end latency. This approach helps absorb short traffic bursts while keeping p95 latency within the service objective.
