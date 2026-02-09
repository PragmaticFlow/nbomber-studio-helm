# HTTPS/TLS Setup for NBomber Studio

This guide explains how to enable HTTPS for NBomber Studio using Kubernetes Ingress.

## Prerequisites

### 1. Install an Ingress Controller

You need an Ingress Controller installed in your cluster. Popular options:

#### NGINX Ingress Controller (Recommended)
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx
```

#### Traefik
```bash
helm repo add traefik https://traefik.github.io/charts
helm install traefik traefik/traefik
```

### 2. Certificate Options

#### Option A: Self-Signed Certificate (Development/Testing)
```bash
# Generate self-signed certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=nbomber-studio.local/O=NBomber"

# Create Kubernetes secret
kubectl create secret tls nbomber-studio-tls \
  --cert=tls.crt --key=tls.key
```

#### Option B: Let's Encrypt with cert-manager (Production)
```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.0/cert-manager.yaml

# Create ClusterIssuer for Let's Encrypt
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF
```

## Enable Ingress in Helm Chart

### Basic HTTPS with existing certificate:

Edit `values.yaml`:
```yaml
ingress:
  enabled: true
  className: "nginx"
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

### With cert-manager (automatic Let's Encrypt):

Edit `values.yaml`:
```yaml
ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
  hosts:
    - host: nbomber-studio.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: nbomber-studio-tls  # cert-manager will create this automatically
      hosts:
        - nbomber-studio.example.com
```

## Deploy/Upgrade

```bash
# Upgrade your Helm release
helm upgrade nbomber-studio ./chart

# Or install fresh
helm install nbomber-studio ./chart
```

## Verify Setup

```bash
# Check ingress
kubectl get ingress

# Check certificate (if using cert-manager)
kubectl get certificate

# Check if cert-manager created the secret
kubectl get secret nbomber-studio-tls
```

## Local Testing with Self-Signed Certificate

If using a self-signed certificate for local testing:

1. Add entry to `/etc/hosts` (Linux/Mac) or `C:\Windows\System32\drivers\etc\hosts` (Windows):
   ```
   127.0.0.1 nbomber-studio.local
   ```

2. Port-forward the ingress controller:
   ```bash
   kubectl port-forward -n default service/ingress-nginx-controller 443:443
   ```

3. Access via browser:
   ```
   https://nbomber-studio.local
   ```
   (You'll need to accept the self-signed certificate warning)

## Production Considerations

1. **DNS**: Point your domain to the Ingress Controller's LoadBalancer IP
   ```bash
   kubectl get service ingress-nginx-controller
   ```

2. **Rate Limiting**: Add NGINX annotations for rate limiting:
   ```yaml
   annotations:
     nginx.ingress.kubernetes.io/limit-rps: "10"
   ```

3. **Security Headers**: Add security headers:
   ```yaml
   annotations:
     nginx.ingress.kubernetes.io/configuration-snippet: |
       more_set_headers "X-Frame-Options: DENY";
       more_set_headers "X-Content-Type-Options: nosniff";
   ```

4. **Client Certificate Authentication** (optional):
   ```yaml
   annotations:
     nginx.ingress.kubernetes.io/auth-tls-verify-client: "on"
     nginx.ingress.kubernetes.io/auth-tls-secret: "default/ca-secret"
   ```

## Troubleshooting

### Certificate not issued by cert-manager
```bash
# Check certificate status
kubectl describe certificate nbomber-studio-tls

# Check cert-manager logs
kubectl logs -n cert-manager deploy/cert-manager
```

### Ingress not working
```bash
# Check ingress controller logs
kubectl logs -n default -l app.kubernetes.io/name=ingress-nginx

# Verify ingress configuration
kubectl describe ingress nbomber-studio
```

### 502 Bad Gateway
- Check if the service and pods are running: `kubectl get pods,svc`
- Verify service port matches ingress backend port
- Check pod logs: `kubectl logs <pod-name>`
