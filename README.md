# argocd-gitops-demo (Minikube only)

Argo CD GitOps lab for a **local Minikube** cluster.

- Argo CD: **3.5.x** via the official `stable` install manifest
- Demo app: public `nginx:1.27-alpine` (no image pull secret)
- Git is the source of truth

**How to run:** follow **Step 1 to Step 10** below.

Repo: https://github.com/Marshall-S-Pillai/argocd-gitops-demo

---

## What this repo contains

```
argocd/
  project.yaml                  # AppProject named demo
  application-minikube.yaml     # Application that syncs the nginx app
  application-springboot.yaml   # optional second app (needs GHCR access)
  README.md                     # same steps, extra notes
apps/demo/base/                 # Deployment + Service
apps/demo/overlays/minikube/    # 1 replica, NodePort
apps/springboot/                # optional Spring Boot image
```

---

## Step 1 — Install tools on your laptop

You need:

| Tool | Why |
|------|-----|
| Docker Desktop (or Podman) | Minikube driver |
| minikube | Local Kubernetes |
| kubectl | Talk to the cluster |
| git | Clone this repo |
| argocd CLI (optional) | Easier than kubectl for apps |

macOS:

```bash
brew install minikube kubectl git argocd
```

Linux (example):

```bash
# minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# argocd CLI (optional)
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd && sudo mv argocd /usr/local/bin/argocd
```

Check:

```bash
docker version
minikube version
kubectl version --client
git --version
```

---

## Step 2 — Clone this repository

```bash
git clone https://github.com/Marshall-S-Pillai/argocd-gitops-demo.git
cd argocd-gitops-demo
```

You must run later `kubectl apply -f argocd/...` from this folder.

---

## Step 3 — Start Minikube

```bash
minikube start --cpus=2 --memory=4096 --addons=ingress
minikube status
kubectl get nodes
```

Expected: one node `Ready`.

If start fails, confirm Docker is running, then retry.

---

## Step 4 — Install Argo CD into Minikube

```bash
kubectl create namespace argocd

kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait until the API server Deployment is ready (first pull can take 2–5 minutes):

```bash
kubectl -n argocd rollout status deploy/argocd-server --timeout=300s
kubectl -n argocd get pods
```

Every pod should be `Running` / `Completed`. If a pod is `ImagePullBackOff`, wait or check `minikube image` / network.

---

## Step 5 — Get the admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

Copy that string. Username is always `admin`.

---

## Step 6 — Open the Argo CD UI

Leave this command running in a dedicated terminal:

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

Browser:

1. Open https://localhost:8080
2. Accept the self-signed certificate warning
3. Login: `admin` + the password from Step 5

You should see an empty Applications list.

Optional CLI login (another terminal):

```bash
argocd login localhost:8080 --username admin --insecure
```

---

## Step 7 — Register the AppProject

An AppProject limits which Git repo and namespaces Argo CD may use.

```bash
cd argocd-gitops-demo   # if you left the folder
kubectl apply -f argocd/project.yaml
kubectl -n argocd get appproject
```

You should see project `demo` besides the built-in `default`.

---

## Step 8 — Create the Application (this starts GitOps)

```bash
kubectl apply -f argocd/application-minikube.yaml
kubectl -n argocd get applications
```

Watch until `SYNC STATUS` is `Synced` and `HEALTH` is `Healthy`:

```bash
kubectl -n argocd get applications -w
```

What this Application does:

- repo: `https://github.com/Marshall-S-Pillai/argocd-gitops-demo.git`
- path: `apps/demo/overlays/minikube`
- destination namespace: `demo` (created automatically)
- auto-sync + self-heal + prune are ON

In the UI you should now see application **demo-minikube**.

If it stays `Unknown` / `Missing`, click **Refresh**, or:

```bash
argocd app get demo-minikube
argocd app sync demo-minikube
```

---

## Step 9 — Open the demo website

```bash
kubectl get deploy,svc,pods -n demo
```

You should see Deployment `demo`, Service `demo` (`NodePort`), and 1 pod `Running`.

Easiest on Minikube:

```bash
minikube service demo -n demo
```

That prints a URL and often opens the browser. You should see the default nginx page.

Alternative (port-forward):

```bash
kubectl -n demo port-forward svc/demo 8081:80
```

Then open http://localhost:8081

---

## Step 10 — Prove GitOps, then clean up

### 10a. Change Git, watch the cluster follow

1. Edit `apps/demo/overlays/minikube/kustomization.yaml`
2. Change `count: 1` to `count: 2`
3. Commit and push:

```bash
git add apps/demo/overlays/minikube/kustomization.yaml
git commit -m "Scale demo to 2 replicas"
git push origin main
```

4. In the UI click **Refresh**, or wait about 3 minutes, or run:

```bash
argocd app sync demo-minikube
kubectl -n demo get deploy demo
```

Replicas should become **2**.

### 10b. Self-heal (optional)

```bash
kubectl -n demo scale deploy/demo --replicas=5
# wait 1–3 minutes
kubectl -n demo get deploy demo
```

Argo CD should scale it back to the Git value (`2` after 10a, or `1` if you did not push).

### 10c. Clean up the lab

```bash
kubectl delete -f argocd/application-minikube.yaml
kubectl delete ns demo

# remove Argo CD itself
kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl delete ns argocd

minikube stop
# minikube delete   # only if you want the cluster gone
```

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `argocd-server` pods not Ready | Wait for image pull; `kubectl -n argocd describe pod ...` |
| UI certificate warning | Expected on port-forward; proceed |
| Application `Unknown` | Check repo URL is public; click Refresh |
| Pod `ImagePullBackOff` on nginx | Minikube/Docker network; retry |
| `minikube service` hangs | Use port-forward in Step 9 |
| Password command prints nothing | Secret exists only until you change the admin password |

---

## Optional — Spring Boot app

After Step 8 works:

```bash
kubectl apply -f argocd/application-springboot.yaml
```

This deploys `ghcr.io/marshall-s-pillai/java-docker-gha-practice:latest`.
If that package is private, create a `docker-registry` secret in namespace `springboot` first.
