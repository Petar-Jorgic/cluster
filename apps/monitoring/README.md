# Monitoring

Prometheus + Grafana monitoring stack.

## Deployment

Helm chart: `kube-prometheus-stack v72.6.2`

## Components

| Component | Details |
|-----------|---------|
| Prometheus | Metrics collection, 24h retention |
| Grafana | Dashboards, PostgreSQL backend |
| Node Exporter | Host metrics |
| Kube-State-Metrics | Kubernetes object metrics |

## Disabled (K3s-specific)

Alertmanager, kubeControllerManager, kubeEtcd, kubeProxy, kubeScheduler

## Grafana

- **URL**: `/grafana` path
- **Auth**: Admin credentials from `grafana-secrets`
- **Anonymous access**: Enabled (Viewer role)
- **Database**: PostgreSQL (`grafana-postgres`)
- **Dashboard auto-discovery**: Via sidecar (label-based)

## Ingress

- Grafana via `/grafana` path on Traefik
