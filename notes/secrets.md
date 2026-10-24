# Secrets

## Why Secrets

ConfigMaps are for **non-confidential** config. Passwords, API keys, tokens and certificates go in a **Secret**.

| | ConfigMap | Secret |
|---|---|---|
| For | Non-sensitive config | Sensitive data |
| Stored as | Plain text | **Base64 encoded** |
| Used as | Env vars or mounted files | Env vars or mounted files |
| Access | Anyone who can read the namespace's ConfigMaps | Can be locked down separately with RBAC |

- Base64 is **encoding, not encryption**. It just turns binary data into text. Anyone can decode it.
- Kubernetes decodes the value before giving it to the pod, so the app sees the original value.

---

## Creating a Secret

### Imperative
```bash
kubectl create secret generic my-secret \
  --from-literal=username=my-user \
  --from-literal=password=my-password
```
`generic` = an opaque key-value secret. Other types: `tls` (cert + key), `docker-registry` (image pull credentials).

### Declarative
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  username: bXktdXNlcg==          # base64 of "my-user"
  password: bXktcGFzc3dvcmQ=      # base64 of "my-password"
```
Use `stringData:` instead of `data:` to write plain values and let Kubernetes encode them.

### Inspect and decode
```bash
kubectl get secret my-secret -o yaml            # values shown base64 encoded
echo 'bXktcGFzc3dvcmQ=' | base64 -d             # my-password
kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 -d
```

---

## Using a Secret in a pod

Same two methods as ConfigMaps. `configMapKeyRef` becomes `secretKeyRef`, and `configMap:` becomes `secret:` in volumes.

| Method | Good for | Picks up changes? |
|---|---|---|
| Env vars (`secretKeyRef`) | Apps that expect config in the environment | No, restart the pod |
| Volume mount | Apps that read a file (config file, certificate) | Yes, after a short delay |

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-demo-pod
spec:
  containers:
  - name: secret-demo-container
    image: nginx
    env:
    - name: SECRET_USERNAME
      valueFrom:
        secretKeyRef:
          name: my-secret          # which Secret
          key: username            # which key in it
    - name: SECRET_PASSWORD
      valueFrom:
        secretKeyRef:
          name: my-secret
          key: password
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secret-volume   # each key becomes a file here
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: my-secret        # note: secretName, not name
```

> **Gotcha:** `did not find expected key` on apply almost always means **bad indentation**. `env` and `volumeMounts` sit at the same level as `image`, inside the container.

### Verify inside the pod
```bash
kubectl apply -f secret-pod.yaml
kubectl exec -it secret-demo-pod -- /bin/sh

echo $SECRET_USERNAME               # my-user
echo $SECRET_PASSWORD               # my-password
cat /etc/secret-volume/username     # my-user
cat /etc/secret-volume/password     # my-password
```

---

## Why Secrets aren't secure by default

| Problem | Detail |
|---|---|
| Only base64 | Anyone with cluster access decodes them with one command |
| etcd | Stored as base64 in etcd. Managed clusters (EKS, GKE, AKS) encrypt the **disk**, but inside etcd it's still just base64 |
| RBAC | Anyone with `get secrets` in the namespace can read them all. Often far too broad |
| Leaks | Values end up in logs, CI/CD pipelines, or committed to Git in a YAML file |

### What to do about it
- **Encryption at rest with KMS v2** (stable since 1.29). Encrypts secrets with your cloud provider's KMS **before** they hit etcd. Defence in depth on top of disk encryption.
- **Tight RBAC.** Not everyone needs every secret.
- **Never store plain Secrets in Git.** Use Sealed Secrets or ESO.
- **Use a real secrets manager:** Vault, AWS Secrets Manager, Azure Key Vault, GCP Secret Manager.

> Kubernetes gives you the primitives. You build the security around them.

---

## External Secrets Operator (ESO)

Keep secrets in a **dedicated secrets manager** (the source of truth). ESO syncs them into the cluster as normal Kubernetes Secrets.

Why better: the secrets manager gives you proper encryption, audit logs, rotation and access control. Kubernetes only holds a synced copy.

| Resource | Does |
|---|---|
| `SecretStore` / `ClusterSecretStore` | The bridge: how ESO connects and authenticates to the provider (credentials or IAM role). Namespaced / cluster-wide |
| `ExternalSecret` | Which external secret to fetch and which Kubernetes Secret to create from it |

### The flow
```
Secrets manager  <-  ESO watches ExternalSecret, fetches value  ->  creates/updates K8s Secret  ->  pod uses it as normal
```

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: db-creds
spec:
  refreshInterval: 1h              # how often to re-sync
  secretStoreRef:
    name: aws-store
    kind: SecretStore
  target:
    name: db-creds                 # the K8s Secret ESO creates
  data:
  - secretKey: password            # key in the K8s Secret
    remoteRef:
      key: prod/db                 # secret in AWS Secrets Manager
      property: password
```

- Pods don't know where the secret came from. To them it's a regular Secret.
- **Separate lanes:** security team manages the secrets manager, platform team configures ESO and stores, DevOps just references Secrets in pod specs.
- No passwords in Slack, Teams, email or Git. Controlled, auditable path the whole way.
- **Rotation:** update at the source, ESO syncs it. Env vars still need a pod restart; volume mounts update after a short delay. Tools like **Reloader** trigger rolling restarts when a Secret changes.

---

## Sealed Secrets

Solves the **GitOps problem**: the whole cluster state lives in Git, but a plain Secret in Git is readable by anyone with repo access.

Sealed Secrets: encrypt the Secret locally, commit the encrypted **SealedSecret** to Git. Only the cluster can decrypt it. You keep version control, code review and GitOps.

### How it works
- A **controller** in the cluster generates a public/private key pair. The private key never leaves the cluster.
- Keys **renew every 30 days** by default. Old keys are kept so existing SealedSecrets still decrypt.
- Locally, **kubeseal** encrypts a normal Secret with the cluster's public key.
- When the SealedSecret is applied, the controller decrypts it and creates the real Secret.

```bash
kubectl create secret generic my-secret \
  --from-literal=password=my-password \
  --dry-run=client -o yaml \
  | kubeseal --format yaml > sealed-secret.yaml   # safe to commit
```

### Scopes
| Scope | Rule |
|---|---|
| `strict` (default) | Tied to the Secret's **name and namespace**. Most secure, use unless you have a reason not to |
| `namespace-wide` | Can be renamed within the same namespace |
| `cluster-wide` | Can be used in any namespace with any name |

> **Gotcha:** lose the cluster's private key without a backup and your SealedSecrets are **unrecoverable**. Back up the key.

---

## Sealed Secrets vs ESO

| | Sealed Secrets | ESO |
|---|---|---|
| Source of truth | Git | Secrets manager |
| External dependency | None, works offline | Needs connectivity to the provider |
| Rotation | Reseal and commit again | Automatic via refresh interval |
| Multi-cluster | Hard, each cluster has its own keys | Centralised across clusters |
| Moving parts | Few | More |
| Best for | Small teams doing GitOps, no secrets manager yet | Enterprises with Vault / AWS Secrets Manager already running |

**Both together:** use ESO for app secrets, and seal ESO's own bootstrap credentials (the ones it uses to reach Vault) so even those live safely in Git.

---
