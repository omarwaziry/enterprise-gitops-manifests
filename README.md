# Enterprise GitOps Manifests

[![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.30+-326CE5?style=flat&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![ArgoCD](https://img.shields.io/badge/Argo_CD-v3.4.4-EF7B4D?style=flat&logo=argo&logoColor=white)](https://argoproj.github.io/cd/)
[![Prometheus](https://img.shields.io/badge/Prometheus-v2.52.0-E6522C?style=flat&logo=prometheus&logoColor=white)](https://prometheus.io/)
[![Grafana](https://img.shields.io/badge/Grafana-11.1.0-F46800?style=flat&logo=grafana&logoColor=white)](https://grafana.com/)

This repository is the **CD half** of a two-repo GitOps pipeline. It contains every Kubernetes manifest needed to deploy the application and its full observability stack. Nothing in this repository is applied manually — Argo CD watches it and reconciles the cluster state to match whatever is committed here.

The **CI half** (Spring Boot source, Dockerfile, Jenkinsfile) lives here:
**[enterprise-gitops-cicd-pipeline](https://github.com/omarwaziry/enterprise-gitops-cicd-pipeline)**

---

## If You Are Cloning This Project

Six values to change before applying anything to a cluster. Every other field is ready to use.

| # | File | What to change |
|---|---|---|
| 1 | `k8s/argocd-app.yaml` | `spec.source.repoURL` — your manifests repo URL |
| 2 | `k8s/deployment.yaml` | `image` — your Docker Hub username and repo name |
| 3 | `k8s/ingress.yaml` | `spec.rules[0].host` — your hostname or local domain |
| 4 | `monitoring/argocd-app.yaml` | `spec.source.repoURL` — same manifests repo URL |
| 5 | `monitoring/prometheus-config.yaml` | `spring-boot-app` target — your service DNS name if namespace differs |
| 6 | `k8s/network-policy.yaml` | Verify `ingress-nginx` matches your ingress controller's namespace label |

---

## Table of Contents

1. [How This Repo Fits the Pipeline](#how-this-repo-fits-the-pipeline)
2. [Repository Structure](#repository-structure)
3. [Application Stack](#application-stack)
4. [Observability Stack](#observability-stack)
5. [Deploying Everything](#deploying-everything)
6. [Live Environment Screenshots](#live-environment-screenshots)
7. [Design Notes](#design-notes)

---

## How This Repo Fits the Pipeline

```
Jenkins CI (other repo)
    │
    │  Commits: "ci: bump image tag to 24 [skip ci]"
    │  Changes one line in k8s/deployment.yaml:
    │    image: your-username/your-app:23  →  your-username/your-app:24
    │
    ▼
┌──────────────────────────────────┐
│  This repository (main branch)   │
│                                  │
│  k8s/                            │
│    deployment.yaml  ◄── tag bumped here automatically
│    service.yaml                  │
│    ingress.yaml                  │
│    argocd-app.yaml               │
│    pdb.yaml                      │
│    network-policy.yaml           │
│                                  │
│  monitoring/                     │
│    prometheus + grafana + node   │
│    exporter manifests            │
└────────────────┬─────────────────┘
                 │
                 │  Argo CD polls every 3 minutes
                 │  (or immediately via webhook)
                 ▼
┌──────────────────────────────────┐
│  Argo CD                         │
│                                  │
│  Compares live cluster state     │
│  to this repo.                   │
│  Finds deployment.yaml differs.  │
│  Runs kubectl apply.             │
│  Rolling update begins.          │
└──────────────────────────────────┘
```

Jenkins has no kubeconfig and no cluster credentials. It writes to Git. Argo CD reads from Git. This means a compromised CI server cannot directly deploy to the cluster.

---

## Repository Structure

```
gitops-manifests-repo/
│
├── k8s/
│   ├── argocd-app.yaml        # Argo CD Application — registers this repo for the app stack
│   ├── deployment.yaml        # Spring Boot workload: 3 replicas, rolling update, probes
│   ├── service.yaml           # ClusterIP: internal routing port 80 → 8080
│   ├── ingress.yaml           # NGINX Ingress: hostname → service routing
│   ├── pdb.yaml               # PodDisruptionBudget: minAvailable 2 during node drain
│   └── network-policy.yaml    # NetworkPolicy: ingress from nginx-ns only, egress to monitoring + DNS
│
└── monitoring/
    ├── monitoring-app.yaml          # Argo CD Application — registers this repo for the monitoring stack
    ├── prometheus-config.yaml       # Prometheus scrape config (ConfigMap)
    ├── prometheus-deployment.yaml   # Prometheus v2.52.0 with resource limits
    ├── prometheus-service.yaml      # ClusterIP :9090
    ├── grafana-datasource.yaml      # Grafana datasource provisioning (ConfigMap)
    ├── grafana-deployment.yaml      # Grafana 11.1.0 with resource limits
    ├── grafana-service.yaml         # NodePort :30001 for browser access
    ├── node-exporter-daemonset.yaml # Node Exporter v1.8.1 with hostPID/hostNetwork
    └── node-exporter-service.yaml   # ClusterIP :9100
```

---

## Application Stack

### `deployment.yaml`

| Field | Value | Why |
|---|---|---|
| `replicas` | `3` | Survives a single node failure; enough for rolling update math |
| `maxUnavailable` | `0` | No pod is terminated before its replacement is ready — zero downtime |
| `maxSurge` | `1` | One extra pod at a time keeps resource usage predictable |
| `resources.requests` | `256Mi / 100m` | Tells the scheduler what the pod needs to land on a suitable node |
| `resources.limits` | `512Mi / 500m` | Hard ceiling that prevents one pod starving others on the node |
| `runAsNonRoot` | `true` | Kubernetes rejects the pod at admission if the image tries to run as root |
| `runAsUser` | `1000` | Matches the `appuser` UID created in the Dockerfile |
| `allowPrivilegeEscalation` | `false` | Blocks any attempt to gain elevated privileges inside the container |
| `livenessProbe` | `/actuator/health/liveness` | Restarts the container if the app enters an unrecoverable state |
| `readinessProbe` | `/actuator/health/readiness` | Removes the pod from Service endpoints during startup or degradation |

The liveness and readiness probes use Spring Boot Actuator's separated health groups. This means a pod can temporarily go un-ready (waiting for a dependency, warming up caches) without being killed and restarted.

The pod template also carries Prometheus scrape annotations so Prometheus can automatically discover and scrape the pod's `/actuator/prometheus` metrics endpoint:

```yaml
prometheus.io/scrape: "true"
prometheus.io/path:   "/actuator/prometheus"
prometheus.io/port:   "8080"
```

---

### `ingress.yaml`

Routes external HTTP traffic to the application service using an NGINX Ingress rule. Uses `spec.ingressClassName` instead of the deprecated `kubernetes.io/ingress.class` annotation.

For local Minikube access:
```bash
echo "$(minikube ip)  your-app.local" | sudo tee -a /etc/hosts
minikube addons enable ingress
```

---

### `pdb.yaml`

A `PodDisruptionBudget` with `minAvailable: 2` guarantees that during voluntary disruptions (node drains, cluster upgrades, autoscaler scale-downs) Kubernetes will evict at most one pod at a time. Without this, all 3 replicas could be evicted simultaneously during a node drain, causing a complete outage.

---

### `network-policy.yaml`

Enforces zero-trust pod-level networking. The application pods:
- Accept inbound traffic **only** from the NGINX Ingress Controller namespace
- Send outbound traffic **only** to the monitoring namespace (for Prometheus scraping) and to kube-dns (for service name resolution)

All other inbound and outbound traffic is dropped.

> **Prerequisite:** NetworkPolicy enforcement requires a CNI plugin that supports it — Calico, Cilium, Weave Net, or Antrea. For Minikube: `minikube start --cni=calico`.

---

## Observability Stack

### Architecture

```
monitoring namespace
┌──────────────────────────────────────────────────────┐
│                                                      │
│  Prometheus v2.52.0 (:9090)                         │
│  ├── scrapes → itself (self-monitoring)             │
│  ├── scrapes → node-exporter.monitoring:9100        │
│  └── scrapes → devops-showcase-service.production:80│
│                   /actuator/prometheus               │
│         │                                           │
│         │ datasource                                │
│         ▼                                           │
│  Grafana 11.1.0 (:3000)                             │
│  NodePort 30001 ─────────────────────────────────► browser
│  Datasource provisioned via ConfigMap               │
│                                                      │
│  Node Exporter v1.8.1 (DaemonSet)                   │
│  hostPID + hostNetwork + hostPath mount             │
│  Exposes hardware/OS metrics at :9100               │
└──────────────────────────────────────────────────────┘
```

### Prometheus scrape jobs

Three jobs are configured in `prometheus-config.yaml`:

| Job | Target | What it collects |
|---|---|---|
| `prometheus` | `localhost:9090` | Prometheus's own health and performance metrics |
| `node-exporter` | `node-exporter.monitoring.svc.cluster.local:9100` | Node CPU, memory, disk, network |
| `spring-boot-app` | `devops-showcase-service.production.svc.cluster.local:80/actuator/prometheus` | Application JVM, HTTP, and custom business metrics |

### Node Exporter DaemonSet

The DaemonSet runs with `hostPID: true`, `hostNetwork: true`, and a read-only host root filesystem mount. These are required for Node Exporter to read actual node-level metrics rather than container-level metrics. A toleration allows it to run on control-plane nodes, which is necessary on single-node clusters like Minikube.

### Image versions

All monitoring images are pinned to specific versions. To upgrade any component, update the tag in the deployment file, commit to Git, and Argo CD will roll it out.

| Component | Image | Version |
|---|---|---|
| Prometheus | `prom/prometheus` | `v2.52.0` |
| Grafana | `grafana/grafana` | `11.1.0` |
| Node Exporter | `prom/node-exporter` | `v1.8.1` |

---

## Deploying Everything

### Step 1 — Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl wait --for=condition=Ready pods --all -n argocd --timeout=120s
```

### Step 2 — Get the initial admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 --decode
echo
```

### Step 3 — Access the Argo CD UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Open https://localhost:8080
# Username: admin   Password: from Step 2
```

### Step 4 — Apply both Application manifests

```bash
# Registers the k8s/ directory — deploys the application stack
kubectl apply -f k8s/argocd-app.yaml

# Registers the monitoring/ directory — deploys the observability stack
kubectl apply -f monitoring/monitoring-app.yaml
```

Argo CD will immediately begin syncing both directories. The `production` and `monitoring` namespaces are created automatically.

### Step 5 — Access the running services

```bash
# Enable Minikube ingress addon if using Minikube
minikube addons enable ingress

# Add your hostname to /etc/hosts
echo "$(minikube ip)  your-app.local" | sudo tee -a /etc/hosts

# Application
open http://your-app.local

# Grafana (NodePort 30001)
open http://$(minikube ip):30001
# Default credentials: admin / admin — change on first login

# Prometheus (via port-forward)
kubectl port-forward svc/prometheus-service -n monitoring 9090:9090
open http://localhost:9090
```

---

## Live Environment Screenshots

### Argo CD — Both Applications Healthy and Synced

Two independent Argo CD `Application` resources manage this repository — one for the application stack (`k8s/`) and one for the monitoring stack (`monitoring/`). Both show `Healthy` and `Synced`, meaning the live cluster state matches Git exactly. The separate applications mean monitoring config changes and application deployments have independent sync histories and can be rolled back independently.

<p align="center">
  <img src="https://raw.githubusercontent.com/your-username/your-cicd-pipeline-repo/main/images/ArgoCD.png" alt="Argo CD Applications dashboard showing devops-showcase-gitops (namespace: production, path: k8s, Healthy, Synced, Last Sync 10 hours ago) and monitoring-gitops (namespace: monitoring, path: monitoring, Healthy, Synced, Last Sync 7 minutes ago). Both track the main branch." width="900"/>
</p>

---

### Prometheus — Active Scrape Targets

Both scrape targets are `UP`. The `node-exporter` job is scraping the Node Exporter DaemonSet via its stable cluster DNS name. The `prometheus` job is Prometheus scraping itself. The scrape durations (16ms and 14ms) and timestamps confirm the collection pipeline is running on the configured 15-second interval.

<p align="center">
  <img src="https://raw.githubusercontent.com/your-username/your-cicd-pipeline-repo/main/images/Prometheus.png" alt="Prometheus targets page. Job node-exporter: 1/1 up, endpoint node-exporter.monitoring.svc.cluster.local:9100/metrics, State UP, last scrape 7.3s ago, 16ms duration. Job prometheus: 1/1 up, endpoint localhost:9090/metrics, State UP, last scrape 11.1s ago, 14ms duration." width="900"/>
</p>

---

### Node Exporter — Metrics Endpoint Active

Node Exporter v1.8.1 is running inside the DaemonSet pod with access to the host filesystem. Its `/metrics` endpoint is reachable and is the target Prometheus scrapes every 15 seconds to collect CPU, memory, disk, and network statistics for the underlying VM.

<p align="center">
  <img src="https://raw.githubusercontent.com/your-username/your-cicd-pipeline-repo/main/images/NodeExporter.png" alt="Prometheus Node Exporter v1.11.1 landing page showing links to /metrics endpoint and pprof profiling endpoints. Running inside the CentOS 9 Minikube cluster." width="900"/>
</p>

---

### Grafana — Live Infrastructure Dashboard

The Node Exporter Full dashboard is showing real VM metrics from the moment the pipeline was running: `CPU Busy: 76%`, `RAM Used: 83.8%`, `SWAP Used: 50.2%`. This reflects the combined load of Jenkins, SonarQube, Minikube, and the running application on an 8 GB VM. The Prometheus datasource was provisioned automatically at startup via the `grafana-datasource.yaml` ConfigMap — no manual configuration.

<p align="center">
  <img src="https://raw.githubusercontent.com/your-username/your-cicd-pipeline-repo/main/images/Grafana.png" alt="Grafana Node Exporter Full dashboard. Gauges: CPU Busy 76%, Sys Load 71%, RAM Used 83.8%, SWAP Used 50.2%. Datasource: Prometheus. Instance: node-exporter.monitoring.svc.cluster.local:9100. Time-series panels for CPU, Memory, Network Traffic, and Disk Space." width="900"/>
</p>

---

## Design Notes

### Why plain YAML instead of Helm or Kustomize?

Every field is visible and every decision is documented inline. There is no indirection through templates or values files. The goal of this project is to demonstrate Kubernetes knowledge, and plain YAML does that more directly than a Helm chart that hides the actual resource definitions.

### Why two separate Argo CD Applications?

The `k8s/` and `monitoring/` directories are managed by separate Argo CD `Application` resources deliberately. A Grafana dashboard config change does not trigger a sync of the application `Deployment`. Each stack has its own health status, sync history, and rollback capability in the Argo CD UI.

### Why does Jenkins only write a single line to this repo?

The only thing that changes between deployments is the image tag in `deployment.yaml`. All other infrastructure configuration — resource limits, probes, network policies, replica counts — is static and must go through a Git commit to change. This contains the blast radius of a CI compromise: a stolen Jenkins token can only bump an image tag, not modify RBAC, network policies, or any other cluster configuration.

### Self-healing in practice

Both Argo CD applications have `selfHeal: true`. If someone runs `kubectl scale deployment --replicas=1` to debug something, Argo CD reverts it back to `replicas: 3` on the next sync cycle. The same applies to `kubectl edit`, `kubectl patch`, and `kubectl delete`. The cluster always converges to what Git says.
