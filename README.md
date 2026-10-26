# Day 20: Health, Resources & Autoscaling (Local Kubernetes Cluster)

## Objective
Make an application self-checking, resource-aware, and dynamically scalable from 1 to 5 Pods under load using Kubernetes health probes (Startup, Readiness, Liveness), resource requests and limits, Quality of Service (QoS) classes, `metrics-server`, and Horizontal Pod Autoscaler (HPA) on a local Kubernetes cluster (Kind / Minikube).

## Architecture & Key Concepts
- **Health Probes:**
  - **Startup Probe:** Gives slow-starting applications dedicated time to initialize before readiness and liveness checks take over.
  - **Readiness Probe:** Determines if a Pod is ready to serve network traffic. Failing this probe removes the Pod from Service endpoints without restarting it.
  - **Liveness Probe:** Detects if a container has entered an unresponsive or deadlocked state. Failing this probe triggers a container restart by the kubelet.
- **Resource Management (Requests vs. Limits):**
  - **Requests:** Used by the Kubernetes scheduler to place Pods on suitable nodes and calculate CPU utilization metrics for autoscaling.
  - **Limits:** Set maximum hard caps on resource consumption. Exceeding CPU limits causes CPU throttling, whereas exceeding memory limits results in container termination (`OOMKilled`).
- **QoS Classes:**
  - **Guaranteed:** CPU and memory requests and limits are explicitly set and equal for all containers.
  - **Burstable:** Requests are specified, but limits differ or are omitted for some resources.
  - **BestEffort:** No resource requests or limits are defined.
- **Metrics Server & HPA:** `metrics-server` collects CPU/memory utilization from cluster nodes and kubelets. HPA automatically scales the target workload replica count up or down based on observed average CPU utilization relative to requested CPU.

---

## Task A: Metrics Server Installation & Verification
- Checked node metric availability using `kubectl top nodes`.
- Applied the official `metrics-server` manifest for the local cluster environment:
  ```bash
  kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
  ```
- Patched the deployment to bypass local TLS verification (`kubelet-insecure-tls` for local Kind setups):
  ```bash
  kubectl patch deployment metrics-server -n kube-system --type='json' -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'
  ```
- Confirmed successful deployment rollout and active metrics output via `kubectl top nodes`.

---

## Task B: Deploy Resource-Aware Workload
- Created the dedicated namespace: `kubectl create namespace day20-health`
- Authored and applied `deployment.yaml` with configured CPU/memory requests, limits, and all three probes:
  - **Image:** `registry.k8s.io/hpa-example`
  - **Requests:** CPU `200m`, Memory `64Mi`
  - **Limits:** CPU `500m`, Memory `128Mi`
  - **Probes:** `startupProbe` (failureThreshold 30, period 2s), `readinessProbe` (initialDelay 5s, period 5s), `livenessProbe` (initialDelay 15s, period 10s).
- Verified deployment rollout using `kubectl rollout status deploy/php-apache -n day20-health`.

---

## Task C: Resource Usage & QoS Class Inspection
- Retrieved Pod status and evaluated the assigned Quality of Service (QoS) class:
  ```bash
  kubectl get pod -n day20-health -l app=php-apache -o jsonpath='{.items[0].status.qosClass}'
  ```
- **Observed QoS Class:** `Burstable` (Reason: Resource requests and limits are set, but CPU/memory requests do not strictly equal limits).
- Inspected resource usage using `kubectl top pods -n day20-health`.

---

## Task D: Configure Horizontal Pod Autoscaler (HPA)
- Configured HPA targeting 50% CPU utilization with replica bounds between 1 and 5:
  ```bash
  kubectl autoscale deployment php-apache -n day20-health --cpu-percent=50 --min=1 --max=5
  ```
- Verified initial HPA status using `kubectl get hpa -n day20-health` and `kubectl describe hpa php-apache -n day20-health`.

---

## Task E: Load Generation & Autoscaling Verification
- Deployed a temporary `busybox` load generator pod executing an HTTP request loop:
  ```bash
  kubectl run load-generator -n day20-health --image=busybox:1.36 --restart=Never -it --rm -- /bin/sh -c "while true; do wget -q -O- http://php-apache.day20-health.svc.cluster.local; done"
  ```
- Monitored live replica scaling using `kubectl get hpa -n day20-health -w` and `kubectl get pods -n day20-health -w`.
- **Observed Scaling Behavior:** Under sustained CPU load, CPU utilization rose past 50%, triggering HPA to scale replicas up from 1 to 5. Upon terminating the load generator, CPU utilization dropped to 0% and HPA successfully scaled the deployment back down to 1 replica after the cooldown window.

---

## Task F: Safe Probe Failure Testing
- Temporarily edited `deployment.yaml` to set an invalid readiness probe path (`path: /nonexistent`) and re-applied the manifest.
- Observed Pod status change from `1/1 Running` to `0/1 Running` (`NotReady`).
- Verified event logs using `kubectl get events -n day20-health --sort-by='.lastTimestamp'`, confirming that traffic routing was revoked while the container remained running (no restart was triggered).
- Restored the readiness probe path to `/` and verified Pod readiness returned to `1/1`.

---

## Troubleshooting Guide

| Symptom | Primary Cause | Solution |
| :--- | :--- | :--- |
| `kubectl top` fails | `metrics-server` pod not running or TLS issue | Ensure `metrics-server` is deployed and patch `--kubelet-insecure-tls` for local cluster setups. |
| HPA shows `<unknown>` target | Missing `metrics-server` or missing CPU requests | Confirm CPU requests exist in deployment spec and allow 1-2 minutes for metric collection. |
| Replicas do not scale up | CPU load insufficient or `maxReplicas` limit hit | Verify load generator connectivity and inspect `kubectl describe hpa`. |
| Pod constantly restarting | Liveness probe failure or OOMKilled | Check `kubectl describe pod` for failed probes or exit code `137` (`OOMKilled`). |
| `OOMKilled` container exit | Memory usage exceeded configured limits | Increase memory limits in `deployment.yaml` to match application consumption. |

---

## Interview / Reflection Answers

### Q1. What is the difference between readiness and liveness probes?
**Answer:** A readiness probe determines whether a Pod is ready to receive network traffic; if it fails, the Pod is removed from Service endpoints without container restarts. A liveness probe detects whether a container is deadlocked or broken; if it fails repeatedly, the kubelet restarts the container.

### Q2. Why use a startup probe?
**Answer:** A startup probe disables liveness and readiness checks until the container finishes its startup sequence, preventing slow-initializing applications from being prematurely killed by liveness probes.

### Q3. What happens when a readiness probe fails?
**Answer:** The Pod's `Ready` condition is set to `False`, and Kubernetes automatically removes its IP address from matching Service endpoints. The container continues running, but receives no incoming traffic until the probe passes.

### Q4. What happens when a liveness probe repeatedly fails?
**Answer:** The kubelet intervenes by stopping the unresponsive container and creating a new container instance according to the Pod's defined `restartPolicy`.

### Q5. What is the difference between a CPU request and a CPU limit?
**Answer:** A CPU request guarantees scheduling resources and forms the baseline for HPA percent calculations. A CPU limit caps maximum CPU time allowed; exceeding the limit causes kernel-level CPU throttling rather than container termination.

### Q6. What can memory limit exceedance cause?
**Answer:** Unlike CPU throttling, memory cannot be compressed. If a container exceeds its defined memory limit, the Linux kernel Out-Of-Memory killer terminates the process with exit code `137` (`OOMKilled`).

### Q7. Why does HPA need CPU requests?
**Answer:** HPA calculates target CPU utilization as a percentage of requested CPU ($ \text{Target Utilization} = \frac{\text{Current CPU Usage}}{\text{Requested CPU}} $). Without explicit requests, HPA cannot calculate percent utilization and remains in an `<unknown>` state.

### Q8. What is the difference between HPA and manual scaling?
**Answer:** Manual scaling requires explicitly updating replica counts (`kubectl scale`), while HPA dynamically adjusts replica counts automatically within configured lower and upper bounds based on real-time metrics.

### Q9. What does `metrics-server` do?
**Answer:** `metrics-server` is a cluster-wide aggregator that collects resource usage metrics (CPU and memory) from `cAdvisor` running on individual kubelets and exposes them via the Metrics API for `kubectl top` and HPA.

### Q10. Explain QoS classes in Kubernetes.
**Answer:** 
- **Guaranteed:** Requests and limits are equal for both CPU and memory across all containers.
- **Burstable:** Requests are specified, but limits are different or omitted for some resources.
- **BestEffort:** No requests or limits are defined. QoS dictates eviction priority during node memory pressure.

### Q11. At 100% CPU usage with a target of 50% and 2 initial replicas, what is the desired count?
**Answer:** Using the HPA formula:
$$ \text{Desired Replicas} = \left\lceil \text{Current Replicas} \times \left( \frac{\text{Current Metric}}{\text{Target Metric}} \right) \right\rceil = \left\lceil 2 \times \left( \frac{100}{50} \right) \right\rceil = 4 \text{ replicas} $$

### Q12. Why might HPA not scale an application that feels slow?
**Answer:** The bottleneck may not be CPU-related (e.g., memory exhaustion, I/O latency, database bottlenecks), metrics may be missing, CPU requests may not be defined, or the HPA may have already reached its `maxReplicas` limit.

### Q13. Should a liveness probe fail whenever a database dependency is unreachable?
**Answer:** No. External dependency failures are often transient. Failing a liveness probe because of an external outage causes cascading, unnecessary container restarts across the cluster. Database health should be handled by application circuit breakers or readiness probes.

### Q14. Does autoscaling guarantee better application performance?
**Answer:** No. Autoscaling is bounded by underlying physical node capacity, network bandwidth, external dependency limits (such as database connection pools), and non-CPU resource constraints.

---

## Repository Structure
```text
day20-health-resources-autoscaling/
├── README.md
├── deployment.yaml
└── screenshot/
    ├── task-1.png
    ├── task-2-1.png
    ├── task-2-2.png
    ├── task-2-3.png
    ├── task-3-1.png
    ├── task-3-2.png
    ├── task-4.png
    ├── task-5.png
    ├── task-6-1.png
    └── task-6-2.png
```