# argocd-gitops-demo

Easy Argo CD GitOps lab for **Minikube** and **AKS**.

- Argo CD line: **3.5.x** (install from `stable`, pin `v3.5.3` in production)
- Sample app: public `nginx:1.27-alpine` so it runs without image pull secrets
- Overlays: Minikube (NodePort, 1 replica) and AKS (LoadBalancer, 2 replicas)

**Full run steps:** [argocd/README.md](argocd/README.md)

```
Git push → Argo CD detects revision → Kustomize renders → cluster reconciled
```

Repo: https://github.com/Marshall-S-Pillai/argocd-gitops-demo
