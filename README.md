# NBomber Studio Helm Chart

A Helm chart for deploying NBomber Studio on Kubernetes.

[![Artifact Hub](https://img.shields.io/endpoint?url=https://artifacthub.io/badge/repository/nbomber-studio)](https://artifacthub.io/packages/search?repo=nbomber-studio)

## Overview

This chart deploys:
- An NBomber Studio Deployment
- A ServiceAccount, Role, and RoleBinding scoped to the test namespace
- A dedicated test namespace (default: `nbomber-tests`) for running NBomber test Jobs and Pods
- A ClusterIP service for internal access
- Optional ConfigMap and Secret for configuration overrides
- Optional Ingress resource for external HTTPS access

NBomber Studio automatically creates a dedicated Kubernetes namespace (default: `nbomber-tests`) where it deploys and runs NBomber test Jobs and Pods.

**Dependencies:** NBomber Studio requires [PostgreSQL with the TimescaleDB extension](https://docs.timescale.com/) for storing test metrics and results. Make sure a TimescaleDB-enabled PostgreSQL instance is accessible before installing this chart. The NBomber team provides a ready-to-use [Timescale Helm chart on Artifact Hub](https://artifacthub.io/packages/helm/timescale/timescale) to simplify this setup.

It is designed for simplicity and can be used in development, testing, or production environments.

### Installation

**Step 1 — Add the Helm repository**

```bash
helm repo add nbomber-studio https://pragmaticflow.github.io/nbomber-studio-helm/
helm repo update
```

**Step 2 — Install the chart with the required Secret**

> **A PostgreSQL connection string is required.** Set `config.secret.enabled=true` with `config.secret.data.PostgreSql.ConnectionString`, or set `config.existingSecret` to the name of an existing Secret. **Make sure to use the full K8s domain name for the PostgreSQL host** (e.g. `nbomber-timescale.default.svc.cluster.local`) so that NBomber test Jobs running in a separate namespace can reach PostgreSQL.

**Option A — without authentication (for local/dev environments only):**

Inline with `--set` flags:

> Use the full Kubernetes domain hostname for the PostgreSQL host (`<service>.<namespace>.svc.cluster.local`). A short hostname will fail because NBomber test Jobs run in a separate namespace.

> Set `Auth.Enabled=false` to skip JWT configuration entirely.

```bash
helm install nbomber-studio nbomber-studio/nbomber-studio \
  --set config.secret.enabled=true \
  --set-string "config.secret.data.PostgreSql.ConnectionString=Host=<timescaledb-service>.<namespace>.svc.cluster.local;Port=5432;Username=<user>;Password=<password>;Database=<db>" \
  --set config.secret.data.Auth.Enabled=false
```

Or in a `values.yaml` file:

```yaml
config:
  secret:
    enabled: true
    data:
      PostgreSql:
        ConnectionString: "Host=<timescaledb-service>.<namespace>.svc.cluster.local;Port=5432;Username=<user>;Password=<password>;Database=<db>;Pooling=true;Maximum Pool Size=300;"
      Auth:
        Enabled: false
```

```bash
helm install nbomber-studio nbomber-studio/nbomber-studio -f my-values.yaml
```

**Option B — with authentication enabled (recommended for production):**

Inline with `--set` flags:

```bash
helm install nbomber-studio nbomber-studio/nbomber-studio \
  --set config.secret.enabled=true \
  --set-string "config.secret.data.PostgreSql.ConnectionString=Host=<timescaledb-service>.<namespace>.svc.cluster.local;Port=5432;Username=<user>;Password=<password>;Database=<db>" \
  --set-string "config.secret.data.Auth.JwtSecret=<your-secure-jwt-secret>"
```

Using a `values.yaml` file (recommended for complex connection strings):

```yaml
config:
  secret:
    enabled: true
    data:
      PostgreSql:
        ConnectionString: "Host=<timescaledb-service>.<namespace>.svc.cluster.local;Port=5432;Username=<user>;Password=<password>;Database=<db>;Pooling=true;Maximum Pool Size=300;"
      Auth:
        JwtSecret: "<your-secure-jwt-secret>"  # generate with: openssl rand -base64 32
```

**Upgrade**

```bash
helm upgrade nbomber-studio nbomber-studio/nbomber-studio -f my-values.yaml
```

### Parameters

The following table lists the configurable parameters of the chart and their default values:

| Parameter | Description | Default |
|-----------|-------------|---------|
| `nameOverride` | Override the chart name | `""` |
| `fullnameOverride` | Override the full name of resources | `""` |
| `image.repository` | NBomber Studio image repository | `nbomberdocker/nbomber-studio` |
| `image.tag` | Image tag (defaults to Chart appVersion) | `""` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `service.type` | Kubernetes service type | `ClusterIP` |
| `service.port` | Service port | `80` |
| `resources.limits.memory` | Memory limit | `2Gi` |
| `resources.requests.memory` | Memory request | `500Mi` |
| `config.configMap.enabled` | Create ConfigMap for config overrides | `false` |
| `config.configMap.data` | ConfigMap data (JSON structure) | `{}` |
| `config.secret.enabled` | Create Secret for config overrides | `false` |
| `config.secret.data` | Secret data (JSON structure) | `{}` |
| `config.secret.data.License` | NBomber Studio license key | `""` |
| `config.existingConfigMap` | Use existing ConfigMap | `""` |
| `config.existingSecret` | Use existing Secret | `""` |
| `ingress.enabled` | Enable ingress resource | `false` |
| `ingress.className` | Ingress class name | `"traefik"` |
| `ingress.annotations` | Ingress annotations | `{}` |
| `ingress.hosts` | Ingress host configurations | see [values.yaml](values.yaml) |
| `ingress.tls` | Ingress TLS configuration | `[]` |
| `serviceAccount.create` | Create a ServiceAccount for the Studio pod | `true` |
| `serviceAccount.name` | ServiceAccount name (auto-generated if empty) | `""` |
| `serviceAccount.annotations` | Annotations to add to the ServiceAccount | `{}` |
| `testNamespace.name` | Namespace where NBomber test Jobs/Pods are deployed | `nbomber-tests` |
| `testNamespace.create` | Create the test namespace (set to `false` if it already exists) | `true` |
| `nodeSelector` | Node labels for pod assignment | `{}` |
| `affinity` | Affinity rules | `{}` |
| `tolerations` | Tolerations for pod assignment | `[]` |

## RBAC and Test Namespace

This chart automatically provisions the Kubernetes RBAC resources needed for NBomber Studio to manage test workloads:

| Resource | Name | Scope |
|----------|------|-------|
| Namespace | `nbomber-tests` (configurable) | cluster |
| ServiceAccount | `<release-name>-nbomber-studio` | Studio's namespace |
| Role | `<release-name>-nbomber-studio-test-manager` | test namespace |
| RoleBinding | `<release-name>-nbomber-studio-test-manager` | test namespace |

The Role grants the Studio pod permissions to manage the following resources **only within the test namespace**:

- `pods`, `pods/log` — get, list, watch, create, delete
- `secrets` — get, list, create, update, patch, delete
- `services` — get, list, create, delete
- `deployments` (apps) — get, list, watch, create, delete
- `jobs` (batch) — get, list, watch, create, delete

To use an existing namespace instead of creating a new one:

```yaml
testNamespace:
  name: "my-existing-namespace"
  create: false
```

To use an existing ServiceAccount:

```yaml
serviceAccount:
  create: false
  name: "my-service-account"
```

## Configuration Overrides

This Helm chart allows you to override configuration values using Kubernetes ConfigMaps and Secrets. The configuration override system works as follows:

1. **ConfigMap** - For non-sensitive settings (logging levels, feature flags, etc.)
2. **Secret** - For sensitive settings (database passwords, JWT secrets, etc.)
3. Both are mounted into the container.

### Usage

#### Option 1: Create New Secret/ConfigMap (Recommended)

Override sensitive settings using a Secret:

```yaml
config:
  secret:
    enabled: true
    data:
      PostgreSql:
        ConnectionString: "Host=nbomber-timescale.default.svc.cluster.local;Port=5432;Username=user;Password=pass;Database=mydb"
      Auth:
        JwtSecret: "your-secure-jwt-secret"
```

> **Important:** Always use the full Kubernetes FQDN for the PostgreSQL host:
> `<service>.<namespace>.svc.cluster.local`
>
> NBomber test Jobs run in a dedicated test namespace (see `testNamespace`), which is separate from the namespace where PostgreSQL is installed. A short hostname resolves only within the same namespace and will cause connection failures from the test Jobs.

Override non-sensitive settings using a ConfigMap:

```yaml
config:
  configMap:
    enabled: true
    data:
      Logger:
        MinimumLogLevel: "Debug"
```

#### Option 2: Use Existing Secret/ConfigMap

If you manage secrets externally (e.g., via Sealed Secrets, External Secrets Operator):

```yaml
config:
  existingSecret: "my-nbomber-secret"
  existingConfigMap: "my-nbomber-config"
```

#### Option 3: Use kubectl to Create Secret

Create a secret from command line:

```bash
kubectl create secret generic nbomber-studio-config \
  --from-literal=config-override.json='{
    "PostgreSql": {
      "ConnectionString": "Host=nbomber-timescale.default.svc.cluster.local;Port=5432;..."
    },
    "Auth": {
      "JwtSecret": "my-secret-key"
    }
  }'
```

Then reference it:

```yaml
config:
  existingSecret: "nbomber-studio-config"
```

### Configuration Structure

The override configuration follows the same JSON structure as the default `config.json`:

```json
{
  "PostgreSql": {
    "ConnectionString": "Host=nbomber-timescale.default.svc.cluster.local;Port=5432;..."
  },
  "Logger": {
    "MinimumLogLevel": "Information"
  },
  "Auth": {
    "Enabled": true,
    "JwtSecret": "your-secret-here",    
    "StaticUserAuth": {
      "Users": [
        {
          "Email": "admin@admin",
          "Hash": "$2y$10$...",
          "UserName": "admin"
        }
      ]
    }
  }
}
```

### Mounted Files

When enabled, the following files are mounted in the container:

- `/app/config-override-cm.json` - ConfigMap overrides
- `/app/config-override-secret.json` - Secret overrides

### License Key

A license key is required to use NBomber Studio. Set it via the `License` field inside the secret data:

```yaml
config:
  secret:
    enabled: true
    data:
      License: "<your-license-key>"
      PostgreSql:
        ConnectionString: "Host=<timescaledb-service>.<namespace>.svc.cluster.local;Port=5432;Username=<user>;Password=<password>;Database=<db>"
      Auth:
        JwtSecret: "<your-secure-jwt-secret>"
```

Or with `--set`:

```bash
helm install nbomber-studio nbomber-studio/nbomber-studio \
  --set config.secret.enabled=true \
  --set-string "config.secret.data.License=<your-license-key>" \
  --set-string "config.secret.data.PostgreSql.ConnectionString=Host=<timescaledb-service>.<namespace>.svc.cluster.local;Port=5432;Username=<user>;Password=<password>;Database=<db>" \
  --set-string "config.secret.data.Auth.JwtSecret=<your-secure-jwt-secret>"
```

### Security Best Practices

1. **Always use Secrets for sensitive data** like passwords, JWT secrets, connection strings
2. **Use ConfigMaps for non-sensitive data** like log levels, timeouts, feature flags
3. **Consider using external secret management** tools like:
   - Sealed Secrets
   - External Secrets Operator
   - HashiCorp Vault
   - AWS Secrets Manager / Azure Key Vault / GCP Secret Manager

## Ingress

To expose NBomber Studio externally with HTTPS/TLS, enable the ingress resource:

```yaml
ingress:
  enabled: true
  className: "traefik"  # or "nginx", depending on your ingress controller
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
  hosts:
    - host: nbomber-studio.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: nbomber-studio-tls
      hosts:
        - nbomber-studio.example.com
```

#### Using NGINX Ingress Controller

```yaml
ingress:
  enabled: true
  className: "nginx"
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
  hosts:
    - host: nbomber-studio.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: nbomber-studio-tls
      hosts:
        - nbomber-studio.example.com
```

#### Using cert-manager for Automatic TLS

```yaml
ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: nbomber-studio.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: nbomber-studio-tls
      hosts:
        - nbomber-studio.example.com
```

## Examples

See [values-example.yaml](values-example.yaml) for complete examples.

See [HTTPS-SETUP.md](HTTPS-SETUP.md) for a detailed guide on enabling HTTPS/TLS.

#### Example: Production Database Override

```yaml
config:
  secret:
    enabled: true
    data:
      PostgreSql:
        ConnectionString: "Host=prod-timescaledb.database.svc.cluster.local;Port=5432;Username=nbomber_prod;Password=SecureP@ssw0rd;Database=nb_studio_prod;Pooling=true;Maximum Pool Size=300;"
      Auth:
        JwtSecret: "prod-jwt-secret-generate-with-openssl-rand-base64-32"
        JwtExpireSec: 3600
```

#### Example: Debug Logging

```yaml
config:
  configMap:
    enabled: true
    data:
      Logger:
        MinimumLogLevel: "Debug"
```
