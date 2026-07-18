# oidc.hexasync.com

Public, **read-only** OIDC issuer for on-prem RKE2 clusters, served via **GitHub Pages** at
`https://oidc.hexasync.com`.

It hosts nothing but small, **non-secret** JSON metadata (OIDC discovery + JWKS public keys) so
**Entra ID** can validate projected ServiceAccount tokens during Workload Identity federation. The
Kubernetes API server itself stays private — this host is only an *identity + metadata location*,
never a live endpoint into any cluster.

> Origin design doc: *"AKS & RKE2 → Azure Key Vault via Workload Identity"* — HexaSync Data
> Protection, story 5.13 Part F.

## Why this exists

On AKS the managed OIDC issuer + workload-identity webhook come for free. On on-prem **RKE2** you
provide them yourself. Entra ID validates the federated SA token by fetching this cluster's OIDC
**discovery + JWKS over the public internet**, so the non-secret issuer metadata must be publicly
reachable over HTTPS with a CA-valid cert.

## Layout

```
oidc.hexasync.com/
├── .nojekyll                                       # REQUIRED — keeps Pages from dropping .well-known/
├── CNAME                                           # oidc.hexasync.com
└── rke2-onprem/                                    # one folder per cluster (add /aks-dr, … as needed)
    ├── .well-known/openid-configuration            # {issuer}/.well-known/openid-configuration
    └── openid/v1/jwks                              # {issuer}/openid/v1/jwks  (== jwks_uri)
```

The `/rke2-onprem` path segment **is** the issuer:
`service-account-issuer=https://oidc.hexasync.com/rke2-onprem`.

## ⚠️ The committed JSON is a PLACEHOLDER

`rke2-onprem/openid/v1/jwks` ships with `REPLACE_WITH_REAL_*` values. Replace both files with the
**verbatim dump from the live RKE2 API server**, so the `issuer` / `jwks_uri` baked inside and the
real signing keys match the cluster exactly.

### Regenerate from the cluster

```bash
# 0) apiserver args must already point at this host (then: systemctl restart rke2-server):
#    /etc/rancher/rke2/config.yaml
#      kube-apiserver-arg:
#        - "service-account-issuer=https://oidc.hexasync.com/rke2-onprem"
#        - "service-account-jwks-uri=https://oidc.hexasync.com/rke2-onprem/openid/v1/jwks"

# 1) one-time: allow anonymous discovery read (RBAC)
kubectl create clusterrolebinding sa-issuer-discovery \
  --clusterrole=system:service-account-issuer-discovery --group=system:authenticated

# 2) dump the two files — THIS is the content you host (non-secret)
kubectl get --raw /.well-known/openid-configuration > rke2-onprem/.well-known/openid-configuration
kubectl get --raw /openid/v1/jwks                    > rke2-onprem/openid/v1/jwks

# 3) sanity — must match the apiserver args byte-for-byte
grep -o '"issuer":"[^"]*"'   rke2-onprem/.well-known/openid-configuration
grep -o '"jwks_uri":"[^"]*"' rke2-onprem/.well-known/openid-configuration

git add -A && git commit -m "chore: publish rke2-onprem OIDC discovery + JWKS from apiserver" && git push
```

## GitHub Pages setup

- Settings → Pages → Source: **Deploy from branch** `main` / `(root)`.
- Settings → Pages → Custom domain: `oidc.hexasync.com` → tick **Enforce HTTPS**.
- DNS → `CNAME  oidc.hexasync.com → hexasync-saas.github.io`.

### Gotchas

- **`.nojekyll` is mandatory.** Without it Pages runs Jekyll, which silently drops any dot-folder —
  `.well-known/` would 404. This is the #1 failure mode.
- **Content-Type.** These files have no extension, so Pages serves them as
  `application/octet-stream` and gives no way to set headers. Entra usually parses the body anyway;
  if your verifier is strict about `application/json`, move to a host with header control
  (Cloudflare Pages `_headers`, Netlify, or Azure Blob / S3 with the content-type set).
- **Re-publish JWKS on key rotation.** If the API server's SA signing keys rotate, re-dump and
  re-commit `jwks` (and `openid-configuration` if it changed) or signature validation fails. RKE2
  SA keys are static unless explicitly rotated.

## Verify (exactly what Entra does — run OFF the cluster)

```bash
curl -s https://oidc.hexasync.com/rke2-onprem/.well-known/openid-configuration | jq .
curl -s https://oidc.hexasync.com/rke2-onprem/openid/v1/jwks | jq .
```

`issuer` / `jwks_uri` must match the apiserver args, and `jwks` must return the real signing key(s).

## Security

Everything served here is **non-secret** public key material and metadata. Never commit the API
server, kubeconfig, client secrets, or any private key. The repo is public by design.
