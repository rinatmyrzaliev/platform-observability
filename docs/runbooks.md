# SLO Alert Runbooks

## How to use these runbooks

When an SLO burn-rate alert fires, find the matching section below. Each runbook follows the same structure: what fired, why it matters, what to check, and how to resolve.

**Access tools:**
```bash
# Grafana (dashboards)
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80

# Prometheus (queries)
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090

# Alertmanager (silence/inhibit)
kubectl port-forward -n monitoring svc/kube-prometheus-stack-alertmanager 9093:9093
```

---

## CatalogAvailabilityBurnRate / OrdersAvailabilityBurnRate

### What fired

The service is returning 5xx errors faster than the error budget allows. If this continues, the 30-day error budget will be exhausted.

- **severity: critical** — 14x burn rate sustained over 5m AND 1h. Budget gone in ~2 days.
- **severity: warning** — 3x burn rate sustained over 30m AND 6h. Budget gone in ~10 days.

### Why it matters

At the current error rate, users are experiencing failures above the agreed SLO target (99.9%). Every minute this continues costs error budget that can't be reclaimed.

### What to check

**1. Confirm the error rate in Grafana:**
Open "Service SLO Detail" dashboard, select the affected service. Look at "Error rate (5xx)" panel.

**2. Check if pods are healthy:**
```bash
kubectl get pods -n <service-namespace>
kubectl describe deploy <service-name> -n <service-namespace>
```

**3. Check recent events:**
```bash
kubectl get events -n <service-namespace> --sort-by='.lastTimestamp' | tail -20
```

**4. Check ingress-nginx for upstream errors:**
```bash
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller --tail=50 | grep "upstream"
```

**5. Check if the backend service endpoint exists:**
```bash
kubectl get endpoints <service-name> -n <service-namespace>
```
If endpoints list is empty, no healthy pods are backing the service.

**6. Check node health:**
```bash
kubectl get nodes
kubectl top nodes
```

### How to resolve

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| 0 endpoints | Pods crashed or scaled to 0 | Check deployment events, fix crash, scale up |
| Pods in CrashLoopBackOff | App error, OOM, or config issue | Check pod logs: `kubectl logs <pod> -n <ns> --previous` |
| 503 from ingress | Backend unreachable | Check Service selector matches pod labels |
| Node NotReady | Node failure or spot reclaim | Wait for ASG replacement or scale nodegroup |
| High latency causing timeouts | Resource exhaustion | Check CPU/memory, increase limits or replicas |

### After resolution

1. Verify error rate drops to 0 in Grafana
2. Check error budget remaining — if exhausted, consider a change freeze
3. Add a breakage log entry
4. If critical: write a brief incident review

---

## CatalogLatencyBurnRate / OrdersLatencyBurnRate

### What fired

More than 1% of requests are taking longer than 500ms. The latency SLO (99% under 500ms) is burning budget.

- **severity: critical** — 14x burn rate over 5m AND 1h. Sustained high latency.
- **severity: warning** — 3x burn rate over 30m AND 6h. Gradual degradation.

### Why it matters

Users are experiencing slow responses. While the service isn't failing (availability may be fine), the user experience is degraded.

### What to check

**1. Confirm latency spike in Grafana:**
Open "Service SLO Detail" dashboard. Check "Latency percentiles" panel — look at p99 vs the 500ms threshold.

**2. Check pod resource usage:**
```bash
kubectl top pods -n <service-namespace>
```
CPU throttling causes latency spikes. Memory pressure triggers GC pauses.

**3. Check ingress-nginx response times:**
```bash
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller --tail=50 | grep "request_time"
```

**4. Check node resource pressure:**
```bash
kubectl top nodes
kubectl describe node <node-name> | grep -A 5 "Conditions"
```

**5. Check for noisy neighbors:**
```bash
kubectl top pods -A --sort-by=cpu | head -10
```

### How to resolve

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| CPU at limit | Throttling | Increase CPU limit or add replicas |
| Memory near limit | GC pressure | Increase memory limit |
| Node CPU > 80% | Noisy neighbor | Spread pods, scale nodegroup |
| Latency only on one pod | Pod-level issue | Delete the pod, let it reschedule |
| Latency across all pods | Upstream dependency or node issue | Check dependent services, node health |

### After resolution

1. Verify p99 drops below 500ms in Grafana
2. Review if the 500ms threshold is appropriate for this service
3. Consider whether this was a one-time spike or a trend

---

## General: Silencing an alert during maintenance

If you need to silence an alert during planned maintenance:

```bash
# Port-forward to Alertmanager
kubectl port-forward -n monitoring svc/kube-prometheus-stack-alertmanager 9093:9093

# Open http://localhost:9093/#/silences/new
# Match on: sloth_service=<service>, severity=<level>
# Set duration for the maintenance window
```

Always remove silences after maintenance is complete.

## General: Checking error budget status

```bash
# Quick check via Prometheus API
curl -s 'http://localhost:9090/api/v1/query' \
  --data-urlencode 'query=slo:period_error_budget_remaining:ratio * 100' \
  | python3 -c "
import json,sys
data=json.load(sys.stdin)
for r in data['data']['result']:
    svc=r['metric'].get('sloth_service','?')
    slo=r['metric'].get('sloth_slo','?')
    pct=float(r['value'][1])
    print(f'{svc}/{slo}: {pct:.1f}% remaining')
"
```

Or open the "SLO Overview" dashboard in Grafana for the visual version.
