# Home Assistant Deployment Plan

Deploy Home Assistant to k3s via ArgoCD using raw manifests. All files go in `manifests/` and are auto-synced by the existing `meerkat` ArgoCD Application — no new ArgoCD Application needed.

## Routing Decision

Home Assistant cannot serve from a subpath (its frontend uses absolute asset paths). It is routed at `/` on `longwood.daphnai.com`. The existing `/argocd` HTTPRoute has a longer prefix and takes precedence, so ArgoCD is unaffected.

Google OIDC (SecurityPolicy on the `longwood` Gateway) applies to Home Assistant. The HA mobile app will not work through the OIDC redirect — this is acceptable for homelab use.

## Files Created

| File | Kind | Namespace |
|---|---|---|
| `manifests/home-assistant-namespace.yaml` | Namespace | — |
| `manifests/home-assistant-pvc.yaml` | PersistentVolumeClaim (10Gi, local-path) | `home-assistant` |
| `manifests/home-assistant-service.yaml` | Service (ClusterIP, port 8123) | `home-assistant` |
| `manifests/home-assistant-deployment.yaml` | Deployment | `home-assistant` |
| `manifests/home-assistant-referencegrant.yaml` | ReferenceGrant | `home-assistant` |
| `manifests/home-assistant-httproute.yaml` | HTTPRoute (`/` → home-assistant) | `default` |

## Deployment Notes

- `hostNetwork: true` + `privileged: true` for mDNS discovery and device access (Zigbee, Z-Wave, Bluetooth)
- `dnsPolicy: ClusterFirstWithHostNet` required alongside `hostNetwork: true`
- dbus (`/run/dbus`) and localtime (`/etc/localtime`) mounted from host
- Image: `ghcr.io/home-assistant/home-assistant:stable`

## Post-Deployment

After first-time onboarding at `https://longwood.daphnai.com`, add to `/config/configuration.yaml`:

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 10.42.0.0/16
```

This prevents HA from banning the Envoy Gateway proxy IP.
