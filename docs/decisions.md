# Architecture Decision Records

## ADR-001: Ingress-layer metrics for SLOs

**Status:** Accepted

**Context:** The Golden Path services are nginx containers that don't emit custom HTTP metrics. We need request count by status code and latency histograms to build SLOs.

**Options considered:**
1. **ingress-nginx metrics** — controller already runs in the cluster, emits `nginx_ingress_controller_requests` and `nginx_ingress_controller_request_duration_seconds_bucket`
2. **Deploy instrumented apps** — replace nginx with Go/Python services that expose `/metrics`
3. **nginx-prometheus-exporter sidecar** — exports `stub_status` data (connections, total requests only)

**Decision:** Use ingress-nginx metrics.

**Why:**
- Zero new code — controller already runs
- Captures user-facing experience — if ingress returns 5xx, that's what the user sees
- Provides both request count (by status code) and latency histogram
- How many production SRE teams actually build SLOs — at the edge, not inside each app

**Trade-off:** We can't distinguish between app errors and ingress errors. A misconfigured Ingress rule would look like an app failure. In production, you'd add app-level metrics too and correlate.

## ADR-002: Sloth CLI over hand-written rules

**Status:** Accepted

**Context:** Multi-window burn-rate alerting requires ~60 recording rules and 2 alert rules per SLO. Writing these by hand is error-prone.

**Options considered:**
1. **Sloth CLI** — write a 30-line SLO spec, Sloth generates all rules
2. **Hand-written PrometheusRules** — full control, full risk of mistakes
3. **Sloth controller** — deploy Sloth into the cluster, create PrometheusServiceLevel CRDs
4. **Pyrra** — similar to Sloth but with a UI

**Decision:** Sloth CLI, generate in dev, commit output to Git.

**Why:**
- Spec is the source of truth (~30 lines), generated rules are build artifacts (~200 lines)
- Same pattern used in CI pipelines — generate on commit, ArgoCD deploys the output
- No extra controller eating memory on our tight 2-node cluster
- We understand the math Sloth generates — can explain it in interviews

**Trade-off:** Generated output needs a manual CRD wrapper (Sloth 0.16 doesn't have `--kube` flag). A wrapper script in CI would handle this.

## ADR-003: Multi-window burn-rate over simple thresholds

**Status:** Accepted

**Context:** We need to alert when SLOs are at risk of being breached, not after they're already breached.

**Options considered:**
1. **Multi-window burn-rate** (Google SRE book) — 14x/5m+1h for page, 3x/30m+6h for ticket
2. **Simple threshold** — alert when error rate > 0.1%
3. **Single-window burn-rate** — alert when burn rate > 14x over 1h

**Decision:** Multi-window burn-rate.

**Why:**
- Two windows eliminate false positives. Short window catches the spike, long window confirms it's real.
- Simple thresholds don't account for budget consumption over time. A brief spike at 1% errors might look scary but barely dents a 30-day budget.
- Single-window alerts either fire too slow (long window) or too often (short window). Multi-window gives fast detection AND low false positive rate.
- Industry standard — Google, Datadog, and most SRE teams use this pattern.

## ADR-004: AlertmanagerConfig CRD over direct Secret patching

**Status:** Accepted

**Context:** We need to route SLO alerts to different receivers based on severity.

**Options considered:**
1. **Direct Secret patch** — edit the `alertmanager.yaml` Secret that Alertmanager reads
2. **AlertmanagerConfig CRD** — Prometheus Operator merges it into the running config
3. **Terraform Helm values** — update kube-prometheus-stack's alertmanager config in Terraform

**Decision:** AlertmanagerConfig CRD.

**Why:**
- The Prometheus Operator owns the Alertmanager Secret and continuously reconciles it. Manual edits get overwritten within seconds. We hit this exact problem — patched the Secret, it reverted immediately.
- AlertmanagerConfig is the Operator's supported API. It merges our routes into the running config and keeps them there.
- Terraform would work but is heavyweight — re-evaluates all 74 foundation resources for a config change.

**Trade-off:** AlertmanagerConfig auto-adds a `namespace` matcher. Alerts must carry a matching `namespace` label or they fall through to the default receiver. This caught us — our test alert initially lacked the label and was silently routed to `null`.

## ADR-005: Dashboard ConfigMaps over Grafana API provisioning

**Status:** Accepted

**Context:** We need to deploy Grafana dashboards in a reproducible, GitOps-friendly way.

**Options considered:**
1. **ConfigMaps with `grafana_dashboard: "1"` label** — Grafana sidecar auto-imports
2. **Grafana API / UI** — manual upload through the web interface
3. **Grafana Operator** — CRD-based dashboard management

**Decision:** ConfigMaps with sidecar label.

**Why:**
- Already configured — kube-prometheus-stack deploys Grafana with the sidecar watching for this label
- GitOps-friendly — dashboard JSON is in Git, deployed as a ConfigMap, deleted when the ConfigMap is removed
- No extra operator needed
- Same pattern kube-prometheus-stack uses for its own dashboards

## ADR-006: 99.9% availability and 99% latency targets

**Status:** Accepted

**Context:** We need to choose SLO targets for the lab services.

**Decision:** 99.9% availability (43 min error budget/30d), 99% latency under 500ms.

**Why:**
- 99.9% availability is the most common starting point for non-critical services. Aggressive enough to catch real problems, lenient enough to not page on every blip.
- 99% latency is deliberately looser than availability. Latency is noisier — network jitter, GC pauses, and cold starts all cause spikes that don't indicate a real problem. A tighter target (99.9%) would generate false pages.
- 500ms threshold is generous for a static nginx page but realistic for a production API. Gives us room to demonstrate the math without constant alerts.

**Trade-off:** In production, targets come from user expectations and business requirements, not arbitrary numbers. These are learning defaults.

## ADR-007: Synthetic traffic generator for lab

**Status:** Accepted

**Context:** SLOs need steady traffic to calculate meaningful burn rates. Lab services have no real users.

**Decision:** BusyBox Deployment sending ~5 requests/second with mixed status codes (~5% 404s).

**Why:**
- Simple — no external dependencies, 10 lines of shell
- Mixed traffic — pure 200s show 100% availability forever, which is useless for demonstrating dashboards. Intentional 404s create visible (but safe) error budget consumption.
- 404s are client errors (4xx), not server errors (5xx), so they don't burn the availability SLO. They show up in the request rate dashboard as context.

**Trade-off:** Synthetic traffic doesn't expose real-world patterns (burst traffic, slow degradation, dependency failures). In production, real user traffic is the SLO signal.
