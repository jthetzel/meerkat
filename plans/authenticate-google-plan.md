# Plan: Google OIDC Authentication via Envoy Gateway SecurityPolicy

## Goal
Protect all routes on the `longwood` Gateway behind Google authentication.
Users must sign in with a Google account before any backend service is reachable.

## Approach
Envoy Gateway v1.7.2 (already deployed) has native OIDC support via the
`SecurityPolicy` CRD. A single `SecurityPolicy` targeting the `longwood`
Gateway will enforce Google OAuth2/OIDC on every attached HTTPRoute.

> **Note on "Identity Aware Proxy":** True Google Cloud IAP requires traffic to
> flow through GCP's managed load balancer infrastructure. For a K3s homelab
> reachable directly on the internet, the equivalent is configuring Envoy
> Gateway to use Google as an OIDC/OAuth2 provider — which delivers the same
> "sign in with Google before you can reach anything" guarantee.

---

## Step 1 — GCP: Enable APIs and Create OAuth2 Credentials

Run these commands (or use the GCP Console):

```bash
# Enable required APIs
gcloud services enable iap.googleapis.com --project daphnai
gcloud services enable oauth2.googleapis.com --project daphnai

# Configure OAuth consent screen (Internal = only your Google Workspace users;
# External = any Google account — choose External for a personal homelab)
# This step must be done in the Console:
# https://console.cloud.google.com/apis/credentials/consent?project=daphnai

# Create a Web Application OAuth2 client
gcloud alpha iap oauth-brands list --project daphnai   # get the brand name first
# Then create client via Console or gcloud; add authorized redirect URI:
#   https://longwood.daphnai.com/oauth2/callback
```

After creation, note the **Client ID** and **Client Secret**.

---

## Step 2 — Create `manifests/google-oauth-secret.yaml` (gitignored)

This file holds the OAuth2 client secret and must NOT be committed.

```yaml
# manifests/google-oauth-secret.yaml  — DO NOT COMMIT
apiVersion: v1
kind: Secret
metadata:
  name: google-oauth-secret
  namespace: default
type: Opaque
stringData:
  client-secret: "<REPLACE_WITH_CLIENT_SECRET>"
```

Apply it directly:
```bash
kubectl apply -f manifests/google-oauth-secret.yaml
```

---

## Step 3 — Create `manifests/google-oidc-security-policy.yaml`

The `SecurityPolicy` targets the `longwood` `Gateway`, so it covers all
HTTPRoutes (including `argocd` and any future routes) automatically.

```yaml
# manifests/google-oidc-security-policy.yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: SecurityPolicy
metadata:
  name: google-oidc
  namespace: default
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: longwood
  oidc:
    provider:
      issuer: "https://accounts.google.com"
    clientID: "<REPLACE_WITH_CLIENT_ID>"   # not secret; safe to commit
    clientSecret:
      name: google-oauth-secret
      key: client-secret
    redirectURL: "https://longwood.daphnai.com/oauth2/callback"
    scopes:
      - openid
      - email
      - profile
    cookieConfig:
      sameSite: Strict
```

---

## Step 4 — Update `.gitignore`

Add the secret file so it is never accidentally committed:

```
manifests/google-oauth-secret.yaml
```

---

## Step 5 — Apply and Verify

```bash
# Apply the SecurityPolicy (secret was already applied in Step 2)
kubectl apply -f manifests/google-oidc-security-policy.yaml

# Watch the SecurityPolicy reconcile
kubectl get securitypolicy google-oidc -n default -w

# Check Envoy Gateway controller logs for OIDC errors
kubectl logs -n envoy-gateway-system -l app.kubernetes.io/name=envoy-gateway --tail=50

# Test: open https://longwood.daphnai.com/argocd — you should be redirected to
# Google's sign-in page before ArgoCD loads.
```

---

## File Summary

| File | Action | Committed? |
|------|--------|------------|
| `manifests/google-oauth-secret.yaml` | Create (Step 2) | No — gitignored |
| `manifests/google-oidc-security-policy.yaml` | Create (Step 3) | Yes |
| `.gitignore` | Update (Step 4) | Yes |

---

## Considerations & Caveats

- **Allowed Google accounts:** By default any Google account can pass
  authentication. To restrict to specific emails or a domain, add a
  `jwt` claim filter or use the OAuth consent screen's "Internal" mode
  (Google Workspace only). Envoy Gateway's `SecurityPolicy` also supports
  `authorization` rules — a follow-up step could add
  `allowedPrincipals` to restrict to `jthetzel@gmail.com`.

- **ArgoCD already has its own auth:** ArgoCD's own login sits behind this
  OIDC layer. The double-auth may be redundant but is harmless. You could
  configure ArgoCD to use the same Google OIDC SSO so both layers use the
  same session.

- **Cookie / token storage:** Envoy Gateway stores OIDC session tokens
  in an encrypted browser cookie. The `cookieConfig.sameSite: Strict`
  setting reduces CSRF risk.

- **No new ArgoCD Application needed:** The `SecurityPolicy` manifest is a
  single YAML file applied directly, consistent with how other routing
  resources (Gateway, HTTPRoute, ReferenceGrant) are managed in this repo.
  If you later want ArgoCD to manage the whole `manifests/` directory, an
  App-of-Apps Application pointing at this repo can be added then.
