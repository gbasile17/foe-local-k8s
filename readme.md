# Local K8s Stack


This repo is a runbook for how to host a local kubernetes deployment (minicube) on Ubuntu with monitoring (Prometheus, Grafana), caching (redis), and ArgoCD as the deployment mechanism. 

## Prerequisites 
- [Docker](https://www.docker.com/)
- [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
- [Helm](https://helm.sh/docs/intro/install/)
- [Argo CLI](https://argo-workflows.readthedocs.io/en/latest/walk-through/argo-cli/)
- [Git](https://git-scm.com/downloads)

# Cluster Setup

## Install Minikube

```bash
sudo apt update && sudo apt upgrade -y
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube config set driver docker
minikube start --cpus=2 --memory=4g --disk-size=5g

```

## Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Accessing Argo via Localhost

Forward argo port to 8080 and change admin password

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
argopass=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
argocd login localhost:8080 --username admin --password $argopass
argocd account update-password
```

Argo's UI is accessible via the browser at localhost:8080

### Setup Argo App Directory

To initialize Argo with a GitOps style approach, we need to bootstrap Argo's configuration by deploying an Argo app that looks for other Argo apps.

Umbrella.yaml:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argo-umbrella
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/gbasileGP/foe-local-k8s'
    path: apps
    targetRevision: HEAD
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: argocd
  syncPolicy:
    automated:
      selfHeal: true
      prune: true

```

Once deployed, Argo will monitor the apps directory for new deployments.

## Setup Monitoring

### Prometheus/Grafana

We will deploy Prometheus from their latest helm chart. Disabling a bunch of the stack we do not need for this demo. 

Chart.yaml:
```yaml
apiVersion: v2
name: prometheus
version: 0.0.1
type: application
dependencies:
  - name: kube-prometheus-stack
    version: 57.1.1
    repository: https://prometheus-community.github.io/helm-charts
```

values.yaml:
```yaml
kube-prometheus-stack:
  alertmanager:
    enabled: false
  kubeStateMetrics:
    enabled: true
  nodeExporter:
    enabled: false
  grafana:
    enabled: true
  kubernetesServiceMonitors:
    enabled: false
  prometheus:
    prometheusSpec:
      serviceMonitorSelectorNilUsesHelmValues: false
      serviceMonitorNamespaceSelector:
        matchLabels:
          useServiceMonitor: "true"
```

We add an Argo app declaration.
monitoring.yaml:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: prometheus
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/gbasileGP/foe-local-k8s.git'
    targetRevision: HEAD
    path: kube-prometheus-stack
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: monitoring
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```

This chart also deploys the latest grafana chart with its default configuration, which should be fine for the purpose of this exercise. The default service port for grafana is 80. We can forward that out to view the dashboard via localhost.
```bash
kubectl port-forward svc/prometheus-grafana -n monitoring 8081:80
```

Get the login password fror admin with 
```bash
kubectl -n monitoring get secret prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 -d
```

## Setup Redis Cluster

We will deploy redis-cluster from the latest helm chart.

Argo app declaration:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: redis-cluster
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/gbasileGP/foe-local-k8s.git'
    targetRevision: HEAD
    path: redis-cluster
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: redis-cluster
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```

to get the password to the cluster run this command
```
kubectl get secret --namespace redis-cluster redis-cluster -o jsonpath="{.data.redis-password}" | base64 --decode
```
use this password to connect to the redis cluster

## Deploy More Apps to the cluster

You may deploy any other container applications to this cluster by committing new Argo app manifests in the /apps directory, and providing the manifests within the repo.

## Enable Nginx Ingress

to enable the Nginx ingress controller in Minicube, run the following command.

```yaml
minikube addons enable ingress
```

Now you may deploy ingress resources that point to your applications like so:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: pubg-leaderboard-ingress
spec:
  rules:
  - host: pubg-leaderboard.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: pubg-leaderboard-service
            port:
              number: 80

```

To access the application first get the minicube IP
```bash

```



