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
