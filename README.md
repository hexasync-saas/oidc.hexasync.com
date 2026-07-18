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

## State of the committed JSON

- `rke2-onprem/openid/v1/jwks` — **REAL public signing key** dumped from the bhs01 (OVH
  Beauharnois) RKE2 API server (`kid=fuXUikIcrkKUuKb_mKYGRbYgwzCxG04O3kVt2PEn1IE`, RS256). Public,
  non-secret; stable unless the cluster's SA signing keys are rotated.
- `rke2-onprem/.well-known/openid-configuration` — carries the **target** issuer
  `https://oidc.hexasync.com/rke2-onprem`. This is exactly what the API server will emit **once the
  issuer cutover below is applied**.

> ⚠️ **Not live yet.** The bhs01 API server currently issues SA tokens with
> `iss=https://kubernetes.default.svc.cluster.local`. Entra federation only works after the
> apiserver is reconfigured to issue `iss=https://oidc.hexasync.com/rke2-onprem` **and** DNS +
> hosting for `oidc.hexasync.com` point at a host that serves these files over a CA-valid cert.
> Today `oidc.hexasync.com` → `20.120.196.31` (an nginx k8s ingress serving a fake cert, 404 on the
> OIDC path). Both are production-impacting cutover steps — see below.

### Re-dump from the cluster (after the issuer cutover)

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

## Go-live checklist (two production cutover steps remain)

The repo + Pages are wired and **already serving** at GitHub's edge (verified via
`--resolve oidc.hexasync.com:443:185.199.108.153`). What's left is production-impacting:

1. **DNS (Cloudflare, zone `hexasync.com`).** Today `oidc.hexasync.com` → `20.120.196.31` (an Azure
   nginx k8s ingress, fake cert, 404 on the OIDC path). Repoint it:
   - Delete the current `oidc` A/CNAME record.
   - Add `CNAME  oidc  →  hexasync-saas.github.io`  (**DNS-only / grey cloud** — GitHub must serve
     the cert and terminate TLS; Cloudflare proxy would break Pages cert issuance).
   - GitHub → repo Settings → Pages → Custom domain `oidc.hexasync.com` → wait for the green check →
     tick **Enforce HTTPS** (Let's Encrypt provisions in a few minutes).
2. **RKE2 issuer cutover** — see [`docs/rke2-issuer-cutover.md`](docs/rke2-issuer-cutover.md).
   Restarts the bhs01 control plane; run in a maintenance window. Prepared, **not applied**.

### Pages source (already configured)
- Settings → Pages → Source: **Deploy from branch** `main` / `(root)`.
- Custom domain `oidc.hexasync.com` set via the `CNAME` file.

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
