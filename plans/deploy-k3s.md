# Plan: Deploy K3s + ArgoCD + Envoy Gateway on Meerkat

## Context
Fresh single-node HomeLab setup on a System 75 Meerkat mini computer. Nothing is installed yet — no k3s, kubectl, ArgoCD, or Envoy Gateway. Goal is a GitOps-ready cluster with Envoy Gateway as the ingress controller, ultimately serving traffic for `longwood.daphnai.com` with TLS.

---

## Step 1: Install k3s (disable Traefik)

Traefik is k3s's default ingress controller; we must disable it so Envoy Gateway can take that role. ServiceLB (klipper-lb) is kept enabled to handle `LoadBalancer` type services that Envoy Gateway will create.

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik" sh -
```

**Configure kubeconfig for non-root access:**
```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
chmod 600 ~/.kube/config
```

**Verify:**
```bash
kubectl get nodes
kubectl get pods -A
```

---

## Step 2: Deploy ArgoCD (non-HA)

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=Available deployment/argocd-server -n argocd --timeout=300s
```

**Get initial admin password:**
```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

**Access the UI** (port-forward for now — Envoy Gateway will replace this later):
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Open https://localhost:8080, login: admin / <password above>
```

---

## Step 3: Deploy Envoy Gateway via ArgoCD

Create manifest: **`manifests/envoy-gateway-argocd-app.yaml`**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: envoy-gateway
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://charts.envoyproxy.io
    chart: gateway-helm
    targetRevision: v1.4.1   # update to latest stable at deploy time
    helm:
      values: |
        deployment:
          replicas: 1
  destination:
    server: https://kubernetes.default.svc
    namespace: envoy-gateway-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Apply:
```bash
kubectl apply -f manifests/envoy-gateway-argocd-app.yaml
```

**Verify:**
```bash
kubectl get pods -n envoy-gateway-system
kubectl get gatewayclass
```

---

## Step 4: TLS + Domain Discussion

This step is a discussion with the user. Key decisions:

- **Domain:** `longwood.daphnai.com` (registrar: Porkbun)
- **Certificate tool:** cert-manager with Let's Encrypt (ACME)
- **Challenge type:** DNS-01 via Porkbun API (recommended for a home IP that may not have port 80 exposed, and for wildcard certs)
- **DNS record:** Point `longwood.daphnai.com` → home IP (or DDNS)

**Options to discuss:**
1. Static IP vs. DDNS (e.g., ddclient or external-dns)
2. Porkbun API key setup for cert-manager DNS-01 solver
3. GatewayClass + HTTPRoute + cert-manager `Certificate` resource wiring

---

## Files to Create
| Path | Purpose |
|------|---------|
| `manifests/envoy-gateway-argocd-app.yaml` | ArgoCD Application for Envoy Gateway |
| `plans/deploy-k3s.md` | Copy of this plan (per CLAUDE.md convention) |

---

## Verification Checklist
- [ ] `kubectl get nodes` shows 1 node Ready
- [ ] `kubectl get pods -A` — no crashlooping pods
- [ ] ArgoCD UI accessible at https://localhost:8080
- [ ] `kubectl get gatewayclass` shows `envoy-gateway` class
- [ ] ArgoCD shows Envoy Gateway app as Synced + Healthy
