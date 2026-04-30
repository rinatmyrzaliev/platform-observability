# platform-observability

SLO & Observability Platform — multi-window burn-rate alerting on EKS.

Built on top of [platform-foundation](https://github.com/rinatmyrzaliev/platform-foundation) (kube-prometheus-stack) and [platform-golden-path](https://github.com/rinatmyrzaliev/platform-golden-path) (standardized services).

## Architecture

```
┌─────────────────┐      ┌──────────────────────────────────────────────┐
│ Traffic          │────▶│ ingress-nginx controller                     │
│ Generator        │     │ (metrics: port 10254)                        │
│ (busybox curl)   │     │   ┌──────────┐ ┌──────────┐ ┌──────────┐     │
└─────────────────┘      │   │ catalog  │ │ orders   │ │ payments │     │
                         │   └──────────┘ └──────────┘ └──────────┘     │
                         └──────────────────────┬───────────────────────┘
                                                │ ServiceMonitor scrape
                                                ▼
                        ┌──────────────────────────────────────────────┐
                        │ Prometheus                                   │
                        │   • Sloth-generated recording rules          │
                        │   • Multi-window burn-rate alert rules       │
                        │   • 56 recording rules + 4 alert rules       │
                        └───────────┬──────────────────┬───────────────┘
                                    │                  │
                          query     │                  │ fire alerts
                                    ▼                  ▼
                        ┌────────────────┐   ┌──────────────────┐
                        │ Grafana        │   │ Alertmanager     │
                        │  • SLO Overview│   │  • slo-critical  │
                        │  • Svc Detail  │   │  • slo-warning   │
                        └────────────────┘   └────────┬─────────┘
                                                      │
                                                      ▼
                                             ┌──────────────────┐
                                             │ Webhook Receiver │
                                             │ (Python HTTP log)│
                                             └──────────────────┘
```

## What's in this repo

```
platform-observability/
├── sloth/                          # SLO definitions (source of truth)
│   ├── catalog.yaml                # Availability 99.9% + Latency p99<500ms
│   └── orders.yaml                 # Same targets, different service
├── manifests/
│   ├── slo-rules/                  # Generated PrometheusRules (Sloth output)
│   │   ├── catalog-rules.yaml      # 28 recording rules + 2 alert rules
│   │   └── orders-rules.yaml       # 28 recording rules + 2 alert rules
│   ├── ingress-nginx/
│   │   └── servicemonitor.yaml     # Scrape ingress-nginx metrics
│   ├── dashboards/
│   │   ├── slo-overview-dashboard.yaml      # Error budget + burn rate at a glance
│   │   └── slo-service-detail-dashboard.yaml # Per-service RED + SLO detail
│   ├── alertmanager/
│   │   └── alertmanagerconfig.yaml # Route SLO alerts by severity
│   ├── traffic-generator/
│   │   └── deployment.yaml         # Synthetic traffic for lab
│   └── webhook-receiver/
│       └── deployment.yaml         # Alert endpoint (lab only)
└── docs/
    ├── decisions.md                # ADRs
    └── runbooks.md                 # Per-alert runbooks
```

## SLO definitions

| Service | SLO | Objective | Error budget (30d) | Metric |
|---------|-----|-----------|--------------------|--------|
| catalog | Availability | 99.9% | 43 min | `nginx_ingress_controller_requests` status 5xx |
| catalog | Latency | 99% (p99 < 500ms) | ~7.2 hours | `nginx_ingress_controller_request_duration_seconds_bucket` le=0.5 |
| orders | Availability | 99.9% | 43 min | Same metric, filtered by `exported_service` |
| orders | Latency | 99% (p99 < 500ms) | ~7.2 hours | Same metric, filtered by `exported_service` |

## Multi-window burn-rate alerting

Follows the Google SRE book approach. Two severity tiers:

| Tier | Burn rate | Short window | Long window | Action |
|------|-----------|-------------|-------------|--------|
| Page (critical) | 14x | 5 min | 1 hour | Wake on-call |
| Ticket (warning) | 3x | 30 min | 6 hours | Create ticket |

Both windows must be true simultaneously to fire. Short window catches the spike, long window confirms it's sustained.

## Quickstart

**Prerequisites:** platform-foundation cluster running with kube-prometheus-stack.

**1. Deploy the metrics pipeline:**
```bash
kubectl apply -f manifests/ingress-nginx/servicemonitor.yaml
kubectl apply -f manifests/traffic-generator/deployment.yaml
```

**2. Deploy SLO rules:**
```bash
kubectl apply -f manifests/slo-rules/
```

**3. Deploy dashboards:**
```bash
kubectl apply -f manifests/dashboards/
```

**4. Deploy alerting:**
```bash
kubectl apply -f manifests/webhook-receiver/deployment.yaml
kubectl apply -f manifests/alertmanager/alertmanagerconfig.yaml
```

**5. Access Grafana:**
```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
# Login: admin / prom-operator
# Dashboards: "SLO Overview" and "Service SLO Detail"
```

## Adding a new service

1. Copy `sloth/catalog.yaml` → `sloth/<service>.yaml`
2. Update `service`, `exported_service` label, and `alerting.name`
3. Generate rules: `sloth generate -i sloth/<service>.yaml -o manifests/slo-rules/<service>-rules.yaml`
4. Wrap in PrometheusRule CRD (add `apiVersion`, `kind`, `metadata` with `release: kube-prometheus-stack` label)
5. Apply: `kubectl apply -f manifests/slo-rules/<service>-rules.yaml`
6. Dashboard auto-discovers via the `$service` variable dropdown

## Regenerating rules after SLO changes

```bash
# Edit the Sloth spec
vi sloth/catalog.yaml

# Regenerate
sloth generate -i sloth/catalog.yaml -o manifests/slo-rules/catalog-rules.yaml

# Re-wrap in CRD (Sloth outputs raw rules, not K8s resources)
# Then apply
kubectl apply -f manifests/slo-rules/catalog-rules.yaml
```

## Key decisions

See [docs/decisions.md](docs/decisions.md) for full ADRs. Highlights:

- **Ingress-layer metrics** over app-level metrics — captures user-facing experience
- **Sloth CLI** over hand-written rules or controller mode — spec is source of truth, generated rules are build artifacts
- **Multi-window burn-rate** over simple threshold alerts — reduces false positives
- **AlertmanagerConfig CRD** over direct Secret patching — works with the Operator, not against it
- **ConfigMap dashboards** over Grafana API provisioning — GitOps-friendly, sidecar auto-imports
