# Secret Management Strategy

This document describes how secrets are handled for the `bmi-health` Kubernetes deployment, provides implementation options, and includes commands to create/update/manage secrets.

## Goals
- Keep secrets out of source control
- Ensure secrets are accessible to pods with least privilege
- Support rotation and audited updates
- Offer a path to use a dedicated secret manager in production

## Provided manifests
- `kubernetes/secrets/db-credentials.yaml` — sample Kubernetes Secret for DB credentials (placeholders).
- `kubernetes/secrets/api-keys.yaml` — sample Kubernetes Secret for API keys (placeholders).

These manifests intentionally include placeholder values. Do NOT commit real credentials. Prefer creating secrets dynamically or using sealed/encrypted secrets for repo storage.

## Options / Recommended approaches

1. Kubernetes Secrets (simple, built-in)
   - Pros: built-in, easy to use, supported by kubectl and manifests
   - Cons: stored (base64) in etcd; ensure etcd encryption at rest and RBAC is configured

2. Sealed Secrets (Bitnami kubeseal)
   - Use `kubeseal` to produce an encrypted `SealedSecret` that is safe to store in Git. The controller in-cluster decrypts it.
   - Pros: safe to store in repo, GitOps-friendly
   - Cons: requires installing the `sealed-secrets` controller

3. External Secret Manager (recommended for production)
   - Examples: HashiCorp Vault, AWS Secrets Manager, Azure Key Vault, Google Secret Manager
   - Use `external-secrets` or native integrations to sync secrets into Kubernetes Secrets at runtime.
   - Pros: strong rotation, auditing, fine-grained access controls

## How to create secrets (examples)

1) Create from literals (imperative):

```bash
kubectl create secret generic db-credentials \
  --from-literal=POSTGRES_USER=bmi_user \
  --from-literal=POSTGRES_PASSWORD='s3cr3t' \
  --from-literal=POSTGRES_DB=bmidb \
  -n bmi-health

kubectl create secret generic api-keys \
  --from-literal=THIRD_PARTY_SERVICE_KEY='replace_me' \
  --from-literal=ANALYTICS_API_KEY='replace_me' \
  -n bmi-health
```

2) Create from YAML manifest (declarative): edit `kubernetes/secrets/db-credentials.yaml` replacing placeholders, then:

```bash
kubectl apply -f kubernetes/secrets/db-credentials.yaml
kubectl apply -f kubernetes/secrets/api-keys.yaml
```

3) Create a secret YAML without exposing values in shell history:

```bash
kubectl create secret generic db-credentials \
  --from-literal=POSTGRES_USER=bmi_user \
  --from-literal=POSTGRES_PASSWORD='s3cr3t' \
  --from-literal=POSTGRES_DB=bmidb \
  -n bmi-health --dry-run=client -o yaml > kubernetes/secrets/db-credentials.generated.yaml

# Inspect then apply
kubectl apply -f kubernetes/secrets/db-credentials.generated.yaml
```

## Using Sealed Secrets (example)

1. Install `kubeseal` controller in cluster following https://github.com/bitnami-labs/sealed-secrets.
2. Create a secret locally and seal it:

```bash
# create secret YAML without committing raw secret
kubectl create secret generic db-credentials \
  --from-literal=POSTGRES_USER=bmi_user \
  --from-literal=POSTGRES_PASSWORD='s3cr3t' \
  --from-literal=POSTGRES_DB=bmidb \
  -n bmi-health --dry-run=client -o yaml > db-credentials.raw.yaml

# seal it using kubeseal (requires kube context)
kubeseal --format yaml < db-credentials.raw.yaml > kubernetes/secrets/db-credentials.sealed.yaml

# commit the sealed file safely to repo. Controller will decrypt in-cluster.
kubectl apply -f kubernetes/secrets/db-credentials.sealed.yaml
```

## How applications access secrets

- Environment variables: use `envFrom.secretRef` or `env` entries that reference secrets in the Deployment spec.
- Volume mount: mount secret as files into the pod filesystem (useful for certificates or multi-line values).

Example (envFrom) in a Deployment:

```yaml
envFrom:
  - secretRef:
      name: db-credentials
```

Or single env var:

```yaml
env:
  - name: POSTGRES_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: POSTGRES_PASSWORD
```

## Rotating and updating secrets

- Update secret using `kubectl create secret --dry-run=client -o yaml` and `kubectl apply -f`.
- For runtime rotation, update the secret and trigger rolling restart of Deployments:

```bash
kubectl rollout restart deployment/backend -n bmi-health
kubectl rollout restart deployment/frontend -n bmi-health
```

If using external secret managers, configure automatic rotation sync and test application behavior on secret changes.

## RBAC, encryption, and auditing

- Enable encryption at rest for etcd (cloud providers may have this enabled by default).
- Restrict who can `get`/`create`/`update` Secrets with Kubernetes RBAC.
- Audit secret access via cloud provider or cluster audit logs.

## Recommendations

- For development: Kubernetes Secrets are acceptable but avoid committing real values.
- For production: use Sealed Secrets (GitOps-friendly) or an external secrets manager (Vault, AWS Secrets Manager) with `external-secrets` operator.
- Automate secret creation in CI/CD pipelines using secure variables (Actions secrets, GitLab CI variables) and avoid storing plaintext credentials in the repository.

## Applying the provided manifests

1. Edit the files in `kubernetes/secrets/` to replace placeholders, or create secrets dynamically using `kubectl create secret` commands above.
2. Apply them:

```bash
kubectl apply -f kubernetes/secrets/db-credentials.yaml
kubectl apply -f kubernetes/secrets/api-keys.yaml
```

3. Restart deployments to pick up changes (if necessary):

```bash
kubectl rollout restart deployment/backend -n bmi-health
kubectl rollout restart deployment/frontend -n bmi-health
```

If you'd like, I can:
- Produce `SealedSecret` manifests using a cluster public key (if you provide access), or
- Add `external-secrets` integration examples for Vault/GCP/AWS.