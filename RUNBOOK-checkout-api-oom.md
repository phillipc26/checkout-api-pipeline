# Runbook: checkout-api OOMKilled Incident

**Service:** `checkout-api`
**Namespace:** `checkout-api`
**Trigger:** Pods restarting repeatedly, `RESTARTS` climbing on `kubectl get pods`, intermittent request failures

Based on the [postmortem of the 2026-09 incident](#) where a memory limit was
lowered from 128Mi to 8Mi during a cost-tuning pass, causing the container to
be killed by the kernel OOM killer on startup.

---

## 1. Confirm the Symptom

Before diving into diagnosis, confirm this is actually an OOM situation and
not something else (crash loop from a bad image, failed liveness probe, etc.).

```bash
kubectl get pods -n checkout-api -o wide
```

Look for a climbing `RESTARTS` count on one or more `checkout-api` pods. Note
the exact pod name(s) — you'll need them for the next step.

---

## 2. Diagnosis

### 2.1 Confirm OOMKilled as the restart reason

```bash
kubectl describe pod <pod-name> -n checkout-api
```

Check the `Last State` block under `Containers`. You're looking for:

```
Last State:     Terminated
  Reason:       OOMKilled
  Exit Code:    137
```

Exit code `137` (128 + SIGKILL/9) alongside `Reason: OOMKilled` confirms the
kernel killed the container for exceeding its memory limit — this is
different from a `CrashLoopBackOff` caused by an application error.

### 2.2 Check the current resource limits

```bash
kubectl get deployment checkout-api -n checkout-api -o jsonpath='{.spec.template.spec.containers[0].resources}'
```

or for a more readable view:

```bash
kubectl describe deployment checkout-api -n checkout-api | grep -A 6 "Limits\|Requests"
```

Compare the configured `memory` limit against what the app actually needs
(next step). If the limit looks implausibly low for the workload (as in the
8Mi case), that's your likely root cause — move straight to Resolution.

### 2.3 Check actual memory usage (if metrics-server is available)

```bash
kubectl top pod -n checkout-api
```

If pods are currently alive, this shows real-time memory consumption you can
compare against the configured limit. If pods are crash-looping too fast to
sample, use the previous container's logs instead:

```bash
kubectl logs <pod-name> -n checkout-api --previous
```

This pulls logs from the terminated container instance, which can show the
app's memory footprint at startup, initialization steps, or any explicit
out-of-memory errors logged before the kill.

### 2.4 Check for a genuine leak vs. a misconfigured limit

```bash
kubectl get events -n checkout-api --sort-by='.lastTimestamp' | grep -i checkout-api
```

Scan the event history:
- **Single limit change immediately followed by OOMKilled events** → misconfigured
  limit (what happened in this incident).
- **Gradual RESTARTS increase over hours/days with no recent limit change** →
  possible memory leak; treat as a separate investigation rather than a
  simple revert.

---

## 3. Resolution

### 3.1 Identify the last known-good value

Check version control / Helm values / Terraform state for the memory limit
before the change (in this incident: `128Mi`). If unavailable, use the
`kubectl top pod` reading from 2.3 as a floor and add headroom (roughly
1.5–2x observed peak usage is a reasonable starting point).

### 3.2 Apply the fix

**Quick fix (imperative, for restoring service fast):**

```bash
sed -i 's/"8Mi"/"128Mi"/g' checkout-api-deployment.yaml
```

**Preferred fix (if the deployment is managed via manifests/Terraform):**
revert the value in source control and reapply, so the fix persists and
isn't clobbered by the next `apply`/`terraform apply`:

```bash
kubectl apply -f checkout-api-deployment.yaml
```

### 3.3 Roll out and verify

```bash
kubectl rollout status deployment checkout-api -n checkout-api
```

Watch pods stabilize (RESTARTS should stop climbing):

```bash
kubectl get pods -n checkout-api -w
```

Confirm no new OOMKilled events since the fix:

```bash
kubectl get events -n checkout-api --sort-by='.lastTimestamp' | tail -n 20
```

Spot-check that requests are succeeding again (adjust for however
checkout-api is exposed — Service/Ingress):

```bash
kubectl get endpoints checkout-api -n checkout-api
```

---

## 4. Prevention Follow-Ups

From the postmortem's action items — track these separately from the
incident response itself:

- **Check usage before changing limits.** Before any resource-limit change,
  pull recent `kubectl top pod` data (or historical data from
  Prometheus/Grafana if available) rather than adjusting values blind.
- **Guardrail unrealistic values.** Consider a pre-merge check (CI lint,
  admission policy, or PR review checklist) that flags memory limits below
  a sane floor for the workload before they reach the cluster.
- **Prefer gradual changes.** Cost-tuning changes to resource limits should
  step down incrementally with observation between steps, not jump straight
  to an aggressive value.

---

## Quick Reference

| Step | Command |
|---|---|
| Check restart count | `kubectl get pods -n checkout-api -o wide` |
| Confirm OOMKilled | `kubectl describe pod <pod> -n checkout-api` |
| Check current limits | `kubectl describe deployment checkout-api -n checkout-api` |
| Check live memory usage | `kubectl top pod -n checkout-api` |
| Check crashed container's logs | `kubectl logs <pod> -n checkout-api --previous` |
| Check recent events | `kubectl get events -n checkout-api --sort-by='.lastTimestamp'` |
| Quick-fix the limit | `kubectl set resources deployment checkout-api -n checkout-api --limits=memory=128Mi --requests=memory=64Mi` |
| Reapply from manifest | `kubectl apply -f deployment.yaml -n checkout-api` |
| Verify rollout | `kubectl rollout status deployment checkout-api -n checkout-api` |
