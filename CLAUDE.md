# K3s Project Instructions for Claude

## Project Context
- **Environment:** K3s Lightweight Kubernetes
- **Purpose:** HomeLab
- **Architecture:** Single-node

## K3s Specifics & Conventions
- **Kubeconfig Location:** Usually `~/.kube/config` or `/etc/rancher/k3s/k3s.yaml` (ensure permissions are set for non-root)
- **Default Namespace:** 'default'
- **Ingress Controller:** Envoy Gateway
- **StorageClass:** Use `local-path` for local persistent volumes.

## Command Preferences
- Use `kubectl` for interaction.
- Use `k3s kubectl` if standard kubectl is not available.
- When applying manifests, prefer `kubectl apply -f`.

## Development Workflows
1. **Deploying Apps:** Create YAML files in `/manifests`, then apply them.
2. **Checking Status:** Use `kubectl get pods -A` and `kubectl logs -f <pod-name>`.
3. **Fixing Traefik:** If an app is not accessible, check `kubectl get ingress` and Traefik logs.

## Safety Rules
- NEVER delete namespaces `kube-system` or `kube-public`.
- Always verify resource names before running `kubectl delete`.
- Use `--dry-run=client -o yaml` when generating manifests.

## Prompts and Plans
- I store previous prompts in the @prompts/ directory. Before implementing an accepted plan, please save the plan to a new markdown file in @plans/ .
