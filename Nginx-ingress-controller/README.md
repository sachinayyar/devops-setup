# Kubernetes NGINX Ingress + Cloudflare + Cert-Manager SSL Setup

## Overview

This guide explains how to:

- Install the **NGINX Ingress Controller**
- Configure **Cloudflare DNS**
- Install **Cert-Manager**
- Generate a **Let's Encrypt Wildcard SSL Certificate**
- Configure Kubernetes **Ingress** with HTTPS
- Secure applications using **TLS**

---

# Architecture

<p align="center">
  <img src="https://github.com/sachinayyar/devops-setup/blob/main/Nginx-ingress-controller/ingress-architecture.png" alt="Kubernetes Cluster Architecture" width="800">
</p>

---

# Prerequisites

- Kubernetes Cluster
- Helm 3
- `kubectl` configured
- Cloudflare account
- Registered domain (e.g., `ayyarsachin.shop`)
- Public IP address or LoadBalancer IP

---

# 1. Install NGINX Ingress Controller

## Add the Helm Repository

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

## Install the Ingress Controller

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=NodePort
```

## Verify Installation

```bash
kubectl get pods -n ingress-nginx

kubectl get svc -n ingress-nginx
```

Expected Output:

```text
NAME                                        READY   STATUS
ingress-nginx-controller-xxxxx              1/1     Running
```

---

# 2. Configure Cloudflare DNS

## Add Your Domain

Log in to **Cloudflare** and add your domain.

Example:

```
ayyarsachin.shop
```

Update the nameservers at your domain registrar with the nameservers provided by Cloudflare.

---

## Create DNS Records

| Type | Name | Value |
|------|------|-------|
| A | @ | `<LoadBalancer-IP>` |
| A | * | `<LoadBalancer-IP>` |

Example:

| Type | Name | Value |
|------|------|-------|
| A | @ | 34.xxx.xxx.xxx |
| A | * | 34.xxx.xxx.xxx |

> **Note:** Disable the Cloudflare proxy (set to **DNS Only**) during the initial SSL certificate issuance.

---

# 3. Create a Cloudflare API Token

Navigate to:

```
Cloudflare Dashboard
    └── My Profile
          └── API Tokens
                └── Create Token
```

Use the **Edit zone DNS** template.

Configure the following permissions:

| Permission | Value |
|------------|-------|
| Zone | DNS |
| Action | Edit |
| Zone Resources | Include → Specific Zone |
| Zone Name | `ayyarsachin.shop` |

Save the generated API token securely.

---

# 4. Install Cert-Manager

## Install the CRDs

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.crds.yaml
```

## Add the Helm Repository

```bash
helm repo add jetstack https://charts.jetstack.io

helm repo update
```

## Install Cert-Manager

```bash
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace
```

## Verify Installation

```bash
kubectl get pods -n cert-manager
```

---

# 5. Create the Cloudflare API Token Secret

```bash
kubectl create secret generic cloudflare-api-token \
  --from-literal=api-token=<YOUR_CLOUDFLARE_API_TOKEN> \
  -n cert-manager
```

Verify:

```bash
kubectl get secrets -n cert-manager
```

---

# 6. Create a ClusterIssuer

Create `clusterissuer.yaml`

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer

metadata:
  name: letsencrypt-cloudflare

spec:
  acme:
    email: admin@ayyarsachin.shop
    server: https://acme-v02.api.letsencrypt.org/directory

    privateKeySecretRef:
      name: letsencrypt-cloudflare

    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: api-token
```

Apply:

```bash
kubectl apply -f clusterissuer.yaml
```

Verify:

```bash
kubectl get clusterissuer
```

---

# 7. Create a Wildcard SSL Certificate

Create `certificate.yaml`

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate

metadata:
  name: wildcard-ayyarsachin-shop
  namespace: default

spec:
  secretName: wildcard-ayyarsachin-shop-tls

  issuerRef:
    name: letsencrypt-cloudflare
    kind: ClusterIssuer

  dnsNames:
    - "*.ayyarsachin.shop"
    - "ayyarsachin.shop"
```

Apply:

```bash
kubectl apply -f certificate.yaml
```

Verify:

```bash
kubectl get certificate

kubectl describe certificate wildcard-ayyarsachin-shop
```

---

# 8. Verify the TLS Secret

```bash
kubectl get secret wildcard-ayyarsachin-shop-tls
```

The secret should contain:

- `tls.crt`
- `tls.key`

---

## Extract the Certificate

```bash
kubectl get secret wildcard-ayyarsachin-shop-tls \
  -o jsonpath="{.data.tls\.crt}" | base64 -d > cert.crt
```

## Extract the Private Key

```bash
kubectl get secret wildcard-ayyarsachin-shop-tls \
  -o jsonpath="{.data.tls\.key}" | base64 -d > cert.key
```
---

# 9. Configure Kubernetes Ingress

Create `ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress

metadata:
  name: app-ingress

  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-cloudflare
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"

spec:
  ingressClassName: nginx

  tls:
    - hosts:
        - app.ayyarsachin.shop
      secretName: wildcard-ayyarsachin-shop-tls

  rules:
    - host: app.ayyarsachin.shop

      http:
        paths:
          - path: /

            pathType: Prefix

            backend:
              service:
                name: demo

                port:
                  number: 80
```

Apply:

```bash
kubectl apply -f ingress.yaml
```

---

# 10. Verification

## Verify Ingress

```bash
kubectl get ingress
```

## Verify Certificates

```bash
kubectl get certificate
```

## Verify Pods

```bash
kubectl get pods -A
```

## Test HTTPS

```text
https://app.ayyarsachin.shop
```

A valid Let's Encrypt SSL certificate should be presented by the browser.

---

# Expected Outcome

- ✅ NGINX Ingress Controller is installed and running.
- ✅ Cloudflare DNS records point to the Kubernetes cluster.
- ✅ Cert-Manager is installed.
- ✅ ClusterIssuer is configured successfully.
- ✅ Wildcard SSL certificate is generated.
- ✅ TLS secret is created.
- ✅ HTTPS is enabled for the application.
- ✅ Traffic is securely routed to Kubernetes services.

---

# Troubleshooting

## Check Cert-Manager Logs

```bash
kubectl logs -n cert-manager deploy/cert-manager
```

## Describe the Certificate

```bash
kubectl describe certificate wildcard-ayyarsachin-shop
```

## Describe the ClusterIssuer

```bash
kubectl describe clusterissuer letsencrypt-cloudflare
```

## Check the Ingress

```bash
kubectl describe ingress app-ingress
```

---

# Security Best Practices

- Store the Cloudflare API Token as a Kubernetes Secret.
- Never commit secrets or API tokens to Git.
- Use the principle of least privilege when creating Cloudflare API tokens.
- Rotate API tokens periodically.
- Enable HTTPS redirection.
- Use wildcard certificates to simplify TLS management across multiple subdomains.

