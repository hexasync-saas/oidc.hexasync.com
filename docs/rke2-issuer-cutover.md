# RKE2 SA-issuer cutover runbook — bhs01 (OVH Beauharnois)

**Status: PREPARED, NOT APPLIED.** This restarts the production control plane and changes the
cluster-wide ServiceAccount token issuer. Run only in a maintenance window, per-node, with a human
watching. `kubectl` is not enough — you need **root SSH on each control-plane node**.

## Why

Azure Entra ID validates the on-prem RKE2 federated SA token by fetching
`{iss}/.well-known/openid-configuration` **over the public internet**. Today the apiserver issues:

```
--service-account-issuer=https://kubernetes.default.svc.cluster.local   # not publicly resolvable
--api-audiences=https://kubernetes.default.svc.cluster.local,rke2
--service-account-key-file=/var/lib/rancher/rke2/server/tls/service.key
--service-account-signing-key-file=/var/lib/rancher/rke2/server/tls/service.current.key
```

Entra cannot reach `kubernetes.default.svc.cluster.local`, so federation fails until the primary
issuer becomes `https://oidc.hexasync.com/rke2-onprem`.

## Prerequisites (do these FIRST, they are non-disruptive)

1. **Host is live.** `oidc.hexasync.com` serves the two JSON files over a CA-valid HTTPS cert with
   anonymous read. (This repo on GitHub Pages once DNS → `hexasync-saas.github.io` + Enforce HTTPS.)
   Verify off-cluster:
   ```bash
   curl -s https://oidc.hexasync.com/rke2-onprem/.well-known/openid-configuration | jq .
   curl -s https://oidc.hexasync.com/rke2-onprem/openid/v1/jwks | jq .
   ```
   The hosted `jwks` already contains the real bhs01 signing key
   (`kid=fuXUikIcrkKUuKb_mKYGRbYgwzCxG04O3kVt2PEn1IE`), so signature validation will pass the moment
   the issuer flips — **no re-dump needed** unless the cluster's SA signing keys are later rotated.
2. **Discovery RBAC** (lets anonymous callers read discovery for the re-dump/verify step):
   ```bash
   kubectl create clusterrolebinding sa-issuer-discovery \
     --clusterrole=system:service-account-issuer-discovery --group=system:authenticated
   ```

## Cutover (per control-plane node, ONE at a time)

Order: `dbcoord001` (192.168.1.201) → `dbcoord002` (192.168.1.202) → `worker001` (192.168.1.1).
Wait for the node's apiserver to be healthy before moving to the next.

On each node:

```bash
# 1) back up
sudo cp /etc/rancher/rke2/config.yaml /etc/rancher/rke2/config.yaml.bak.$(date +%s)

# 2) append the kube-apiserver-arg block from apiserver-config.fragment.yaml
#    (merge into any existing kube-apiserver-arg: list — do NOT duplicate the key)
sudo vi /etc/rancher/rke2/config.yaml

# 3) restart the server component
sudo systemctl restart rke2-server

# 4) wait for the local apiserver to come back
until sudo /var/lib/rancher/rke2/bin/kubectl --kubeconfig /etc/rancher/rke2/rke2.yaml \
      get --raw /readyz >/dev/null 2>&1; do echo waiting; sleep 5; done; echo "apiserver ready"
```

## Verify (after all three nodes)

```bash
# discovery issuer MUST now be the public URL (this is the make-or-break check)
kubectl get --raw /.well-known/openid-configuration | jq -r .issuer
#   expected: https://oidc.hexasync.com/rke2-onprem
kubectl get --raw /.well-known/openid-configuration | jq -r .jwks_uri
#   expected: https://oidc.hexasync.com/rke2-onprem/openid/v1/jwks
```

If `issuer` still shows the cluster URL, RKE2's default won the primary slot. Fix by removing the
RKE2 default explicitly — set only your two `service-account-issuer` args and, if needed, use
`kube-apiserver-arg+:` is not supported for de-dupe, so pin via the flag ordering RKE2 documents for
your version, then restart and re-verify.

Then re-dump and re-commit the discovery doc so the hosted copy is byte-identical to the apiserver:

```bash
kubectl get --raw /.well-known/openid-configuration > rke2-onprem/.well-known/openid-configuration
kubectl get --raw /openid/v1/jwks                    > rke2-onprem/openid/v1/jwks
git add -A && git commit -m "chore: re-dump discovery post-cutover" && git push
```

Finally run the app-side gate (from an in-cluster pod, while `KEK_ACTIVE=Local` is still
authoritative):

```
POST /api/v3/admin/dataprotection/self-test?provider=AzureKeyVault   → ok=true
```

## Rollback

Per node, in reverse: restore the backup and restart.

```bash
sudo cp /etc/rancher/rke2/config.yaml.bak.<ts> /etc/rancher/rke2/config.yaml
sudo systemctl restart rke2-server
```

Because the legacy issuer is kept as the secondary `service-account-issuer` during cutover,
existing tokens keep validating throughout, making rollback low-risk.

## Blast-radius notes

- New SA tokens cluster-wide will carry `iss=https://oidc.hexasync.com/rke2-onprem`. In-cluster
  TokenReview goes through the apiserver, which accepts BOTH issuers — safe.
- Risk lives in any **external** validator/webhook that hardcodes the old issuer string. Audit
  before flipping if such integrations exist.
- SA signing keys are unchanged by this — only the issuer/jwks_uri strings change. The hosted JWKS
  stays valid.
