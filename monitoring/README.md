# Monitoring stack (GitOps: Argo CD -> Helm -> AKS)

Deploys into namespace `monitoring` (auto-created) using **official charts only**:

| Component | Chart repo | Chart | Version |
|---|---|---|---|
| Prometheus + Operator + node-exporter + kube-state-metrics + Alertmanager | `https://prometheus-community.github.io/helm-charts` | `kube-prometheus-stack` | **91.9.0** |
| Grafana | `https://grafana-community.github.io/helm-charts` (successor of `grafana/helm-charts` since Jan-2026) | `grafana` | **13.2.7** |

No custom Deployment for Prometheus/Grafana. No `helm install` / `kubectl apply`
needed after the one-time bootstrap below.

## Layout (only `gitops/` is touched)

```text
gitops/
├── argocd/
│   ├── application.yaml      # existing app (namespace realtime-monitoring) - untouched
│   └── monitoring.yaml       # NEW app-of-apps -> path: monitoring (apply ONCE)
├── monitoring/
│   ├── kustomization.yaml    # lists the two child Applications
│   ├── README.md             # this file
│   ├── prometheus/
│   │   ├── application.yaml  # Argo CD Application (multi-source: Git values + chart)
│   │   └── values.yaml       # kube-prometheus-stack values (grafana.enabled=false)
│   └── grafana/
│       ├── application.yaml  # Argo CD Application (values + chart + manifests/)
│       ├── values.yaml       # ClusterIP, persistence, datasource, hostname knob
│       └── manifests/
│           ├── secretproviderclass-grafana.yaml  # Key Vault -> grafana-admin-secret
│           └── dashboards.yaml                   # 2 provisioned dashboards
└── deploy/helm/realtime-monitoring/monitoring/
    ├── grafana-application.yaml  # PRE-EXISTING 0-byte placeholders, left untouched
    └── grafana-values.yaml       # (kept to avoid breaking anything referencing them)
```

## One-time bootstrap

```bash
# from the Gitops repo root
kubectl apply -f gitops/argocd/monitoring.yaml
# Argo CD then syncs monitoring-prometheus and monitoring-grafana automatically
```

## How scraping works

`monitoring/prometheus/values.yaml` defines one `additionalServiceMonitors`
(`realtime-monitoring-spring-boot`) selecting Services in namespace
`realtime-monitoring` with label `app.kubernetes.io/part-of: realtime-monitoring`
(which `deploy/helm/realtime-monitoring/templates/services.yaml` already sets),
port `http`, path `/actuator/prometheus`, every 30s. Node/pod/workload metrics
come from node-exporter + kube-state-metrics. `kubeControllerManager/Scheduler/
Etcd/Proxy` are disabled because AKS manages the control plane.

Services with no actuator endpoint are dropped:
`frontend-service|config-service|discovery-server`.

## Spring Boot prerequisite (NOT applied - action required in service repos)

Actuator is present in all poms, but NO service has
`micrometer-registry-prometheus` and no config-repo yaml exposes
`/actuator/prometheus` (only `/actuator/health/*` probes exist).

Minimal change per service (example for Spring Boot 3.x/4.x):

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  endpoint:
    prometheus:
      access: unrestricted
```

Until then, the `up{job="realtime-monitoring-spring-boot"}` targets stay DOWN
except dropped services; cluster/node metrics still work.

## Grafana

- Service `ClusterIP` (no ingress controller in repo). Optional ingress block is
  present but disabled; set `grafanaHostname: grafana.<my-domain>` + enable it
  when an ingress controller exists.
- Datasource `Prometheus` (default):
  `http://monitoring-prometheus.monitoring.svc.cluster.local:9090`
  (deterministic via `fullnameOverride: monitoring`).
- Persistence: 5Gi PVC on `managed-csi` (configurable, set `""` for default SC).
- Credentials: `admin.existingSecret: grafana-admin-secret` synced from Key Vault
  objects `grafana-admin-user` / `grafana-admin-password` (create them once with
  `az keyvault secret set`; never in Git).
- Dashboards: `Spring Boot / JVM overview` + `Kubernetes pods overview`
  (sidecar `grafana_dashboard: "1"`). Community IDs intentionally inlined for
  compatibility; import 6417/315/4701 later if desired.

## Verify

```bash
kubectl -n argocd get applications monitoring monitoring-prometheus monitoring-grafana
kubectl -n monitoring get pods,svc,pvc
kubectl -n monitoring port-forward svc/monitoring-prometheus 9090:9090   # Targets page
kubectl -n monitoring port-forward svc/grafana 3000:80                   # Grafana UI
```

In Grafana: Configuration -> Data sources -> Prometheus (default) -> Save & test.
