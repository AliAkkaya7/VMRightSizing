# VM Right-Sizing with RHACM 2.17 (GitOps)

OpenShift Virtualization VM right-sizing with RHACM Observability, deployed by OpenShift GitOps,
with metrics exported to an external Prometheus Remote-Write endpoint.

## Architecture

```
Argo CD (openshift-gitops)
 └─ root app: vm-rightsizing  ──► apps/
     1 acm-operator       RHACM operator, channel release-2.17
     2 acm-hub            MultiClusterHub (hub is imported as local-cluster)
     3 acm-observability  S3 bucket (ODF MCG) + Thanos secret + MultiClusterObservability
                          ├─ virtualizationRightSizingRecommendation: enabled
                          └─ writeStorage ──► external endpoint (Remote-Write)
     3 metrics-receiver   test endpoint: Prometheus with Remote-Write receiver
     3 demo-vms           3 VMs: oversized / undersized / well-sized

Managed cluster: PrometheusRule acm-rs-virt-prometheus-rules ─► acm_rs_vm:* ─► hub Thanos
Hub: Grafana "RightSizing Recommendation" │ rbac-query-proxy API │ Remote-Write ─► metrics-receiver
```

## Repository

| Path | Content |
|---|---|
| `bootstrap/root-app.yaml` | App-of-apps; the only manual step |
| `apps/` | One Argo CD Application per component (sync waves 1-3) |
| `manifests/01-acm-operator` | Namespace, OperatorGroup, Subscription |
| `manifests/02-acm-hub` | MultiClusterHub |
| `manifests/03-observability` | ObjectBucketClaim, Job building `thanos-object-storage`, endpoint Secret, MultiClusterObservability |
| `manifests/04-metrics-receiver` | Test Remote-Write endpoint |
| `manifests/05-demo-vms` | Demo VMs |

## Prerequisites

- OpenShift 4.21+ with OpenShift Virtualization and ODF (MCG for S3, Ceph RBD for PVCs)
- OpenShift GitOps; the application controller needs cluster-admin

## Deploy

```bash
oc apply -f bootstrap/root-app.yaml
```

## Configuration

**Right-sizing** (`manifests/03-observability/04-multiclusterobservability.yaml`):
```yaml
spec:
  capabilities:
    platform:
      analytics:
        virtualizationRightSizingRecommendation:
          enabled: true
```

**Metrics export** (RHACM docs: *Observability > Export metrics to external endpoints*):
```yaml
# Secret metrics-receiver, namespace open-cluster-management-observability
stringData:
  ep.yaml: |
    url: http://metrics-receiver.metrics-receiver.svc:9090/api/v1/write
---
# MultiClusterObservability
spec:
  storageConfig:
    writeStorage:
      - name: metrics-receiver
        key: ep.yaml
```

## Verify

```bash
T=$(oc whoami -t)
HUB=$(oc get route rbac-query-proxy -n open-cluster-management-observability -o jsonpath='{.spec.host}')
EXT=$(oc get route metrics-receiver -n metrics-receiver -o jsonpath='{.spec.host}')
Q='acm_rs_vm:namespace:cpu_recommendation{namespace="vm-rightsizing-demo"}'

# 1. Right-sizing data on the hub (rbac-query-proxy, Prometheus HTTP API)
curl -sk -H "Authorization: Bearer $T" --data-urlencode "query=$Q" "https://$HUB/api/v1/query"

# 2. Same data in the external endpoint
curl -sk --data-urlencode "query=$Q" "https://$EXT/api/v1/query"

# 3. Export status (hub, Observe > Metrics): code 204 = accepted by the endpoint
sum by (name, code) (acm_remote_write_requests_total)
```

Grafana: route `grafana` in `open-cluster-management-observability`, folder **RightSizing Recommendation**.
Data appears about 20 minutes after Observability is ready (rules evaluate every 5m/15m).

## Test results

cluster-f8xhb, OCP 4.21.10, OpenShift Virtualization 4.21.24, ODF 4.21.15, RHACM 2.17.4, 2026-11-18.

| Check | Result |
|---|---|
| GitOps | 6 Argo CD apps Synced/Healthy from one `oc apply` |
| Hub install | MultiClusterHub Running, MultiClusterObservability Ready, `local-cluster` joined |
| Right-sizing rule | `acm-rs-virt-prometheus-rules` distributed to `openshift-monitoring`; all rules `ok` |
| Hub API | `acm_rs_vm:*` for all 3 VMs via `rbac-query-proxy` |
| Export | `acm_remote_write_requests_total{name="metrics-receiver",code="204"}`; external endpoint holds the same series and values as the hub, with `cluster`, `profile`, `aggregation` labels |

First-day values (profile `Max OverAll`, headroom 110%; identical on hub and external endpoint):

| VM | CPU request / peak / recommended | Memory GiB request / peak / recommended |
|---|---|---|
| demo-oversized-idle | 4 / 0.01 / 0.01 | 8.0 / 0.54 / 0.59 |
| demo-undersized-busy | 1 / 0.70 / 0.77 | 2.0 / 1.81 / 1.99 |
| demo-wellsized | 2 / 0.90 / 0.99 | 4.0 / 2.94 / 3.23 |

Values are the highest 5-minute peak of the day; they settle once a full day of data is collected.

## Notes

- `writeStorage` exports everything the hub receives (all collected metrics, not only right-sizing).
- In RHACM 2.17 the only profile is `Max OverAll`; P99/P95 profiles arrive with RHACM 5.0.
- The Thanos bucket credentials are generated in-cluster; a sync-hook Job writes `thanos-object-storage`, so no secret is stored in Git.
- Lab sizing: `availabilityConfig: Basic` and small PVCs. Production: `High`, default PVC sizes, TLS/auth on the endpoint (`http_client_config` in `ep.yaml`).

## Remove

```bash
oc delete application vm-rightsizing -n openshift-gitops --cascade=foreground
```
