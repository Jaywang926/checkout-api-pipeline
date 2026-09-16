# Runbook: checkout-api OOMKilled Incident

## Purpose

This runbook provides a step-by-step procedure for diagnosing and resolving `OOMKilled` incidents affecting the `checkout-api` Kubernetes workload.

An `OOMKilled` event means the container exceeded its configured memory limit and Kubernetes terminated the container. The cause may be an incorrectly configured memory limit, an application memory leak, or an unexpected workload increase.

---

## 1. Symptoms

Common symptoms include:

* `checkout-api` pods repeatedly restarting
* Increasing `RESTARTS` count
* Intermittent request failures
* Pods entering `CrashLoopBackOff`
* Container status showing `OOMKilled`
* Memory usage approaching or exceeding the configured limit

---

# Diagnosis

## 2. Check Pod Status

Start by checking the pods in the `checkout-api` namespace:

```bash
kubectl get pods -n checkout-api
```

Example:

```text
NAME                            READY   STATUS    RESTARTS      AGE
checkout-api-7d8f9c6b7f-x2abc   1/1     Running   8 (2m ago)   15m
```

A rapidly increasing `RESTARTS` count is an indication that the container may be repeatedly crashing.

---

## 3. Inspect the Pod

Describe the affected pod:

```bash
kubectl describe pod <pod-name> -n checkout-api
```

Look at the **Last State** section.

An OOMKilled container will typically show:

```text
Last State:     Terminated
  Reason:       OOMKilled
  Exit Code:    137
```

`Reason: OOMKilled` confirms that the container was terminated because it exceeded its memory limit.

---

## 4. Check Recent Events

Review Kubernetes events:

```bash
kubectl get events -n checkout-api --sort-by='.lastTimestamp'
```

Look for events involving:

* `OOMKilled`
* container restarts
* failed probes
* scheduling failures
* deployment changes

You can also inspect events for the specific pod:

```bash
kubectl describe pod <pod-name> -n checkout-api
```

The **Events** section at the bottom is particularly useful.

---

## 5. Check the Current Resource Configuration

Inspect the deployment:

```bash
kubectl get deployment checkout-api -n checkout-api -o yaml
```

To quickly find the configured resources:

```bash
kubectl get deployment checkout-api -n checkout-api \
  -o jsonpath='{.spec.template.spec.containers[0].resources}'
```

Example:

```text
{"limits":{"cpu":"500m","memory":"8Mi"},"requests":{"cpu":"100m","memory":"8Mi"}}
```

If the memory limit is unexpectedly low, compare it against the application's normal memory requirements.

For this incident, the problematic value was:

```text
memory limit: 8Mi
```

The previous known-good configuration was:

```text
memory limit: 128Mi
```

---

## 6. Check Current Memory Usage

If the Kubernetes Metrics Server is available:

```bash
kubectl top pod -n checkout-api
```

For a specific pod:

```bash
kubectl top pod <pod-name> -n checkout-api
```

Example:

```text
NAME                            CPU(cores)   MEMORY(bytes)
checkout-api-7d8f9c6b7f-x2abc   15m          72Mi
```

Compare the observed memory usage against the configured limit.

For example:

```text
Observed usage: 72Mi
Memory limit:    8Mi
```

This indicates that the container requires substantially more memory than its configured limit.

---

## 7. Check Container Logs

Retrieve the current container logs:

```bash
kubectl logs <pod-name> -n checkout-api
```

Because the container may have restarted, also inspect the logs from the previous container instance:

```bash
kubectl logs <pod-name> -n checkout-api --previous
```

Look for:

* memory-related errors
* application crashes
* unusually large requests
* unexpected workload behavior
* errors immediately before termination

An absence of an application-level memory error does not rule out `OOMKilled`. Kubernetes may terminate the container before the application can log anything useful.

---

## 8. Determine the Likely Cause

Use the following decision process.

### Case A: Memory limit is clearly too low

Example:

```text
Memory limit: 8Mi
Normal usage: 40–70Mi
```

The likely cause is an incorrectly configured resource limit.

Proceed to the **Resolution** section.

### Case B: Memory usage is unusually high

Example:

```text
Memory limit: 128Mi
Normal usage: 40Mi
Current usage: 125Mi+
```

Investigate:

* increased traffic
* unusually large requests
* recent application changes
* possible memory leak
* unexpected workload behavior

Do not immediately assume the resource limit is the root cause.

### Case C: Memory usage is consistently increasing

If memory continually increases over time until the container is killed, investigate the application for a potential memory leak.

Review:

```bash
kubectl logs <pod-name> -n checkout-api --previous
```

and any available application/Prometheus/Grafana memory metrics.

---

# Resolution

## 9. Restore the Known-Good Memory Limit

If investigation confirms that the memory limit is incorrectly low, restore the known-good value.

For the incident described in this postmortem, that value was `128Mi`.

If the deployment is managed directly with `kubectl`, the memory limit can be changed with:

```bash
sed -i 's/"8Mi"/"128Mi"/g' checkout-api-deployment.yaml
kubectl apply -f checkout-api-deployment.yaml
```
---

## 10. Verify the Deployment

Check the rollout:

```bash
kubectl rollout status deployment/checkout-api -n checkout-api
```

Then check the pods:

```bash
kubectl get pods -n checkout-api
```

Confirm that the pods are running and the restart count has stopped increasing.

---

## 11. Verify the New Resource Configuration

Confirm that the deployment has the expected memory limit:

```bash
kubectl get deployment checkout-api -n checkout-api \
  -o jsonpath='{.spec.template.spec.containers[0].resources}'
```

Expected result should include:

```text
memory: 128Mi
```

---

## 12. Verify Application Health

Check the application logs:

```bash
kubectl logs deployment/checkout-api -n checkout-api
```

If the application exposes a health endpoint, test it through the Kubernetes service.

For example:

```bash
kubectl port-forward service/checkout-api 8080:80 -n checkout-api
```

Then, from another terminal:

```bash
curl http://localhost:8080/health
```

Expected result:

```text
healthy
```

or whatever successful response the application normally returns.

---

## 13. Monitor for Additional Restarts

Continue monitoring the pods:

```bash
kubectl get pods -n checkout-api -w
```

You can also monitor resource usage:

```bash
kubectl top pod -n checkout-api
```

Confirm that:

* Pods remain `Running`
* `READY` remains `1/1`
* Restart counts stop increasing
* Memory usage remains comfortably below the limit
* Health checks continue succeeding

---

# Escalation

Escalate the incident if:

* Pods continue getting `OOMKilled` after restoring an appropriate limit.
* Memory usage continues increasing unexpectedly.
* A suspected application memory leak is identified.
* Memory requirements are significantly higher than expected.
* Multiple services are experiencing memory pressure.
* The node itself is experiencing memory pressure.

Check node status with:

```bash
kubectl get nodes
```

And inspect the affected node:

```bash
kubectl describe node <node-name>
```

Look for:

```text
MemoryPressure
```

---

# Post-Incident Actions

After the service is stable:

1. Record the time of the incident and duration.
2. Record the memory limit before and after the change.
3. Record observed memory usage.
4. Document the root cause.
5. Determine whether the resource change came from Terraform, Helm, a manifest, or a manual `kubectl` change.
6. Review whether resource limits should be validated before deployment.
7. Consider adding alerts for:

   * container restarts
   * high memory utilization
   * `OOMKilled` events
8. Update this runbook with any additional findings.

---

# Quick Reference

```bash
# Check pods
kubectl get pods -n checkout-api

# Inspect affected pod
kubectl describe pod <pod-name> -n checkout-api

# Check events
kubectl get events -n checkout-api --sort-by='.lastTimestamp'

# Check resource usage
kubectl top pod -n checkout-api

# Check deployment resources
kubectl get deployment checkout-api -n checkout-api \
  -o jsonpath='{.spec.template.spec.containers[0].resources}'

# Check current logs
kubectl logs <pod-name> -n checkout-api

# Check logs from previous container
kubectl logs <pod-name> -n checkout-api --previous

# Restore known-good memory limit
kubectl set resources deployment checkout-api \
  -n checkout-api \
  --limits=memory=128Mi

# Monitor rollout
kubectl rollout status deployment/checkout-api -n checkout-api

# Monitor pods
kubectl get pods -n checkout-api -w

# Verify application
kubectl port-forward service/checkout-api 8080:80 -n checkout-api
curl http://localhost:8080/health
```

## Incident-specific conclusion

For the incident documented in the postmortem, the key diagnostic sequence was:

```text
Increasing pod restarts
        ↓
kubectl describe pod
        ↓
Last State: OOMKilled
        ↓
Check resource configuration
        ↓
Memory limit = 8Mi
        ↓
Compare against observed application usage
        ↓
Limit is insufficient
        ↓
Restore limit to 128Mi
        ↓
Verify rollout + health + restart count
        ↓
Incident resolved
```
