# Kubernetes Interview Mastery
# CATEGORY 5: CONFIGURATION & SECRETS

---

> **How to use this document:**
> Each topic: Simple Explanation → Why It Exists → Internal Working → YAML/Commands → Short Answer → Deep Answer → Gotchas → Interview Q&A → Connections.
> ⚠️ = High priority, frequently asked, commonly misunderstood.

---

## Table of Contents

| # | Topic | Level |
|---|-------|-------|
| 5.1 | ConfigMaps — injecting config into pods | 🟢 Beginner |
| 5.2 | Secrets — base64, opaque, TLS, docker-registry types ⚠️ | 🟢 Beginner |
| 5.3 | Secret encryption at rest (EncryptionConfiguration) ⚠️ | 🟡 Intermediate |
| 5.4 | External Secrets Operator — pulling from Vault/AWS SM | 🟡 Intermediate |
| 5.5 | Projected Volumes — combining secrets/configmaps/SA tokens | 🟡 Intermediate |
| 5.6 | Immutable ConfigMaps/Secrets | 🟡 Intermediate |
| 5.7 | Sealed Secrets — GitOps-safe secret management | 🔴 Advanced |
| 5.8 | CSI Secret Store driver — secrets as volumes | 🔴 Advanced |
| 5.9 | Secret rotation & pod reload strategies | 🔴 Advanced |

---

## Difficulty Legend
- 🟢 **Beginner** — Expected from ALL candidates
- 🟡 **Intermediate** — Expected from 3+ year engineers
- 🔴 **Advanced** — Differentiates senior/staff candidates
- ⚠️ **High Priority** — Frequently asked / commonly misunderstood

---

# 5.1 ConfigMaps — Injecting Config Into Pods

## 🟢 Beginner

### What it is in simple terms

A ConfigMap is a **Kubernetes object that stores non-sensitive configuration data as key-value pairs**. It decouples configuration from container images — the same image runs in dev with dev config and in prod with prod config. This is the twelve-factor app principle applied to Kubernetes.

---

### Why ConfigMaps Exist

```
THE CONFIGURATION PROBLEM
═══════════════════════════════════════════════════════════════

WITHOUT ConfigMaps:
  Config baked into Docker image:
    docker build --build-arg LOG_LEVEL=DEBUG .
    docker build --build-arg LOG_LEVEL=INFO .
    → Different images for dev vs prod
    → Image rebuild on every config change
    → Config and code releases are coupled
    → Impossible to change config without a new deployment

  Config as env vars in Pod spec:
    env:
    - name: DB_HOST
      value: "prod-db.internal"
    → Config lives inside Kubernetes YAML
    → Multiple places to update for same config value
    → No grouping of related config
    → Hard to diff between environments

WITH ConfigMaps:
  Single source of truth for config per environment.
  Same Pod YAML in dev and prod — only ConfigMap differs.
  Config changes without image rebuilds.
  Config changes can be made without touching Pod spec.
  Config is a first-class Kubernetes object — versionable, auditable.
```

---

### Three Ways to Inject ConfigMap Into a Pod

```
INJECTION METHOD 1: Environment Variables (individual keys)
INJECTION METHOD 2: Environment Variables (all keys at once)
INJECTION METHOD 3: Volume Mount (files)

Each has different behavior for updates — critical to understand.
```

---

### ConfigMap YAML — All Patterns

```yaml
# ConfigMap definition — multiple data types
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  # Simple key-value pairs
  LOG_LEVEL: "INFO"
  MAX_CONNECTIONS: "100"
  CACHE_TTL: "300"
  DB_HOST: "postgres.production.svc.cluster.local"
  DB_PORT: "5432"
  FEATURE_DARK_MODE: "true"

  # Multi-line config files — key is filename, value is content
  app.yaml: |
    server:
      port: 8080
      timeout: 30s
    database:
      pool_size: 10
      max_idle: 5
    logging:
      level: INFO
      format: json

  nginx.conf: |
    server {
      listen 8080;
      server_name _;
      location /health {
        return 200 "OK";
      }
      location / {
        proxy_pass http://localhost:8081;
        proxy_read_timeout 30s;
      }
    }

  # Binary data — use binaryData (base64 encoded)
# binaryData:
#   logo.png: <base64-encoded-bytes>
```

---

### Injection Method 1: Individual Key as Env Var

```yaml
spec:
  containers:
  - name: app
    image: myapp:v1
    env:
    - name: LOG_LEVEL                   # env var name in container
      valueFrom:
        configMapKeyRef:
          name: app-config              # ConfigMap name
          key: LOG_LEVEL                # specific key from ConfigMap
          optional: false               # fail pod if key missing (default)
    - name: MAX_CONNECTIONS
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: MAX_CONNECTIONS
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DB_HOST

# BEHAVIOR:
# ConfigMap changes → env var DOES NOT update in running pod
# Pod must restart to pick up new env var values
# Pros: Simple. No volume mount needed.
# Cons: Restart required for changes. Verbose for many keys.
```

---

### Injection Method 2: All Keys as Env Vars (envFrom)

```yaml
spec:
  containers:
  - name: app
    image: myapp:v1
    envFrom:
    - configMapRef:
        name: app-config                # ALL keys injected as env vars
        optional: false
      prefix: "APP_"                    # optional prefix: APP_LOG_LEVEL etc.

    - configMapRef:
        name: feature-flags             # second ConfigMap — also all keys
      prefix: "FF_"                     # FF_DARK_MODE etc.

# RESULT inside container:
#   APP_LOG_LEVEL=INFO
#   APP_MAX_CONNECTIONS=100
#   APP_DB_HOST=postgres.production...
#   FF_DARK_MODE=true

# BEHAVIOR:
# Same as individual — env vars DO NOT update in running pod
# Restart required for changes
# Pros: Concise. All keys in one line.
# Cons: All keys injected — can pollute env namespace.
#       Key names must be valid env var names (no dots, hyphens become _)
```

---

### Injection Method 3: Volume Mount (Files)

```yaml
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    volumeMounts:
    - name: nginx-config
      mountPath: /etc/nginx/conf.d      # mount directory
      readOnly: true
    - name: app-yaml
      mountPath: /etc/app               # mount directory
      readOnly: true

  - name: app
    image: myapp:v1
    volumeMounts:
    - name: app-yaml
      mountPath: /etc/app/config.yaml   # mount single file
      subPath: app.yaml                 # ← specific key from ConfigMap
      readOnly: true
      # subPath: mount a single key as a file at exact path
      # Without subPath: ALL keys mounted as separate files in directory

  volumes:
  - name: nginx-config
    configMap:
      name: app-config
      items:                            # optional: select specific keys
      - key: nginx.conf
        path: nginx.conf                # filename in mount
        mode: 0644
      defaultMode: 0644                 # permissions for all files

  - name: app-yaml
    configMap:
      name: app-config
      items:
      - key: app.yaml
        path: app.yaml

# BEHAVIOR:
# ConfigMap changes → files UPDATE AUTOMATICALLY in ~60 seconds
# Updates are atomic: symlink swap — no partial reads
# App must explicitly re-read the file to pick up changes
# nginx: needs SIGHUP/nginx -s reload
# Java: needs PropertySource refresh
# Go: needs inotify watcher or periodic re-read
#
# ⚠️ subPath volumes DO NOT auto-update!
# subPath breaks the symlink mechanism that enables auto-update
# Avoid subPath if you need live config reloading
```

---

### kubectl ConfigMap Commands

```bash
# Create ConfigMap from literal values
kubectl create configmap app-config \
  --from-literal=LOG_LEVEL=INFO \
  --from-literal=MAX_CONNECTIONS=100 \
  -n production

# Create ConfigMap from a file (key = filename, value = file content)
kubectl create configmap nginx-config \
  --from-file=nginx.conf \
  -n production

# Create from a directory (all files become keys)
kubectl create configmap app-configs \
  --from-file=./config-dir/ \
  -n production

# Create from env file (.env format: KEY=VALUE per line)
kubectl create configmap app-config \
  --from-env-file=production.env \
  -n production

# List ConfigMaps
kubectl get configmap -n production

# View ConfigMap content
kubectl get configmap app-config -n production -o yaml

# Describe (good for checking data keys)
kubectl describe configmap app-config -n production

# Edit in place
kubectl edit configmap app-config -n production

# Delete
kubectl delete configmap app-config -n production

# Check what env vars a pod sees from ConfigMap
kubectl exec -it myapp-pod -n production -- env | grep -E "LOG|MAX|DB"
```

---

### ConfigMap Update Behavior Summary

```
UPDATE BEHAVIOR BY INJECTION METHOD
═══════════════════════════════════════════════════════════════

Method          Auto-Update?  How to Apply Changes
──────────────────────────────────────────────────────────────
env (valueFrom) NO            kubectl rollout restart deployment
envFrom         NO            kubectl rollout restart deployment
volume mount    YES (~60s)    App must re-read file (signal/poll)
volume+subPath  NO            kubectl rollout restart deployment

RECOMMENDATION:
  Use volume mounts for config that changes often (feature flags,
  tuning parameters, log levels) — enables live reload.
  Use env vars for config that rarely changes (DB host, port) —
  simpler, restart is acceptable.
```

---

### 🎤 Short Crisp Interview Answer

> *"ConfigMaps store non-sensitive configuration data as key-value pairs, decoupling config from container images. There are three injection methods: individual key as env var, all keys via envFrom, and volume mount as files. The critical difference is update behavior — env var injections require a pod restart to pick up ConfigMap changes, while volume mounts auto-update within about 60 seconds via an atomic symlink swap. The app must re-read the file to use the new values. The one exception is subPath volume mounts — they break the symlink mechanism and do NOT auto-update."*

---

### ⚠️ Gotchas

1. **subPath blocks auto-update** — using `subPath` to mount a single key as a file at an exact path breaks the symlink-based update mechanism. The file becomes static and never updates. Use a full directory mount and reference the file inside instead.
2. **Key names for envFrom must be valid env var names** — if a ConfigMap key has dots or hyphens (e.g., `app.properties`), injecting it as envFrom fails. Use volume mount for files with non-env-var-safe names.
3. **ConfigMap size limit is 1MB** — stored in etcd which has a 1.5MB object size limit. Very large configs should use a custom storage solution or mounted from a PVC.
4. **No validation on ConfigMap data** — Kubernetes doesn't validate that the config content is valid JSON, YAML, or nginx syntax. The app discovers errors at startup, not at `kubectl apply` time.

---

### Common Interview Questions

**Q: What is the difference between envFrom and valueFrom for ConfigMaps?**
> valueFrom injects a single specific key from a ConfigMap as one env var — you control the env var name. envFrom injects ALL keys from a ConfigMap as env vars at once — the ConfigMap keys become the env var names (with optional prefix). Both require pod restart to see updates. envFrom is concise but can pollute the environment namespace and fails if any key has an invalid env var name.

**Q: How do ConfigMap volume mount updates work internally?**
> Kubelet syncs ConfigMap data from the API server periodically (configSyncPeriod, default 60s). When a ConfigMap changes, kubelet writes new files to a temp directory then atomically swaps a symlink to point to it. The mounted path resolves through the symlink to the new files. This atomicity prevents partial reads during updates. The app sees new files within ~60 seconds but must explicitly re-read them.

---

### Connections
- Secrets follow same injection patterns — **5.2**
- Projected volumes combine ConfigMap + Secret + token — **5.5**
- Immutable ConfigMaps prevent accidental changes — **5.6**
- subPath behavior — **Category 4 (4.1)**

---

---

# ⚠️ 5.2 Secrets — Types, Base64, Security

## 🟢 Beginner — HIGH PRIORITY

### What it is in simple terms

A Secret is a **Kubernetes object for storing sensitive data** — passwords, tokens, certificates, SSH keys. It uses the same injection patterns as ConfigMaps (env vars, volume mounts) but adds security properties: values are base64-encoded (not plaintext in YAML), mounted as tmpfs (never touches node disk), and can be encrypted at rest in etcd (topic 5.3).

---

### Base64 Is NOT Encryption

```
⚠️  THE MOST IMPORTANT THING TO UNDERSTAND ABOUT SECRETS
═══════════════════════════════════════════════════════════════

Base64 encoding is NOT encryption.
Base64 is REVERSIBLE with zero knowledge:
  echo "bXlwYXNzd29yZA==" | base64 -d
  → mypassword

Purpose of base64 in Secrets:
  Allows binary data (TLS certs, SSH keys) to be stored in YAML.
  Makes YAML parsing reliable (no special character issues).
  It is a DATA FORMAT, not a security mechanism.

WHAT base64 IS:
  ✓ Format to safely store binary data in text fields
  ✓ Handled automatically by kubectl for you

WHAT base64 IS NOT:
  ✗ Encryption
  ✗ Obfuscation that provides any security
  ✗ A reason to think your Secret data is protected in etcd

ACTUAL SECURITY PROTECTIONS FOR SECRETS:
  1. RBAC — restrict who can GET/LIST Secrets
     (LIST is dangerous — returns all secret data at once)
  2. Encryption at rest — EncryptionConfiguration on API Server (5.3)
  3. Node isolation — Secret only sent to nodes that need it
  4. tmpfs mount — Secret volume never written to node disk
  5. External secret managers — Vault, AWS Secrets Manager (5.4, 5.8)

ETCD WITHOUT ENCRYPTION:
  kubectl get secret mysecret -o yaml
  # data:
  #   password: bXlwYXNzd29yZA==   ← trivially decoded
  # ETCD stores this as plaintext bytes
  # Anyone with etcd access has your secrets
  # → Always enable EncryptionConfiguration in production
```

---

### Secret Types — All Six

```
KUBERNETES SECRET TYPES
═══════════════════════════════════════════════════════════════

TYPE: Opaque (default — general purpose)
  type: Opaque
  Any key-value pairs. No schema validation.
  Used for: DB passwords, API keys, app credentials
  Most commonly used type.

TYPE: kubernetes.io/tls
  type: kubernetes.io/tls
  Required keys: tls.crt (certificate) and tls.key (private key)
  Used for: TLS termination in Ingress, mTLS between services
  Validated: API Server checks tls.crt is a valid PEM cert
  Created by: cert-manager, manual kubectl, ACME issuers

TYPE: kubernetes.io/dockerconfigjson
  type: kubernetes.io/dockerconfigjson
  Required key: .dockerconfigjson (JSON with registry credentials)
  Used for: pulling images from private container registries
  Referenced in pod spec: imagePullSecrets
  Created by: kubectl create secret docker-registry

TYPE: kubernetes.io/service-account-token
  type: kubernetes.io/service-account-token
  Required key: token (JWT), ca.crt, namespace
  Used for: ServiceAccount authentication to API Server
  Auto-created by K8s for ServiceAccounts (legacy — K8s < 1.24)
  K8s 1.24+: tokens are projected into pods directly (not Secret objects)

TYPE: kubernetes.io/ssh-auth
  type: kubernetes.io/ssh-auth
  Required key: ssh-privatekey
  Used for: Git clone over SSH, server SSH access

TYPE: kubernetes.io/basic-auth
  type: kubernetes.io/basic-auth
  Required keys: username, password
  Used for: HTTP Basic Auth credentials
  Provides type validation — both keys must be present
```

---

### Secret YAML — All Types

```yaml
# Opaque — general purpose (most common)
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
  namespace: production
type: Opaque
data:
  # Values MUST be base64 encoded in YAML
  username: cHJvZF91c2Vy      # echo -n "prod_user" | base64
  password: c3VwZXJzZWNyZXQ=  # echo -n "supersecret" | base64
  host: cG9zdGdyZXMucHJvZHVjdGlvbi5zdmMuY2x1c3Rlci5sb2NhbA==
stringData:
  # stringData lets you write plaintext — K8s auto-encodes to base64
  # Use for readability in manifests
  # Gets merged with data: after base64 encoding
  api_key: "sk-abc123def456ghi789"
  connection_string: "postgres://prod_user:supersecret@postgres:5432/mydb"

---
# TLS Secret — for Ingress TLS, mTLS
apiVersion: v1
kind: Secret
metadata:
  name: tls-cert
  namespace: production
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-certificate-PEM>
  tls.key: <base64-encoded-private-key-PEM>

---
# Docker registry — for private image pulls
apiVersion: v1
kind: Secret
metadata:
  name: ecr-pull-secret
  namespace: production
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-config-json>

# OR create with kubectl (recommended):
# kubectl create secret docker-registry ecr-pull-secret \
#   --docker-server=123456789.dkr.ecr.us-east-1.amazonaws.com \
#   --docker-username=AWS \
#   --docker-password=$(aws ecr get-login-password) \
#   -n production

---
# SSH auth — for Git operations
apiVersion: v1
kind: Secret
metadata:
  name: git-ssh-key
  namespace: production
type: kubernetes.io/ssh-auth
data:
  ssh-privatekey: <base64-encoded-private-key>
```

---

### Injecting Secrets Into Pods

```yaml
# Method 1: Individual key as env var
spec:
  containers:
  - name: app
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: database-credentials
          key: password
          optional: false           # fail pod if secret/key missing

# Method 2: All keys as env vars (envFrom)
spec:
  containers:
  - name: app
    envFrom:
    - secretRef:
        name: database-credentials
        optional: false

# Method 3: Volume mount (RECOMMENDED for sensitive data)
spec:
  containers:
  - name: app
    volumeMounts:
    - name: db-creds
      mountPath: /etc/secrets       # app reads /etc/secrets/password
      readOnly: true
    - name: tls-certs
      mountPath: /etc/tls
      readOnly: true

  volumes:
  - name: db-creds
    secret:
      secretName: database-credentials
      defaultMode: 0400             # owner read-only — most restrictive
      items:
      - key: password
        path: db-password           # file: /etc/secrets/db-password
      - key: username
        path: db-username           # file: /etc/secrets/db-username

  - name: tls-certs
    secret:
      secretName: tls-cert
      defaultMode: 0400

# imagePullSecrets — for private registry image pulls
spec:
  imagePullSecrets:
  - name: ecr-pull-secret
  containers:
  - name: app
    image: 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1
```

---

### kubectl Secret Commands

```bash
# Create Opaque secret from literals
kubectl create secret generic database-credentials \
  --from-literal=username=prod_user \
  --from-literal=password=supersecret \
  -n production

# Create Opaque from files
kubectl create secret generic tls-files \
  --from-file=tls.crt=./server.crt \
  --from-file=tls.key=./server.key

# Create TLS secret (validates PEM format)
kubectl create secret tls tls-cert \
  --cert=server.crt \
  --key=server.key \
  -n production

# Create docker-registry secret
kubectl create secret docker-registry ecr-pull-secret \
  --docker-server=123456789.dkr.ecr.us-east-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region us-east-1) \
  -n production

# View secret (data is base64 — needs decoding)
kubectl get secret database-credentials -n production -o yaml
# data:
#   password: c3VwZXJzZWNyZXQ=

# Decode a specific key
kubectl get secret database-credentials -n production \
  -o jsonpath='{.data.password}' | base64 -d
# supersecret

# List all secrets in namespace
kubectl get secret -n production

# Describe (shows keys but NOT values — safe to share output)
kubectl describe secret database-credentials -n production
# Data
# ====
# password:  11 bytes   ← shows size, not value
# username:  9 bytes

# Delete
kubectl delete secret database-credentials -n production
```

---

### Secret vs ConfigMap — When to Use Which

```
USE ConfigMap FOR:
  ✓ Log levels, debug flags
  ✓ Feature flags (non-security-sensitive)
  ✓ Connection strings without credentials
  ✓ Application tuning parameters
  ✓ nginx/app config files
  ✓ Environment names (staging, production)
  ✓ Any config that can appear in logs safely

USE Secret FOR:
  ✓ Passwords, API keys, tokens
  ✓ TLS private keys and certificates
  ✓ Database connection strings with credentials
  ✓ SSH private keys
  ✓ Any data that must not appear in logs
  ✓ OAuth client secrets
  ✓ Webhook signing secrets
  ✓ Anything you would store in a password manager
```

---

### 🎤 Short Crisp Interview Answer

> *"Secrets store sensitive data in Kubernetes. The most important point is that base64 encoding is NOT encryption — it's just a format for binary-safe storage, trivially reversible. Real security comes from RBAC restricting who can read secrets, EncryptionConfiguration encrypting secrets at rest in etcd, and secrets only being sent to nodes that need them. Kubernetes has six Secret types — Opaque for general purpose, kubernetes.io/tls for certificates, dockerconfigjson for registry auth, service-account-token for SA authentication, ssh-auth, and basic-auth. Volume mount is more secure than env vars because the value isn't in the process environment and is stored as tmpfs never touching node disk."*

---

### ⚠️ Gotchas

1. **base64 is NOT encryption** — the most common misconception. Anyone with kubectl get secret access sees the value. Protect with RBAC and encryption at rest.
2. **LIST permission on Secrets is dangerous** — `get` on a specific secret is fine. `list` on all secrets in a namespace returns ALL secret data at once. Minimize who has list access.
3. **Secret env vars appear in container env dumps** — `kubectl exec pod -- env` prints all env vars including secret values. Volume mounts are safer.
4. **stringData is write-only** — when you read back a Secret created with stringData, the data appears in the `data` field (base64 encoded), never in `stringData`. stringData is a convenience write-only input field.
5. **imagePullSecrets are namespace-scoped** — a private registry secret in namespace A is not automatically available in namespace B. Must create the secret in each namespace, or attach it to the default ServiceAccount.

---

### Common Interview Questions

**Q: Are Kubernetes Secrets secure?**
> By default, Secrets offer limited security. Values are base64-encoded (not encrypted) and stored in etcd as near-plaintext. Security depends on: RBAC restricting secret access, EncryptionConfiguration enabling encryption at rest in etcd, network policies limiting pod access, and ideally an external secrets manager like Vault or AWS Secrets Manager as the source of truth. Secrets by themselves, without these protections, are essentially ConfigMaps with a false sense of security.

**Q: What is the difference between stringData and data in a Secret?**
> data requires base64-encoded values. stringData accepts plaintext values and Kubernetes automatically base64-encodes them before storage. Both fields end up in the same place — when you read the Secret back, everything is in the data field as base64. stringData is a write convenience that makes Secret manifests more readable but is write-only — it never appears in kubectl get output.

---

### Connections
- Encryption at rest for etcd → **5.3**
- External Secrets from Vault/AWS SM → **5.4**
- Projected volumes combine Secret + ConfigMap → **5.5**
- Immutable Secrets → **5.6**
- Sealed Secrets for GitOps → **5.7**
- CSI Secret Store as alternative injection → **5.8**
- Secret rotation strategies → **5.9**

---

---

# ⚠️ 5.3 Secret Encryption at Rest (EncryptionConfiguration)

## 🟡 Intermediate — HIGH PRIORITY

### What it is in simple terms

By default, Secrets in Kubernetes are stored in etcd **as base64-encoded plaintext** — anyone with direct etcd access sees them clearly. EncryptionConfiguration tells the API Server to encrypt Secret data before writing to etcd, so etcd only ever holds ciphertext. Without this, a compromised etcd backup or etcd node exposes all cluster secrets.

---

### What Happens Without Encryption at Rest

```
WITHOUT EncryptionConfiguration
═══════════════════════════════════════════════════════════════

Developer creates secret:
  kubectl create secret generic db-pass --from-literal=password=supersecret

API Server writes to etcd:
  Key:   /registry/secrets/production/db-pass
  Value: {"kind":"Secret","data":{"password":"c3VwZXJzZWNyZXQ="}}
         ↑ base64("supersecret") stored as near-plaintext

Attack vectors:
  1. etcd backup file exfiltrated → all secrets readable
  2. etcd node compromised → direct etcdctl get → all secrets
  3. etcd snapshot copied → extract with etcdctl → all secrets
  4. Cloud provider etcd snapshot (EKS managed etcd) → if AWS
     account compromised, etcd snapshots may be accessible

Read secret directly from etcd (demonstrates the problem):
  ETCDCTL_API=3 etcdctl get \
    /registry/secrets/production/db-pass \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    | strings
  # → plaintext secret data visible
```

---

### Encryption Providers — Options and Trade-offs

```
ENCRYPTION PROVIDERS (strongest to weakest):
═══════════════════════════════════════════════════════════════

kms (v2 — RECOMMENDED for production):
  Keys managed by external KMS (AWS KMS, GCP KMS, Vault)
  DEK (Data Encryption Key) per object, encrypted by KMS KEK
  Key rotation handled by KMS provider
  ✓ Keys never stored on cluster nodes
  ✓ Hardware security module backing (AWS CloudHSM option)
  ✓ Audit log of every key use in KMS
  ✓ Key rotation without re-encrypting all secrets (envelope)
  ✗ Dependency on KMS availability
    (KMS unreachable → API Server cannot decrypt → cluster degraded)

aescbc (legacy — NOT recommended for new deployments):
  AES-CBC with 256-bit key stored in EncryptionConfiguration file
  ✓ Simple — no external dependency
  ✗ Key stored on API Server node — server compromise = key compromise
  ✗ Key rotation requires manual re-encrypt all secrets
  ✗ No hardware backing
  ✗ CBC mode has known weaknesses vs GCM

aesgcm:
  AES-GCM with 256-bit key stored in EncryptionConfiguration file
  ✓ Better than CBC (authenticated encryption, no padding oracle)
  ✗ Same key storage weakness as aescbc
  ✗ Key must be rotated every 200k writes (counter rollover risk)
  Not recommended for large clusters

secretbox:
  XSalsa20+Poly1305 (NaCl secretbox)
  ✓ Modern authenticated cipher
  ✗ Still local key storage problem
  Rarely used in practice

identity (default — NO encryption):
  Data stored as-is (base64)
  Used to read unencrypted secrets during migration only
```

---

### EncryptionConfiguration — Full Setup

```yaml
# /etc/kubernetes/encryption-config.yaml
# (placed on API Server node, referenced in kube-apiserver flags)
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets                       # what to encrypt
    - configmaps                    # encrypt ConfigMaps too (optional)
    providers:
    # FIRST provider = used for new writes (encryption)
    # ALL providers = tried in order for reads (decryption)

    # Production: KMS v2 (AWS KMS example)
    - kms:
        apiVersion: v2
        name: aws-kms-provider
        endpoint: unix:///var/run/kmsplugin/socket.sock
        # KMS plugin runs as DaemonSet alongside API Server
        # Plugin proxies encrypt/decrypt calls to AWS KMS
        cachesize: 1000             # DEK cache entries (performance)
        timeout: 3s                 # KMS call timeout

    # Fallback: read legacy unencrypted secrets during migration
    - identity: {}                  # MUST be last for reads to work
                                    # MUST NOT be first (would disable encryption)
```

---

### AWS KMS Provider Setup (EKS Approach)

```yaml
# EKS: Enable envelope encryption at cluster creation
# (cannot be enabled after cluster creation on EKS)
aws eks create-cluster \
  --name my-cluster \
  --encryption-config '[{
    "resources": ["secrets"],
    "provider": {
      "keyArn": "arn:aws:kms:us-east-1:123456789:key/abc-def-ghi"
    }
  }]'

# Verify encryption is enabled on EKS cluster
aws eks describe-cluster --name my-cluster \
  --query 'cluster.encryptionConfig'
# [
#   {
#     "resources": ["secrets"],
#     "provider": {
#       "keyArn": "arn:aws:kms:us-east-1:123456789:key/abc-def-ghi"
#     }
#   }
# ]

# ── For self-managed K8s (non-EKS): ─────────────────────────
# 1. Add to kube-apiserver flags (kubeadm: /etc/kubernetes/manifests/kube-apiserver.yaml):
#    - --encryption-provider-config=/etc/kubernetes/encryption-config.yaml

# 2. After enabling encryption, force-rewrite all existing secrets:
#    (previously stored secrets are NOT automatically re-encrypted)
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
# This reads each secret (API Server decrypts with old provider)
# and writes it back (API Server encrypts with new KMS provider)
# ⚠️ This is slow for large clusters — run in batches
```

---

### Verify Encryption Is Working

```bash
# Method 1: Check etcd directly (self-managed only)
ETCDCTL_API=3 etcdctl get \
  /registry/secrets/production/my-secret \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  | hexdump -C | head -20
# If encrypted with AES-CBC: output starts with "k8s:enc:aescbc:v1"
# If encrypted with KMS v2:  output starts with "k8s:enc:kms:v2"
# If unencrypted (identity): output is readable JSON/base64

# Method 2: API Server audit logs
# Encrypted writes show: "verb":"create/update" on secrets
# with encryption provider in request metadata

# Method 3: EKS — use CloudTrail
# KMS decrypt/encrypt calls appear in CloudTrail
# Each secret read = one KMS decrypt call (unless cached)
```

---

### 🎤 Short Crisp Interview Answer

> *"Without EncryptionConfiguration, Kubernetes Secrets are stored in etcd as base64-encoded plaintext — anyone with etcd access sees them. EncryptionConfiguration tells the API Server to encrypt Secret data before writing to etcd using a specified provider. For production, the KMS provider is strongly recommended — it uses an external KMS like AWS KMS or Vault. The encryption uses envelope encryption: a per-object DEK encrypts the data, the KMS encrypts the DEK. Keys never live on the cluster nodes. On EKS, you enable it at cluster creation time by specifying a KMS key ARN — it cannot be enabled after the fact. After enabling, you must force-rewrite all existing Secrets because previously stored Secrets are not automatically re-encrypted."*

---

### ⚠️ Gotchas

1. **Existing Secrets are NOT automatically re-encrypted** — enabling EncryptionConfiguration only encrypts NEW writes. Must force-rewrite all existing Secrets with `kubectl get secrets --all-namespaces -o json | kubectl replace -f -`.
2. **EKS: must be enabled at cluster creation** — EKS does not support enabling encryption after cluster creation. Cannot be added to an existing EKS cluster.
3. **KMS unavailability = API Server degraded** — if the KMS endpoint is unreachable, the API Server cannot decrypt Secrets when pods start, and cannot write new Secrets. Design KMS availability carefully (VPC endpoint for AWS KMS on EKS).
4. **identity provider must be LAST, not first** — if identity is first, new Secrets are written unencrypted. identity must be last to serve as fallback for reading old unencrypted Secrets during migration.
5. **ConfigMaps can also be encrypted** — Secrets get the most attention but ConfigMaps sometimes contain sensitive data too. Consider encrypting both.

---

### Connections
- Secrets without encryption → **5.2**
- External secret managers reduce etcd exposure → **5.4, 5.8**
- EKS KMS key setup → **Category 12**
- etcd as storage backend → **Category 1 (1.4)**

---

---

# 5.4 External Secrets Operator (ESO) — Pulling From Vault/AWS SM

## 🟡 Intermediate

### What it is in simple terms

External Secrets Operator (ESO) is a **Kubernetes operator that syncs secrets from external secret managers** — AWS Secrets Manager, AWS Parameter Store, HashiCorp Vault, GCP Secret Manager, Azure Key Vault — into Kubernetes Secret objects. Applications consume standard Kubernetes Secrets and never need to know or care about the external backend.

---

### Why External Secrets Operator Exists

```
PROBLEM WITH NATIVE KUBERNETES SECRETS
═══════════════════════════════════════════════════════════════

  1. Secret values must be stored somewhere to create them
     → They end up in git repos, CI/CD pipelines, Helm values
     → Exposure risk at rest and in transit

  2. Secret rotation is manual
     → Update Kubernetes Secret → restart pods
     → Operationally painful, so rotations are deferred
     → Long-lived secrets = larger blast radius if compromised

  3. No central audit trail
     → Who accessed which secret? When?
     → Kubernetes RBAC audit shows K8s access but not original source

  4. No secret versioning or rollback
     → Cannot roll back to a previous secret value easily

WHAT AWS SECRETS MANAGER / VAULT PROVIDES:
  ✓ Encrypted storage with hardware KMS backing
  ✓ Automatic rotation (RDS, Redis, custom Lambda rotators)
  ✓ Fine-grained access control via IAM / Vault policies
  ✓ Complete audit trail (CloudTrail / Vault audit log)
  ✓ Versioning — previous secret versions retained
  ✓ Secret never needs to be stored in git or CI env vars

ESO BRIDGE:
  External secret manager = source of truth
  ESO = sync engine
  Kubernetes Secret = local cache for pods
  Pods = never change — still consume standard K8s Secret
```

---

### ESO Architecture

```
ESO SYNC FLOW
═══════════════════════════════════════════════════════════════

  AWS Secrets Manager                    Kubernetes
  ──────────────────                     ──────────
  secret: prod/db/password               ExternalSecret CRD
    value: supersecret123        ◄────── spec.secretStoreRef
    rotation: every 30 days             spec.data.remoteRef
                                               │
                                               │ ESO controller
                                               │ runs reconciliation loop
                                               │ every refreshInterval
                                               │
                                               ▼
                                         Kubernetes Secret
                                           name: db-credentials
                                           data:
                                             password: <synced value>

ESO COMPONENTS:
  1. SecretStore (namespaced) or ClusterSecretStore (cluster-wide):
     Defines HOW to authenticate to the external secret backend.
     Contains the backend type, auth method, and connection details.

  2. ExternalSecret:
     Defines WHAT to sync — which keys from which backend path.
     Specifies refreshInterval for how often to re-sync.
     References a SecretStore.
     ESO controller creates/updates a Kubernetes Secret from this.

  3. ESO Controller:
     Watches ExternalSecret objects.
     Calls external backend API to fetch secret values.
     Creates or updates Kubernetes Secret object.
     Re-syncs on refreshInterval schedule.
```

---

### ESO YAML — AWS Secrets Manager

```yaml
# Step 1: ClusterSecretStore — defines connection to AWS SM
# (admin creates once, shared by all namespaces)
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa       # ServiceAccount with IRSA
            namespace: external-secrets

---
# IAM Role for ESO (IRSA annotation on ServiceAccount)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-secrets-sa
  namespace: external-secrets
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/ExternalSecretsRole
    # Role needs: secretsmanager:GetSecretValue
    #             secretsmanager:DescribeSecret

---
# Step 2: ExternalSecret — defines what to sync
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-credentials
  namespace: production
spec:
  # How often ESO re-reads from AWS SM and updates the K8s Secret
  refreshInterval: "1h"                     # check for updates every hour
  # For rotation: set to "5m" to pick up rotated values quickly

  # Which SecretStore to use
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore

  # What Kubernetes Secret to create/maintain
  target:
    name: db-credentials                    # K8s Secret name to create
    creationPolicy: Owner                   # ESO owns/manages this Secret
    # Owner: ESO creates and deletes the K8s Secret with ExternalSecret
    # Merge:  ESO merges into existing Secret (non-destructive)
    # None:   ESO does not create, only updates if Secret exists

    # Optional: template to transform values
    template:
      type: Opaque
      data:
        # Can transform values using Go templates
        connection_string: >-
          postgres://{{ .username }}:{{ .password }}
          @{{ .host }}:5432/{{ .database }}

  # What to pull from AWS SM
  data:
  - secretKey: username                     # key in Kubernetes Secret
    remoteRef:
      key: prod/database/credentials        # AWS SM secret name
      property: username                    # JSON property within the secret

  - secretKey: password
    remoteRef:
      key: prod/database/credentials
      property: password

  - secretKey: host
    remoteRef:
      key: prod/database/credentials
      property: host

  # Pull ALL properties from one AWS SM secret at once
  dataFrom:
  - extract:
      key: prod/app/config                  # entire JSON object as K8s Secret keys

---
# ESO for AWS SSM Parameter Store
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-parameters
  namespace: production
spec:
  refreshInterval: "15m"
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: app-params
  data:
  - secretKey: feature_flags
    remoteRef:
      key: /production/app/feature_flags   # SSM Parameter path

---
# ESO for HashiCorp Vault
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: production
spec:
  provider:
    vault:
      server: "https://vault.production.internal:8200"
      path: "secret"
      version: "v2"                         # KV v2
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "production-app"            # Vault role
          serviceAccountRef:
            name: vault-auth-sa
```

---

### kubectl ESO Commands

```bash
# Install ESO via Helm
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets \
  external-secrets/external-secrets \
  -n external-secrets --create-namespace

# Check ESO controller is running
kubectl get pods -n external-secrets

# List ExternalSecrets
kubectl get externalsecret -n production
# NAME                   STORE                  REFRESH INTERVAL  STATUS
# database-credentials   aws-secrets-manager    1h                SecretSynced

# Describe — shows last sync status and any errors
kubectl describe externalsecret database-credentials -n production
# Status:
#   Conditions:
#     Type:    Ready
#     Status:  True
#     Message: Secret was synced

# Force immediate re-sync (add annotation)
kubectl annotate externalsecret database-credentials \
  force-sync=$(date +%s) -n production

# Verify the synced K8s Secret was created
kubectl get secret db-credentials -n production
kubectl describe secret db-credentials -n production

# Check SecretStore connectivity
kubectl get secretstore -n production
kubectl describe secretstore vault-backend -n production
# Conditions:
#   Type:    Ready
#   Status:  True  ← backend is reachable and auth works
```

---

### 🎤 Short Crisp Interview Answer

> *"External Secrets Operator bridges external secret managers — AWS Secrets Manager, Vault, GCP Secret Manager — with Kubernetes. You define a SecretStore specifying how to authenticate to the backend, and an ExternalSecret specifying what to sync and how often. ESO's controller polls the external backend at the refreshInterval and creates or updates a standard Kubernetes Secret. Pods consume the standard Secret and don't know or care about the backend. This solves three problems native Secrets don't: secrets never need to live in git or CI, rotation is automatic when the external backend rotates the value and ESO syncs it, and you get a full audit trail in CloudTrail or Vault audit logs."*

---

### ⚠️ Gotchas

1. **refreshInterval is not instant rotation** — if AWS SM rotates a secret, Kubernetes pods don't see the new value until the next refreshInterval sync AND a pod restart. Set a short refreshInterval and implement pod reload (topic 5.9).
2. **ESO deletion policy** — with creationPolicy: Owner, deleting an ExternalSecret also deletes the Kubernetes Secret it manages. Pods using that Secret immediately fail. Be careful with ExternalSecret lifecycle.
3. **IRSA permissions must be exact** — missing `secretsmanager:GetSecretValue` causes sync to fail silently after initial setup. Monitor ExternalSecret status conditions.
4. **dataFrom.extract pulls entire secret as flat keys** — the JSON object in AWS SM is flattened. Nested JSON is not supported well — design secrets as flat key-value JSON objects.

---

### Connections
- Kubernetes Secrets as the output of ESO → **5.2**
- IRSA for AWS authentication from pods → **Category 12**
- CSI Secret Store is an alternative approach → **5.8**
- Secret rotation strategies using ESO → **5.9**

---

---

# 5.5 Projected Volumes — Combining Secrets/ConfigMaps/SA Tokens

## 🟡 Intermediate

### What it is in simple terms

A Projected Volume is a **single volume mount that combines multiple sources** — Secrets, ConfigMaps, ServiceAccount tokens, and pod metadata — into one directory. Instead of multiple volume mounts cluttering the pod spec, everything appears in one cohesive location in the container filesystem.

---

### Why Projected Volumes Exist

```
PROBLEM: MULTIPLE MOUNTS FOR RELATED CONFIG
═══════════════════════════════════════════════════════════════

WITHOUT Projected Volume:
  volumeMounts:
  - name: app-config
    mountPath: /etc/app/config
  - name: db-secret
    mountPath: /etc/app/secrets
  - name: tls-cert
    mountPath: /etc/app/tls
  - name: sa-token
    mountPath: /var/run/secrets/token

  volumes:
  - name: app-config
    configMap: { name: app-config }
  - name: db-secret
    secret: { secretName: db-credentials }
  - name: tls-cert
    secret: { secretName: tls-cert }
  - name: sa-token
    serviceAccountToken: { path: token }

  → 4 separate volume mounts, 4 volume definitions, scattered locations

WITH Projected Volume:
  volumeMounts:
  - name: combined
    mountPath: /etc/app          # ALL sources appear here together

  volumes:
  - name: combined
    projected:
      sources:
      - configMap: { name: app-config }
      - secret: { secretName: db-credentials }
      - secret: { secretName: tls-cert }
      - serviceAccountToken: { path: token, expirationSeconds: 3600 }

  → 1 volume mount, 1 volume definition, unified location
  → /etc/app/config-key, /etc/app/password, /etc/app/tls.crt, /etc/app/token
```

---

### Projected Volume Sources — Full Reference

```
FOUR SOURCES COMBINABLE IN PROJECTED VOLUME:
═══════════════════════════════════════════════════════════════

1. configMap
   All keys or selected keys from a ConfigMap.
   Same as regular configMap volume.

2. secret
   All keys or selected keys from a Secret.
   Mounted as tmpfs (RAM) — never touches node disk.
   Same file permission control as regular secret volume.

3. serviceAccountToken
   ⚠️  This is the MODERN approach (K8s 1.20+, stable 1.22)
   Bound ServiceAccount token — projected directly into pod.
   NOT a long-lived token stored as a Secret object.
   Characteristics:
     - Time-limited: expiration configured (min 10 minutes)
     - Audience-bound: valid only for specified audience
     - Auto-rotated: kubelet automatically refreshes before expiry
     - Pod-bound: token becomes invalid when pod dies
   Old approach (K8s < 1.20): mounted /var/run/secrets/kubernetes.io/serviceaccount/
     token was a long-lived Secret — never expired, high blast radius
   New projected approach: short-lived, auto-rotated, much safer

4. downwardAPI
   Exposes pod metadata as files in the container.
   What you can expose:
     metadata.name          → pod name
     metadata.namespace     → namespace
     metadata.labels        → all labels as key=value file
     metadata.annotations   → all annotations as key=value file
     spec.nodeName          → which node pod is on
     spec.serviceAccountName → SA name
     status.podIP           → pod IP address
     status.hostIP          → node IP
     limits.cpu             → CPU limit
     limits.memory          → memory limit
     requests.cpu           → CPU request
     requests.memory        → memory request
```

---

### Projected Volume YAML — Complete Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  namespace: production
spec:
  serviceAccountName: myapp-sa

  containers:
  - name: app
    image: myapp:v1
    volumeMounts:
    - name: app-combined
      mountPath: /etc/app          # ALL sources accessible here
      readOnly: true

  volumes:
  - name: app-combined
    projected:
      defaultMode: 0440            # default permissions for all files

      sources:
      # Source 1: ConfigMap — app configuration
      - configMap:
          name: app-config
          items:
          - key: app.yaml
            path: config/app.yaml  # /etc/app/config/app.yaml

      # Source 2: Secret — database credentials
      - secret:
          name: database-credentials
          items:
          - key: password
            path: secrets/db-password    # /etc/app/secrets/db-password
            mode: 0400                   # very restrictive — owner read-only
          - key: username
            path: secrets/db-username

      # Source 3: Secret — TLS certificates
      - secret:
          name: tls-cert
          items:
          - key: tls.crt
            path: tls/server.crt         # /etc/app/tls/server.crt
          - key: tls.key
            path: tls/server.key
            mode: 0400

      # Source 4: ServiceAccount Token — for calling K8s API or other services
      - serviceAccountToken:
          path: token                    # /etc/app/token
          expirationSeconds: 3600        # 1 hour — kubelet auto-refreshes at 80%
          audience: "https://kubernetes.default.svc"
          # audience: can be any string — limits token validity to specific recipient
          # For Vault: audience: "vault"
          # For AWS IRSA: audience: "sts.amazonaws.com"

      # Source 5: DownwardAPI — pod metadata as files
      - downwardAPI:
          items:
          - path: pod-info/name          # /etc/app/pod-info/name
            fieldRef:
              fieldPath: metadata.name
          - path: pod-info/namespace
            fieldRef:
              fieldPath: metadata.namespace
          - path: pod-info/node
            fieldRef:
              fieldPath: spec.nodeName
          - path: pod-info/ip
            fieldRef:
              fieldPath: status.podIP
          - path: resources/memory-limit
            resourceFieldRef:
              containerName: app
              resource: limits.memory
              divisor: "1Mi"             # convert to MiB

# Result: /etc/app/
#   config/app.yaml          ← from ConfigMap
#   secrets/db-password      ← from Secret (tmpfs)
#   secrets/db-username      ← from Secret (tmpfs)
#   tls/server.crt           ← from Secret (tmpfs)
#   tls/server.key           ← from Secret (tmpfs) mode 0400
#   token                    ← auto-rotating ServiceAccount token
#   pod-info/name            ← "myapp-abc123"
#   pod-info/namespace       ← "production"
#   pod-info/node            ← "worker-node-2"
#   pod-info/ip              ← "10.244.2.15"
#   resources/memory-limit   ← "512" (MiB)
```

---

### ServiceAccount Token Evolution

```
OLD APPROACH (K8s < 1.24 — avoid in new deployments):
  Each ServiceAccount gets a Secret auto-created:
    secret/myapp-sa-token-xxxxx
      data:
        token: <long-lived JWT, never expires>
        ca.crt: <cluster CA>
        namespace: <namespace>
  Pod mounts at: /var/run/secrets/kubernetes.io/serviceaccount/

  Problems:
  ✗ Token never expires — compromise = permanent access until manually deleted
  ✗ Not audience-bound — valid for any audience
  ✗ Pod can outlive the token Secret accidentally

NEW APPROACH (K8s 1.20+ — use this):
  Projected serviceAccountToken in pod spec.
  Token created fresh for each pod, bound to pod UID.
  Time-limited — expires in expirationSeconds.
  Kubelet auto-rotates when 80% of lifetime elapsed.
  Pod dies → token immediately revoked.

  Benefits:
  ✓ Short-lived — compromise window is small
  ✓ Audience-bound — valid only for specified service
  ✓ Pod-bound — automatically revoked on pod termination
  ✓ Auto-rotated — kubelet handles refresh transparently

WORKLOAD IDENTITY (EKS):
  IRSA uses projected ServiceAccount token with
  audience: "sts.amazonaws.com"
  AWS STS validates the token → returns temporary IAM credentials
  → Covered in Category 12
```

---

### 🎤 Short Crisp Interview Answer

> *"Projected volumes combine multiple sources — Secrets, ConfigMaps, ServiceAccount tokens, and pod downwardAPI metadata — into a single volume mount. Instead of four separate volume mounts in the pod spec, everything appears in one directory. The most important source is serviceAccountToken, which implements the modern approach to SA token injection: short-lived, audience-bound, and auto-rotated by kubelet, as opposed to the old long-lived token Secrets that never expired. The downwardAPI source is useful for injecting pod metadata like pod name, namespace, and IP into the container without needing to call the Kubernetes API."*

---

### ⚠️ Gotchas

1. **serviceAccountToken path must be relative** — the path field in serviceAccountToken must be a relative path, not absolute. It is relative to the projected volume's mountPath.
2. **Conflicting file paths in projected volume** — if two sources map files to the same path, the last one wins. No error is raised. Check paths carefully.
3. **downwardAPI resourceFieldRef requires containerName** — when referencing container resource limits/requests, you must specify containerName. Omitting it fails silently or picks wrong container.
4. **defaultMode applies to all files** — but individual items can override with their own mode field. Secret items that need 0400 must set mode explicitly even with defaultMode: 0440.

---

### Connections
- ConfigMaps and Secrets as projected sources → **5.1, 5.2**
- ServiceAccount tokens for IRSA on EKS → **Category 12**
- downwardAPI for pod identity in distributed systems → **Category 2 (2.1)**

---

---

# 5.6 Immutable ConfigMaps/Secrets

## 🟡 Intermediate

### What it is and why it matters

Marking a ConfigMap or Secret as **immutable: true** permanently prevents any changes to its data. Once set, the object's content cannot be modified — only deleted and recreated. This sounds restrictive but provides critical performance and safety benefits at scale.

---

### Why Immutability Matters at Scale

```
THE CONFIGMAP WATCH PROBLEM AT SCALE
═══════════════════════════════════════════════════════════════

Default behavior (mutable ConfigMaps/Secrets):
  Every kubelet on every node watches ALL ConfigMaps and Secrets
  that are mounted in pods on that node.
  Why? To detect changes and update mounted files.

  100 nodes × 500 mounted ConfigMaps/Secrets per node
  = 50,000 open API Server watches

  When any ConfigMap changes:
  → API Server must fan-out the change to all watching kubelets
  → With 1000 pods watching one ConfigMap: 1000 notifications
  → Significant etcd and API Server load in large clusters

  Every watch is a long-lived HTTP connection.
  At scale: API Server connection pressure becomes a bottleneck.

IMMUTABLE ConfigMaps/Secrets:
  Kubelet does NOT watch immutable objects.
  It caches the value once and never watches for changes.
  50,000 watches → 0 watches for immutable objects.
  Dramatically reduces API Server load at scale.

PERFORMANCE NUMBERS (from Kubernetes team):
  Clusters with thousands of nodes:
  Removing watches for immutable ConfigMaps reduced
  API Server CPU by 20-40% in some large deployments.
```

---

### Immutable ConfigMap/Secret YAML

```yaml
# Immutable ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v3            # ← version in name for immutable objects
  namespace: production
immutable: true                  # ← the key field
data:
  LOG_LEVEL: "INFO"
  MAX_CONNECTIONS: "100"
  BUILD_VERSION: "v3.2.1"
  # Once immutable: true is set and object is created:
  # kubectl edit configmap app-config-v3 → Error: field is immutable
  # kubectl apply with changed data → Error: field is immutable
  # kubectl patch → Error: field is immutable

---
# Immutable Secret
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials-v2
  namespace: production
type: Opaque
immutable: true
data:
  password: c3VwZXJzZWNyZXQ=
  username: cHJvZF91c2Vy

# To "update" an immutable ConfigMap/Secret:
#   1. Create new version: app-config-v4 with new data
#   2. Update Deployment to reference app-config-v4
#   3. Perform rolling update
#   4. Delete old app-config-v3 when all pods migrated
# This is the IMMUTABLE INFRASTRUCTURE pattern
```

---

### Immutability Change Rules

```bash
# Set immutable (irreversible on existing object)
kubectl patch configmap app-config \
  -p '{"immutable": true}'

# Try to change data after marking immutable
kubectl patch configmap app-config \
  -p '{"data":{"LOG_LEVEL":"DEBUG"}}'
# Error: ConfigMap "app-config" is immutable

# Try to remove immutable flag
kubectl patch configmap app-config \
  -p '{"immutable": false}'
# Error: field is immutable — cannot un-set immutable: true

# Only delete is allowed
kubectl delete configmap app-config
# (then recreate if needed)

# UPGRADE PATTERN:
# 1. Create app-config-v2 with updated data
kubectl apply -f configmap-v2.yaml

# 2. Update deployment to reference new ConfigMap
kubectl set env deployment/myapp \
  --from=configmap/app-config-v2

# 3. Rolling update happens automatically
# 4. After rollout complete, cleanup old version
kubectl delete configmap app-config-v1
```

---

### When to Use Immutable vs Mutable

```
USE IMMUTABLE: true FOR:
  ✓ Configuration that represents a fixed release version
    ("these settings match v3.2.1 of our app")
  ✓ TLS certificates with specific expiry
  ✓ Build-time constants (git commit hash, build timestamp)
  ✓ Feature flag snapshots tied to a specific deployment
  ✓ Any config in a GitOps pipeline where immutable infra is desired
  ✓ Large clusters (>100 nodes) — performance benefit significant
  ✓ High-security environments (accidental change prevention)

USE MUTABLE (default) FOR:
  ✓ Frequently changed config (feature flags, tuning params)
  ✓ Live config reload (changing log level without restart)
  ✓ Config that operations team adjusts in emergencies
  ✓ Development environments where iteration speed matters
  ✓ Config where volume mount auto-update is relied upon
```

---

### 🎤 Short Crisp Interview Answer

> *"Immutable ConfigMaps and Secrets, set with immutable: true, prevent any modification to the object's data after creation. The primary motivation is performance at scale — kubelets watch all mutable ConfigMaps and Secrets mounted on their pods for changes, which creates thousands of open API Server connections in large clusters. Immutable objects are never watched — kubelet caches them once. This reduces API Server and etcd load by 20-40% in large deployments. The trade-off is that updates require creating a new versioned object and rolling the Deployment to reference it — the immutable infrastructure pattern — rather than editing the object in place."*

---

### ⚠️ Gotchas

1. **Cannot un-set immutable** — once `immutable: true` is set, it cannot be changed back to false. The only option is delete and recreate.
2. **Immutable blocks ALL data changes** — adding a new key, removing a key, or changing a value all fail. Label and annotation changes are still allowed (metadata is mutable, only data/stringData/binaryData are immutable).
3. **Old object must be deleted to reuse name** — you cannot update an immutable ConfigMap even if you want to change the name. The immutable flag locks the content, not the name, but practically you use new names for new versions.

---

### Connections
- Auto-update behavior of mutable volumes → **5.1**
- Immutable objects in GitOps workflows → **5.7**
- API Server watch mechanism → **Category 1 (1.13)**

---

---

# 5.7 Sealed Secrets — GitOps-Safe Secret Management

## 🔴 Advanced

### What it is in simple terms

Sealed Secrets is a **Kubernetes controller + CLI tool that encrypts Secret manifests** so they are safe to commit to Git. A SealedSecret is a CRD that contains an encrypted version of a Kubernetes Secret. Only the Sealed Secrets controller running in the cluster can decrypt it — not even the person who created it can decrypt it without the cluster's private key.

---

### The GitOps Secret Problem

```
THE CORE PROBLEM: SECRETS IN GIT
═══════════════════════════════════════════════════════════════

GitOps requirement:
  ALL Kubernetes manifests in Git → Argo CD / Flux syncs them

Problem:
  Secret manifest has base64-encoded values → plaintext in Git
  git push → secret committed to repo
  Anyone with repo access = anyone with secret access
  Git history = permanent record of every secret value

  This fundamentally breaks GitOps for secrets.

SOLUTIONS COMPARISON:

  Option A: Sealed Secrets
    Encrypt Secret → SealedSecret CRD → commit to Git
    Cluster's Sealed Secrets controller decrypts
    + No external dependency
    + Works offline
    + Fully GitOps-native (everything in Git)
    - Controller's private key = ultimate secret (must protect/backup)
    - Secret rotation means re-sealing

  Option B: External Secrets Operator (5.4)
    External backend (Vault/AWS SM) = source of truth
    ExternalSecret CRD (no secret values) → safe to commit
    ESO syncs values from backend into K8s Secrets
    + Centralized secret management
    + Rotation handled by backend
    + No sensitive data in Git at all
    - Requires external backend infrastructure

  Option C: Helm + SOPS (not covered here)
    SOPS encrypts Helm values files
    + Works with existing Helm workflows
    - More complex key management

  RECOMMENDATION:
    Large teams / enterprises: ESO + Vault or AWS SM
    Small teams / simple infra: Sealed Secrets
    Both: valid in production, choose based on existing infra
```

---

### How Sealed Secrets Works

```
SEALED SECRETS MECHANISM
═══════════════════════════════════════════════════════════════

SETUP:
  1. Install sealed-secrets controller in cluster (kube-system)
     Controller generates a 4096-bit RSA keypair on first boot:
       Public key:  stored in Secret/sealed-secrets-key (in kube-system)
       Private key: stored in same Secret, NEVER leaves cluster
     Public key published: kubectl get secret sealed-secrets-key
                            -n kube-system -o jsonpath='{.data.tls\.crt}' | base64 -d

SEALING (developer's machine):
  Developer uses kubeseal CLI:
    kubeseal < secret.yaml > sealed-secret.yaml

  kubeseal:
    1. Fetches public key from cluster (or uses --cert flag for offline)
    2. Encrypts secret value with RSA-OAEP + AES-256-GCM hybrid encryption
       (RSA encrypts a random AES key, AES encrypts the actual data)
    3. Encryption is SCOPED:
       Default scope: namespace + secret name + field name baked into ciphertext
       → Encrypted value is only valid in specific namespace/name combination
       → Protects against copy-paste attacks between namespaces
    4. Outputs SealedSecret CRD YAML containing ciphertext

  Developer commits sealed-secret.yaml to Git — SAFE (ciphertext only)

UNSEALING (cluster-side):
  Argo CD / Flux applies SealedSecret YAML to cluster.
  Sealed Secrets controller watches SealedSecret objects.
  Controller decrypts using private key:
    1. Verifies namespace and name match the scope
    2. Decrypts AES key using RSA private key
    3. Decrypts data using AES key
    4. Creates regular Kubernetes Secret in the same namespace
  Pod mounts the regular Kubernetes Secret as normal.

KEY ROTATION:
  Controller can have multiple keys (key history maintained)
  New key generated on schedule (default: every 30 days)
  Old keys retained for decryption of old SealedSecrets
  Re-sealing with new key: kubeseal --rotate-all
```

---

### Sealed Secrets YAML and CLI

```yaml
# Input: regular Kubernetes Secret (DO NOT commit this)
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
  namespace: production
type: Opaque
stringData:
  password: supersecret123
  username: prod_user
```

```bash
# Install kubeseal CLI
KUBESEAL_VERSION=0.24.0
curl -sSL https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION}/kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz | tar xz
sudo install -m 755 kubeseal /usr/local/bin/

# Install controller
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets \
  sealed-secrets/sealed-secrets \
  -n kube-system \
  --set fullnameOverride=sealed-secrets-controller

# Seal a secret (fetches public key from cluster automatically)
kubeseal < secret.yaml > sealed-secret.yaml
# OR pipe:
kubectl create secret generic database-credentials \
  --from-literal=password=supersecret123 \
  --dry-run=client -o yaml | \
  kubeseal --format yaml > sealed-secret.yaml

# For CI/CD without cluster access (offline sealing):
# Fetch and store public key first:
kubeseal --fetch-cert \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system > pub-cert.pem

# Seal using stored public key (no cluster connection needed):
kubeseal --cert pub-cert.pem < secret.yaml > sealed-secret.yaml
```

```yaml
# Output: SealedSecret (SAFE to commit to Git)
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: database-credentials
  namespace: production           # scope — only works in this namespace
spec:
  encryptedData:
    password: AgA7x9K3mQ...      # RSA-OAEP + AES-256-GCM ciphertext
                                  # Cannot be decrypted without cluster private key
    username: AgBL9pR2nX...
  template:
    metadata:
      name: database-credentials
      namespace: production
    type: Opaque
```

---

### Scoping Modes

```bash
# Strict scope (default): namespace + secret name + key baked into ciphertext
kubeseal < secret.yaml > sealed.yaml
# Ciphertext is ONLY valid for:
#   namespace: production, name: database-credentials

# Namespace-wide scope: only namespace baked in (any name in namespace)
kubeseal --scope namespace-wide < secret.yaml > sealed.yaml

# Cluster-wide scope: no namespace/name binding (portable)
kubeseal --scope cluster-wide < secret.yaml > sealed.yaml
# ⚠️ Less secure — anyone who obtains this ciphertext can use it
#   in any namespace with any name

# Re-seal all secrets with latest key
kubeseal --re-encrypt --all-namespaces
```

---

### Backup and Recovery

```bash
# CRITICAL: Backup the sealed-secrets controller key
# If lost, cannot decrypt any SealedSecrets → must re-seal everything
kubectl get secret sealed-secrets-key \
  -n kube-system \
  -o yaml > sealed-secrets-master-key.yaml
# Store this securely (AWS Secrets Manager, Vault, encrypted S3)

# Restore key on new cluster
kubectl apply -f sealed-secrets-master-key.yaml
kubectl delete pod -n kube-system \
  -l name=sealed-secrets-controller
# Controller restarts and picks up restored key
# All existing SealedSecrets now decrypt correctly on new cluster
```

---

### 🎤 Short Crisp Interview Answer

> *"Sealed Secrets solves the GitOps secret problem — you can't commit plaintext Secrets to Git, but GitOps requires everything in Git. The kubeseal CLI fetches the cluster's RSA public key and encrypts a Kubernetes Secret manifest into a SealedSecret CRD. The ciphertext is scope-bound — namespace and secret name are baked in, so a sealed secret from production cannot be replayed into staging. Argo CD or Flux applies the SealedSecret YAML, and the Sealed Secrets controller decrypts it using its private key and creates a real Kubernetes Secret. The private key never leaves the cluster. The critical operational concern is backing up the controller's master key — if it's lost, all sealed secrets become undecryptable."*

---

### ⚠️ Gotchas

1. **Backup the master key or lose everything** — the sealed-secrets-key Secret in kube-system is the only thing that can decrypt all your SealedSecrets. Cluster rebuild without this key = all secrets need re-sealing.
2. **Strict scope prevents cross-namespace copy** — a SealedSecret sealed for `production/database-credentials` cannot be applied to `staging` namespace. Must re-seal for each namespace.
3. **Key rotation requires re-sealing** — old SealedSecrets sealed with old keys continue to work (old keys retained), but best practice is to re-seal all secrets with the current key periodically.
4. **Public key must be fetched from correct cluster** — if you seal against the wrong cluster's public key, the ciphertext cannot be decrypted by your target cluster.

---

### Connections
- Solves GitOps safety for Secrets → **5.2**
- Alternative: ExternalSecrets for GitOps → **5.4**
- Used with Argo CD / Flux GitOps workflows → **Category 9**

---

---

# 5.8 CSI Secret Store Driver — Secrets as Volumes

## 🔴 Advanced

### What it is in simple terms

The CSI Secret Store driver is a **Kubernetes CSI driver that mounts secrets from external providers directly as files in pod volumes**, bypassing the need to create a Kubernetes Secret object. The secret value is fetched at pod mount time directly from Vault, AWS Secrets Manager, or Azure Key Vault and written only to the pod's tmpfs — never stored in etcd.

---

### CSI Secret Store vs External Secrets Operator

```
TWO APPROACHES TO EXTERNAL SECRETS
═══════════════════════════════════════════════════════════════

EXTERNAL SECRETS OPERATOR (ESO):
  Flow: External SM → K8s Secret object (in etcd) → Pod env/volume
  Secret stored in: etcd (even if encrypted)
  Access: Standard kubectl get secret
  Sync: Periodic (refreshInterval)
  K8s Secret created: YES

CSI SECRET STORE DRIVER:
  Flow: External SM → Pod tmpfs volume (DIRECT — no etcd)
  Secret stored in: Pod's tmpfs ONLY (never in etcd)
  Access: Only visible inside the pod as files
  Sync: At pod mount time (latest version at pod start)
  K8s Secret created: Optional (sync-as-Secret feature)

WHICH TO USE:
  Use ESO when:
    + Application uses env vars (CSI only does volumes)
    + Need secret accessible to multiple pods simultaneously
    + Want Kubernetes-native RBAC on the secret
    + Secret needs to be referenced in init containers before main starts

  Use CSI Secret Store when:
    + Compliance: secret must NEVER appear in etcd
    + Secret file rotation needed at pod level without restart
    + Direct mount is sufficient (files in container)
    + Secrets too sensitive for even encrypted etcd
    + Financial/healthcare compliance (HIPAA, PCI-DSS)
```

---

### CSI Secret Store Architecture

```
CSI SECRET STORE — HOW IT WORKS
═══════════════════════════════════════════════════════════════

COMPONENTS:
  secrets-store-csi-driver:  DaemonSet on every node
                             Acts as CSI node plugin
  Provider plugin:           DaemonSet for each backend
                             (aws provider, vault provider, etc.)

POD MOUNT FLOW:
  1. Pod spec references a SecretProviderClass volume
  2. Kubelet calls CSI node driver: NodePublishVolume
  3. CSI driver calls provider plugin via gRPC Unix socket:
     GetSecrets(SecretProviderClass config)
  4. Provider plugin authenticates to backend (AWS SM / Vault)
     using pod's projected ServiceAccount token (IRSA on EKS)
  5. Provider fetches secret value from backend
  6. CSI driver writes secret value to pod's tmpfs mount
  7. Pod container sees /mnt/secrets/password as a regular file
  8. Secret value NEVER touched etcd — only in pod's RAM

UNMOUNT:
  When pod terminates → tmpfs unmounted → secret data gone from node
  No cleanup needed in external backend
```

---

### CSI Secret Store YAML — AWS Secrets Manager

```yaml
# Step 1: SecretProviderClass — defines what to fetch and from where
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: aws-secrets
  namespace: production
spec:
  provider: aws                            # aws, vault, azure, gcp

  parameters:
    objects: |
      - objectName: "prod/database/credentials"   # AWS SM secret name
        objectType: "secretsmanager"
        jmesPath:                          # extract JSON property
        - path: password
          objectAlias: db-password         # filename in pod
        - path: username
          objectAlias: db-username

      - objectName: "prod/tls/certificate"
        objectType: "secretsmanager"
        objectAlias: tls.crt

      - objectName: "/production/app/api-key"     # SSM Parameter
        objectType: "ssmparameter"
        objectAlias: api-key

  # OPTIONAL: Also sync as Kubernetes Secret (for env var use)
  secretObjects:
  - secretName: db-credentials-synced     # K8s Secret to create/update
    type: Opaque
    data:
    - objectName: db-password             # from objects above
      key: password
    - objectName: db-username
      key: username

---
# Step 2: Pod using the SecretProviderClass as a CSI volume
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  namespace: production
spec:
  serviceAccountName: myapp-sa           # must have IRSA for AWS SM access

  containers:
  - name: app
    image: myapp:v1
    volumeMounts:
    - name: secrets-vol
      mountPath: /mnt/secrets
      readOnly: true

    # If secretObjects defined above:
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials-synced    # the synced K8s Secret
          key: password

  volumes:
  - name: secrets-vol
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: aws-secrets  # references above CRD

# Result in container at /mnt/secrets/:
#   db-password    ← value from prod/database/credentials.password
#   db-username    ← value from prod/database/credentials.username
#   tls.crt        ← value from prod/tls/certificate
#   api-key        ← value from /production/app/api-key
#
# All on tmpfs — never in etcd
```

---

### Installation

```bash
# Install CSI Secret Store driver
helm repo add secrets-store-csi-driver \
  https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts

helm install csi-secrets-store \
  secrets-store-csi-driver/secrets-store-csi-driver \
  --namespace kube-system \
  --set syncSecret.enabled=true \      # enable K8s Secret sync
  --set enableSecretRotation=true \    # enable rotation (watch below)
  --set rotationPollInterval=2m        # check backend every 2 min

# Install AWS provider
kubectl apply -f https://raw.githubusercontent.com/aws/secrets-store-csi-driver-provider-aws/main/deployment/aws-provider-installer.yaml

# Verify
kubectl get pods -n kube-system | grep -E "csi-secrets|csi-provider"

# Check SecretProviderClass status
kubectl get secretproviderclass -n production
kubectl describe secretproviderclass aws-secrets -n production
```

---

### 🎤 Short Crisp Interview Answer

> *"The CSI Secret Store driver mounts secrets from external providers directly as tmpfs volumes in pods, bypassing etcd entirely. A SecretProviderClass CRD defines what to fetch from which backend. At pod mount time, the CSI node driver calls the provider plugin — AWS, Vault, Azure — which fetches the secret using the pod's service account identity via IRSA on EKS, and writes it to the pod's tmpfs. The secret never touches etcd. This is the strongest security model because even encrypted etcd can theoretically be decrypted if the KMS key is compromised. It's used in regulated industries where compliance requires secrets never be stored in cluster storage. The trade-off vs External Secrets Operator is that it's volume-only — no env var injection without the optional secretObjects sync feature."*

---

### ⚠️ Gotchas

1. **Pod fails to start if provider is unavailable** — unlike ESO where the K8s Secret exists and pods start even if ESO is down, CSI fetches at mount time. If AWS SM is unreachable, pod stays in ContainerCreating indefinitely.
2. **Rotation doesn't restart pod by default** — when enableSecretRotation is true, the CSI driver updates the file in the pod's tmpfs when the secret changes. But apps reading the file at startup (not continuously) won't see the change. Need app-level inotify watching or SIGTERM reload.
3. **IRSA must be on the pod's ServiceAccount** — the CSI provider uses the pod's projected SA token to authenticate. The pod's ServiceAccount needs the IAM role with secretsmanager:GetSecretValue permission.

---

### Connections
- Alternative to ESO for external secrets → **5.4**
- IRSA for AWS authentication → **Category 12**
- CSI driver architecture (same pattern) → **Category 4 (4.11)**
- Secret rotation with CSI driver → **5.9**

---

---

# 5.9 Secret Rotation & Pod Reload Strategies

## 🔴 Advanced

### What it is in simple terms

Secret rotation — changing a secret value to limit blast radius from a compromise — is only useful if the new value actually reaches running pods. Kubernetes provides no automatic pod reload mechanism. This topic covers the complete set of strategies for propagating rotated secrets to running workloads.

---

### The Rotation Problem

```
WHY SECRET ROTATION IS HARD IN KUBERNETES
═══════════════════════════════════════════════════════════════

Rotation pipeline:
  1. Rotate secret in backend (AWS SM auto-rotates, or manual)
  2. New value must reach Kubernetes (ESO sync, CSI refresh)
  3. Running pods must start using the new value

  STEP 3 is the hard part.

Methods of secret injection and their rotation behavior:

Method              Auto-update?  What's needed for rotation
──────────────────────────────────────────────────────────────
Env var (secret)    NO            Pod restart required
Volume mount        YES (~60s)    App must re-read file
CSI volume          YES*          App must re-read file (*if rotation enabled)
ESO → env var       NO            Pod restart after ESO sync
ESO → volume        YES (~60s)    App must re-read file after ESO sync

ROTATION CHALLENGE SCENARIOS:

Scenario A: DB password rotated in AWS SM
  1. AWS SM rotates password
  2. ESO syncs new password to K8s Secret (after refreshInterval)
  3. Running pods still use OLD password in env var
  4. DB now rejects old password
  5. Application errors until pods restart
  ← This is a production outage if not handled properly

Scenario B: TLS certificate expiry (cert-manager renews)
  1. cert-manager creates new TLS Secret
  2. Pods with volume mount see new cert files within 60s
  3. nginx must reload: nginx -s reload (SIGHUP)
  4. If nginx doesn't reload: serving expired cert

Scenario C: API key rotated for security
  1. New key provisioned
  2. Brief window where both old and new work (in provider)
  3. New key synced to Kubernetes
  4. Pods restarted to pick up new key
  5. Provider disables old key
```

---

### Strategy 1: Rolling Restart (Simplest, Most Common)

```bash
# Strategy: update K8s Secret → trigger rolling restart
# Best for: env var injection, simple setups

# 1. Update the Secret (manually or via ESO sync)
kubectl create secret generic db-credentials \
  --from-literal=password=newpassword123 \
  --dry-run=client -o yaml | kubectl apply -f -

# 2. Trigger rolling restart of all pods using this secret
kubectl rollout restart deployment/myapp -n production
kubectl rollout restart statefulset/mysql -n production

# 3. Monitor rollout
kubectl rollout status deployment/myapp -n production

# AUTOMATION: Watch Secret, trigger restart on change
# Use a custom controller or Reloader (below)

# TIMING CONCERN:
# Brief window where some pods have old secret, others have new
# For DB password rotation: use dual-write period
#   1. Add new password to DB (old still works)
#   2. Rotate secret to new password
#   3. Restart pods
#   4. Remove old password from DB
```

---

### Strategy 2: Reloader Controller (Automated Restart)

```bash
# Reloader: open-source controller that watches Secrets/ConfigMaps
# and automatically triggers rolling restarts when they change
# GitHub: stakater/Reloader

# Install Reloader
helm repo add stakater https://stakater.github.io/stakater-charts
helm install reloader stakater/reloader -n reloader --create-namespace

# Annotate Deployment to watch specific secrets
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    # Watch specific secret:
    secret.reloader.stakater.com/reload: "database-credentials,tls-cert"
    # Watch specific configmap:
    configmap.reloader.stakater.com/reload: "app-config"
    # Watch ALL secrets and configmaps (broad):
    reloader.stakater.com/auto: "true"
spec:
  # rest of deployment spec

# How Reloader works:
# Watches K8s Secret/ConfigMap objects for changes
# When watched object changes → triggers rolling restart
# Uses same mechanism as kubectl rollout restart (annotation bump)

# Reloader + ESO combination:
# 1. AWS SM rotates secret
# 2. ESO syncs to K8s Secret (after refreshInterval)
# 3. Reloader detects K8s Secret changed
# 4. Reloader triggers rolling restart of annotated Deployments
# 5. New pods start with new secret value
# Full automation with ~refreshInterval delay
```

---

### Strategy 3: Volume Mount + App-Level File Watch

```python
# Strategy: volume mount + inotify file watcher in app
# Best for: zero-downtime rotation, high-frequency rotation

# Python example — watch for secret file changes
import inotify.adapters
import threading
import os

SECRET_PATH = "/etc/secrets/db-password"
_db_password = None

def load_secret():
    global _db_password
    with open(SECRET_PATH, 'r') as f:
        _db_password = f.read().strip()
    print(f"Loaded new secret (length: {len(_db_password)})")

def watch_secret():
    i = inotify.adapters.Inotify()
    i.add_watch(os.path.dirname(SECRET_PATH))
    load_secret()  # load initial value
    for event in i.event_gen(yield_nones=False):
        (_, type_names, path, filename) = event
        # K8s updates secrets via symlink swap — watch for IN_CREATE/IN_MOVED_TO
        if 'IN_CREATE' in type_names or 'IN_MOVED_TO' in type_names:
            if filename == '..data':  # K8s projected symlink
                load_secret()

# Start file watcher in background thread
watcher_thread = threading.Thread(target=watch_secret, daemon=True)
watcher_thread.start()

# Go example — inotify watcher
# Use fsnotify package: github.com/fsnotify/fsnotify
# Watch directory for IN_CREATE events on "..data" symlink
```

```yaml
# nginx example — reload via SIGHUP after cert rotation
# Use init container + sidecar pattern
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    volumeMounts:
    - name: tls-cert
      mountPath: /etc/nginx/ssl
      readOnly: true

  - name: cert-reloader
    image: bitnami/kubectl:latest
    command:
    - /bin/sh
    - -c
    - |
      # Watch for symlink change in mounted secret dir
      CERT_DIR="/etc/nginx/ssl"
      CURRENT=$(readlink "${CERT_DIR}/..data" 2>/dev/null)
      while true; do
        sleep 10
        NEW=$(readlink "${CERT_DIR}/..data" 2>/dev/null)
        if [ "$CURRENT" != "$NEW" ]; then
          echo "Certificate changed, reloading nginx"
          nginx -s reload  # SIGHUP to master process
          CURRENT=$NEW
        fi
      done
    volumeMounts:
    - name: tls-cert
      mountPath: /etc/nginx/ssl
      readOnly: true
```

---

### Strategy 4: CSI Driver Rotation (Auto File Update)

```yaml
# CSI driver with rotation enabled (set at driver install time):
#   --set enableSecretRotation=true
#   --set rotationPollInterval=2m

# When backend secret changes:
#   CSI driver polls backend every rotationPollInterval
#   Detects new version → updates tmpfs file in pod
#   App must still re-read the file (inotify or periodic re-read)
#   No pod restart needed — file updated in place

# Monitor rotation events
kubectl get events -n production --field-selector reason=SecretRotated

# CSI driver also supports:
#   --set filteredWatchSecret=true
#   Only rotate secrets that have changed (reduces API calls)
```

---

### Strategy 5: Dual-Write Pattern for DB Password Rotation

```
DUAL-WRITE — ZERO DOWNTIME DB CREDENTIAL ROTATION
═══════════════════════════════════════════════════════════════

Step 1: Add new password to database (both passwords work)
  ALTER USER prod_user IDENTIFIED BY 'newpassword123'
    RETAIN CURRENT PASSWORD;
  # Or: create new user, grant permissions

Step 2: Update K8s Secret with new password
  kubectl create secret generic db-credentials \
    --from-literal=password=newpassword123 \
    --dry-run=client -o yaml | kubectl apply -f -

Step 3: Rolling restart of pods
  kubectl rollout restart deployment/myapp

Step 4: Monitor — verify new pods using new password
  kubectl rollout status deployment/myapp
  # Watch application logs for DB connection errors

Step 5: Remove old password from database
  ALTER USER prod_user DISCARD OLD PASSWORD;
  # Old password now invalid
  # Only new password works
  # All pods already using new password ✓

ZERO DOWNTIME: At no point is service interrupted.
  During rollout: some pods use old password (still works)
                  some pods use new password (works)
  After rollout: all pods use new password
  After step 5: old password disabled
```

---

### Rotation Timeline with ESO + Reloader

```
AUTOMATED ROTATION WITH ESO + RELOADER
═══════════════════════════════════════════════════════════════

t=0:   AWS SM rotates DB password (scheduled rotation)
t=0:   AWS SM Lambda rotator updates DB with new password
       (dual-write: both old and new work during rotation window)
t=30m: ESO refreshInterval elapses
       ESO calls AWS SM: GetSecretValue
       ESO detects new version
       ESO updates K8s Secret object
t=30m: Reloader detects K8s Secret changed
       Reloader triggers rolling restart of myapp Deployment
t=32m: New pods start with new DB password from Secret env var
       (or new mounted file, if using volume mount)
t=35m: Rolling restart completes
       All pods using new password
t=35m: AWS SM Lambda rotator removes old DB password
       Rotation complete ✓

TOTAL WINDOW: ~35 minutes for full rotation
  Reduce by shortening ESO refreshInterval
  refreshInterval: "5m" → rotation complete in ~7 minutes
  Trade-off: more AWS SM API calls (charged per 10k calls)

REDUCING THE WINDOW:
  Option 1: refreshInterval: "5m" (adequate for most)
  Option 2: Push-based rotation via EventBridge → Lambda →
            update K8s Secret directly (instant, complex)
  Option 3: CSI Secret Store with enableSecretRotation: true
            + 2m rotationPollInterval (~2-3 min total)
```

---

### 🎤 Short Crisp Interview Answer

> *"Secret rotation is only valuable if the new value actually reaches running pods. For env var injection, the simplest approach is: update the Secret, then trigger a rolling restart with kubectl rollout restart. The Reloader controller automates this — it watches Secrets and ConfigMaps and triggers rolling restarts on any change, combining well with ESO's periodic sync. For zero-downtime rotation, use volume mounts with inotify file watchers — the app detects the file change and reloads its connection pool without restart. For database passwords, the dual-write pattern is critical: add the new password to the DB alongside the old one, rotate the secret, do the rolling restart, then remove the old password. This ensures no connection errors during the rotation window."*

---

### ⚠️ Gotchas

1. **Volume mount updates don't help if app only reads at startup** — most database connection pool libraries read credentials once at startup. Volume mount auto-update is useless unless the app explicitly polls or uses inotify to detect file changes.
2. **ESO refreshInterval is not a rotation trigger** — ESO will sync the new value within one interval, but pods still need a restart or file-watch reload to use it. The full rotation time = ESO sync + pod restart time.
3. **subPath kills rotation** — volume mounts using subPath do not auto-update. Secret rotation in the backend never reaches the pod's file. Always use full directory mounts for rotating secrets.
4. **Symlink watches, not file watches** — Kubernetes updates secret files atomically via symlink swap. Watch for the `..data` symlink changing, not the file itself. Apps that watch the raw file path may miss updates because the inode changes.

---

### Common Interview Questions

**Q: Walk me through how you would rotate a database password in Kubernetes with zero downtime.**
> First, add the new password to the database alongside the old one using a dual-write approach — both passwords accepted simultaneously. Second, update the Kubernetes Secret with the new password. Third, trigger a rolling restart of the Deployment. During the rollout, some pods use the old password and some use the new — both work because of dual-write. Once all pods have restarted and are using the new password, remove the old password from the database. No service interruption at any point.

**Q: How does Reloader work and why would you use it?**
> Reloader is an open-source controller that watches Kubernetes Secrets and ConfigMaps. When an annotated Secret or ConfigMap changes, Reloader automatically triggers a rolling restart of all Deployments, StatefulSets, and DaemonSets that reference it via annotations. Combined with External Secrets Operator, it creates a full automation pipeline: external backend rotates a secret, ESO syncs it to a Kubernetes Secret within the refreshInterval, Reloader detects the change and triggers pod restarts — all without manual intervention.

---

### Connections
- ESO as the sync mechanism feeding rotation → **5.4**
- CSI Secret Store with rotation enabled → **5.8**
- Volume mount auto-update mechanism → **5.1**
- subPath blocking updates → **5.1 gotcha, 5.9 gotcha**
- Rolling restart mechanics → **Category 2 (2.3 Deployment)**

---

---

# 🏁 Category 5 — Complete Configuration & Secrets Map

```
CONFIGURATION & SECRETS DECISION TREE
═══════════════════════════════════════════════════════════════════

DATA TYPE: Non-sensitive config?
  YES → ConfigMap
    Inject how?
      Rarely changes, simple values → envFrom / env (env var)
      Changes often, want live reload → volume mount (auto-updates ~60s)
      Fixed release version, scale matters → immutable: true + versioned name
      GitOps with Argo CD / Flux → regular ConfigMap (no sensitive data)
  NO → Sensitive data → Secret

SECRET STORAGE: Where is source of truth?
  Simple/small cluster, no external infra →  Native K8s Secret
    Secure it:
      RBAC restrict GET/LIST access
      EncryptionConfiguration → KMS provider (non-EKS)
      EKS: enable KMS envelope encryption at cluster creation
  Enterprise, central management → External Secret Manager
    AWS Secrets Manager / Parameter Store → ESO + IRSA
    HashiCorp Vault → ESO (Vault provider) or CSI Secret Store
    Compliance: secret must never enter etcd → CSI Secret Store
    GitOps + external manager → ExternalSecret CRD in Git (safe)

GITOPS: How to put secrets in Git safely?
  External SM as source of truth → ExternalSecret CRD (no values in Git)
  No external SM, everything in Git → Sealed Secrets
    kubeseal encrypts → SealedSecret CRD → safe to commit
    Cluster's private key does decryption

INJECTION METHOD: How do pods receive secrets?
  Simple values, restart acceptable → env var (secretKeyRef / secretRef)
  Sensitive, security-conscious → volume mount (tmpfs, no disk)
  Multiple sources, clean layout → projected volume
  Compliance: never in etcd → CSI Secret Store volume
  Immutable, performance at scale → immutable: true

ROTATION: How to propagate rotated secrets?
  Env var injection → rolling restart (manual or Reloader)
  Volume mount → app file watch (inotify) or restart
  ESO + Reloader → automated: SM rotates → ESO syncs → Reloader restarts
  DB passwords → dual-write pattern (zero downtime)
  CSI + rotation enabled → auto file update (app still re-reads)
```

---

# Quick Reference — Category 5 Cheat Sheet

| Topic | Key Facts |
|-------|-----------|
| **ConfigMap** | Non-sensitive config, env var (no auto-update) vs volume (auto-update ~60s), 1MB limit |
| **subPath** | Mounts single key at exact path BUT breaks auto-update — avoid for live reload |
| **Secret types** | Opaque (general), tls, dockerconfigjson, service-account-token, ssh-auth, basic-auth |
| **base64** | NOT encryption — just format. Trivially decoded. Real security = RBAC + encryption at rest |
| **stringData** | Write-only plaintext convenience field, auto-converted to base64, never in GET output |
| **EncryptionConfiguration** | Encrypts Secrets in etcd. KMS provider recommended. Existing secrets need force-rewrite |
| **EKS encryption** | Must be enabled at cluster creation with KMS key ARN. Cannot add after creation |
| **ESO** | Syncs external SM → K8s Secret. SecretStore + ExternalSecret CRDs. refreshInterval = sync lag |
| **Projected Volume** | Combines ConfigMap + Secret + SA token + downwardAPI into one mount |
| **SA token (modern)** | Short-lived, audience-bound, pod-bound, auto-rotated by kubelet at 80% lifetime |
| **Immutable** | immutable: true = no data changes, no watches by kubelet, 20-40% API Server load reduction |
| **Sealed Secrets** | kubeseal encrypts with cluster RSA public key → SealedSecret CRD → safe in Git |
| **Sealed Secrets master key** | MUST be backed up. Loss = all SealedSecrets permanently unreadable |
| **CSI Secret Store** | Secrets never touch etcd. Fetched at mount time. Compliance use case |
| **Reloader** | Controller that auto-restarts pods when Secret/ConfigMap changes |
| **Dual-write** | Add new password to DB, rotate K8s secret, restart pods, remove old password |
| **inotify rotation** | Watch `..data` symlink (not file path) for K8s atomic volume updates |

---

## Key Numbers to Remember

| Fact | Value |
|------|-------|
| ConfigMap/Secret volume update time | ~60 seconds (kubelet sync period) |
| ConfigMap/Secret size limit | 1 MB (etcd object limit) |
| Projected SA token default expiry | 3600s (1 hour) — configurable |
| SA token auto-rotation trigger | 80% of expirationSeconds |
| Immutable watch reduction (large clusters) | 20-40% API Server CPU reduction |
| Sealed Secrets key size | 4096-bit RSA |
| Sealed Secrets default key rotation | Every 30 days (new key, old retained) |
| ESO refreshInterval (typical production) | 1h (routine), 5m (fast rotation) |
| CSI rotation poll interval (typical) | 2 minutes |
