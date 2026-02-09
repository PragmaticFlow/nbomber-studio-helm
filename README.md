# NBomber Studio Helm Chart

A Helm chart for deploying NBomber Studio on Kubernetes.

## Overview

This chart deploys:
- An NBomber Studio Deployment
- A ClusterIP service for internal access
- Optional ConfigMap and Secret for configuration overrides
- Optional Ingress resource for external HTTPS access

It is designed for simplicity and can be used in development, testing, or production environments.

### Installation

Add repository
```bash
helm repo add nbomber-studio https://pragmaticflow.github.io/nbomber-studio-helm/
```

Install chart
```bash
helm install nbomber-studio nbomber-studio/nbomber-studio
```

Install with custom values
```bash
helm install nbomber-studio nbomber-studio/nbomber-studio -f my-values.yaml
```

Upgrade
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
| `config.existingConfigMap` | Use existing ConfigMap | `""` |
| `config.existingSecret` | Use existing Secret | `""` |
| `ingress.enabled` | Enable ingress resource | `false` |
| `ingress.className` | Ingress class name | `"traefik"` |
| `ingress.annotations` | Ingress annotations | `{}` |
| `ingress.hosts` | Ingress host configurations | see [values.yaml](values.yaml) |
| `ingress.tls` | Ingress TLS configuration | `[]` |
| `nodeSelector` | Node labels for pod assignment | `{}` |
| `affinity` | Affinity rules | `{}` |
| `tolerations` | Tolerations for pod assignment | `[]` |

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
        ConnectionString: "Host=my-db;Port=5432;Username=user;Password=pass;Database=mydb"
      Auth:
        JwtSecret: "your-secure-jwt-secret"
```

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
      "ConnectionString": "Host=my-db;Port=5432;..."
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
    "ConnectionString": "Host=localhost;Port=5432;..."
  },
  "Logger": {
    "MinimumLogLevel": "Information"
  },
  "Auth": {
    "Enabled": true,
    "JwtSecret": "your-secret-here",
    "JwtExpireSec": 86400,
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
