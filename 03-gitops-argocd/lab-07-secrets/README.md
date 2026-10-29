# Lab 03-07 - Secrets in GitOps: External Secrets Operator, Sealed Secrets, SOPS

## Goal
Run the three patterns, break each, and be able to choose. Primary pattern: External Secrets Operator (ESO) backed by a real secret store (HashiCorp Vault dev mode locally; Azure Key Vault/AWS SM mapping shown).

## Prereqs
hub or dev cluster; `helm`, `kubectl`, `age`, `sops`, `kubeseal`.

## Part A - External Secrets Operator + Vault
```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -n external-secrets --create-namespace --set installCRDs=true
kubectl -n external-secrets rollout status deploy/external-secrets
```
Dev Vault (NOT for production):
```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault -n vault --create-namespace --set server.dev.enabled=true --set server.dev.devRootToken=root --set injector.enabled=false
kubectl -n vault exec vault-0 -- vault kv put secret/payments/dev db_password='S3cr3t!' api_key='abc123'
kubectl -n payments-dev create secret generic vault-token --from-literal=token=root
```
SecretStore (namespaced) and ExternalSecret:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata: { name: vault, namespace: payments-dev }
spec:
  provider:
    vault:
      server: http://vault.vault:8200
      path: secret
      version: v2
      auth: { tokenSecretRef: { name: vault-token, key: token } }
---
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata: { name: payments-db, namespace: payments-dev }
spec:
  refreshInterval: 1h
  secretStoreRef: { name: vault, kind: SecretStore }
  target: { name: payments-db, creationPolicy: Owner }
  data:
    - { secretKey: DB_PASSWORD, remoteRef: { key: payments/dev, property: db_password } }
EOF
kubectl -n payments-dev get externalsecret payments-db
kubectl -n payments-dev get secret payments-db -o jsonpath='{.data.DB_PASSWORD}' | base64 -d; echo
```
Expected: `STATUS SecretSynced`, `READY True`. Git contains only the ExternalSecret (a reference), never the value.
Production mapping:
- **Azure Key Vault**: `provider.azurekv` with `authType: WorkloadIdentity`, `vaultUrl`, SA annotated with the managed identity client ID (Module 04 workload identity).
- **AWS Secrets Manager**: `provider.aws.service: SecretsManager`, auth via IRSA `jwt.serviceAccountRef`.
Use `ClusterSecretStore` only with `conditions` restricting namespaces, or per-tenant `SecretStore`, to prevent one team reading another team's secrets. Rotation: change in the store, ESO refreshes at `refreshInterval`; pods need a reload (Stakater Reloader or checksum annotations).

## Part B - Sealed Secrets (encrypt in git, controller decrypts)
```bash
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets -n kube-system
kubectl -n payments-dev create secret generic app-token --from-literal=token=hello --dry-run=client -o yaml \
 | kubeseal --controller-name sealed-secrets --controller-namespace kube-system --format yaml > apps/payments/overlays/dev/app-token.sealed.yaml
cat apps/payments/overlays/dev/app-token.sealed.yaml | head
kubectl apply -f apps/payments/overlays/dev/app-token.sealed.yaml
kubectl -n payments-dev get secret app-token
```
Pros: simple, no external store. Cons: cluster-specific key (DR: back up the sealing key; new cluster cannot decrypt), rotation is manual re-seal, secrets live in git forever (history), no central audit. Back up: `kubectl -n kube-system get secret -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > sealing-key.backup.yaml` and store it in a vault.

## Part C - SOPS + age (encrypt files in git)
```bash
age-keygen -o age.key && cat age.key | grep public
cat > .sops.yaml <<'EOF'
creation_rules:
  - path_regex: .*\.enc\.yaml$
    encrypted_regex: ^(data|stringData)$
    age: age1REPLACE_WITH_PUBLIC_KEY
EOF
cat > secret.enc.yaml <<'EOF'
apiVersion: v1
kind: Secret
metadata: { name: app-sops, namespace: payments-dev }
stringData: { token: hello }
EOF
sops --encrypt --in-place secret.enc.yaml && head secret.enc.yaml
SOPS_AGE_KEY_FILE=age.key sops --decrypt secret.enc.yaml
```
With ArgoCD: use a plugin/KSOPS (kustomize exec plugin) or Helm secrets, with the age/KMS key mounted in the repo-server (Azure Key Vault/AWS KMS keys instead of age in production). Pros: works offline, KMS-backed, diffable keys. Cons: repo-server needs decrypt access (bigger blast radius), plugin maintenance.

## Decision table
| | ESO | Sealed Secrets | SOPS |
|---|---|---|---|
| Source of truth | external store (KV/SM/Vault) | git (encrypted) | git (encrypted) |
| Rotation | central, automatic | manual | manual |
| Audit | store audit logs | weak | KMS audit |
| DR | store is separate | must back up key | KMS key |
| Fits | enterprise, multi-cluster | small teams, simple | repo-centric, Helm/Kustomize |
My default: ESO + cloud secret manager + workload identity. Secrets are *never* created by CI as plain Kubernetes Secrets.

## Break it on purpose
1. Delete the Vault key; ESO `SecretSyncedError`; the existing Secret remains (stale but available): discuss `creationPolicy` and alerting on `externalsecret_status_condition`.
2. Re-create the cluster without the sealing key: sealed secret fails to decrypt (`no key could decrypt secret`); restore key backup.
3. Commit a plain Secret by accident: add a Kyverno/pre-commit rule that rejects `kind: Secret` with `data` in the gitops repo (gitleaks in CI).

## Cleanup
```bash
helm uninstall vault -n vault; helm uninstall sealed-secrets -n kube-system; helm uninstall external-secrets -n external-secrets
```

## Talking points
References not values; identity-based access to the store; per-tenant stores; rotation + reloads; DR for keys; secret scanning guardrails.
