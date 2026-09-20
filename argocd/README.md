# Argo CD GitOps Demo (Minikube + AKS)

Senior-SME walkthrough for **Argo CD 3.5.x** (current supported line as of Sep 2026).
Git is the source of truth. Argo CD pulls and reconciles the cluster.

```
Git push  →  Argo CD detects revision  →  Kustomize renders  →  Cluster reconciled
```

Repo: https://github.com/Marshall-S-Pillai/argocd-gitops-demo.git

## What you get

| Path | Purpose |
|------|---------|
| `argocd/project.yaml` | AppProject `demo` |
| `argocd/application-minikube.yaml` | Application CR for Minikube |
| `argocd/application-aks.yaml` | Application CR for AKS |
| `apps/demo/base` | Public nginx demo |
| `apps/demo/overlays/minikube` | 1 replica, NodePort |
| `apps/demo/overlays/aks` | 2 replicas, LoadBalancer |
| `apps/springboot` | Optional Spring Boot GHCR image |

Start with the nginx demo.

## Prerequisites

- kubectl
- minikube or an AKS cluster
- Optional: argocd CLI

## Path A — Minikube

```bash
minikube start --cpus=2 --memory=4096 --addons=ingress

kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server --timeout=180s

kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
kubectl -n argocd port-forward svc/argocd-server 8080:443
# https://localhost:8080  user=admin

kubectl apply -f argocd/project.yaml
kubectl apply -f argocd/application-minikube.yaml

kubectl get deploy,svc,pods -n demo
minikube service demo -n demo
```

Prove GitOps: change replica count in `apps/demo/overlays/minikube/kustomization.yaml`, push, wait or `argocd app sync demo-minikube`.

## Path B — AKS

```bash
az aks get-credentials --resource-group <RG> --name <AKS_NAME>

kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl apply -f argocd/project.yaml
kubectl apply -f argocd/application-aks.yaml
kubectl get svc demo -n demo -w
```

Do not expose default admin on a public LoadBalancer without TLS and SSO.

## Useful commands

```bash
argocd app list
argocd app get demo-minikube
argocd app sync demo-minikube
argocd app diff demo-minikube
kubectl delete -f argocd/application-minikube.yaml
kubectl delete ns demo
```
