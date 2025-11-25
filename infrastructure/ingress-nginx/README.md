# Nginx Ingress Controller Configuration

## Overview
This directory contains the Nginx Ingress Controller configuration with Cloudflare Origin Certificate as default TLS fallback.

## Components
- `install.yaml` - Main Nginx Ingress Controller installation
- `configmap.yaml` - Controller configuration with default TLS certificate
- `kustomization.yaml` - Kustomize manifest

## Manual Secret Setup

**IMPORTANT:** The TLS certificate secret is NOT stored in this repository for security reasons.

### Create the TLS Secret Manually

Before applying this configuration, you must manually create the `cloudflare-origin-tls` secret in the `ingress-nginx` namespace:

```bash
# Copy from existing namespace (if exists)
kubectl get secret cloudflare-origin-tls -n observability -o yaml | \
  sed 's/namespace: observability/namespace: ingress-nginx/' | \
  kubectl apply -f -
```

OR create from certificate files:

```bash
kubectl create secret tls cloudflare-origin-tls \
  --cert=/path/to/origin-cert.pem \
  --key=/path/to/origin-key.pem \
  -n ingress-nginx
```

### Verify Secret
```bash
kubectl get secret cloudflare-origin-tls -n ingress-nginx
```

### Configuration Explanation

The ConfigMap sets `default-ssl-certificate: ingress-nginx/cloudflare-origin-tls` which:
- Provides TLS fallback for direct IP access
- Eliminates fake certificate exposure
- Improves security posture for non-SNI clients

## Deployment

Apply via ArgoCD or manually:
```bash
kubectl apply -k infrastructure/ingress-nginx/
```

## Security Notes

⚠️ **NEVER commit TLS certificates or keys to version control**
- Use External Secrets Operator for automated secret management
- Use Sealed Secrets for encrypted secrets in Git
- Keep certificates in secure secret management systems (GCP Secret Manager, Vault, etc.)
