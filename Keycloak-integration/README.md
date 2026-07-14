# Keycloak Deployment and Integration on Kubernetes

## Overview

This guide explains how to deploy **Keycloak** on a Kubernetes cluster using **Helm**, configure a **PostgreSQL** database, expose it using **NGINX Ingress**, and secure it with **TLS** using Cert-Manager.

Keycloak provides centralized authentication and authorization using industry-standard protocols such as **OpenID Connect (OIDC)** and **OAuth 2.0**.

---

# Architecture

```text
                    Internet
                        │
                        ▼
                 Cloudflare DNS
                        │
                        ▼
               NGINX Ingress Controller
                        │
                        ▼
                 keycloak.example.com
                        │
                        ▼
                Keycloak Application
                        │
                        ▼
                  PostgreSQL Database
```

---

# Prerequisites

- Kubernetes Cluster
- Helm 3
- kubectl configured
- NGINX Ingress Controller installed
- Cert-Manager installed
- PostgreSQL database (internal or external)
- Domain configured in Cloudflare

---

# 1. Add the Bitnami Helm Repository

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami

helm repo update
```

---

# 2. Create Namespace

```bash
kubectl create namespace keycloak
```

---

# 3. Create `values.yaml`

```yaml
auth:
  adminUser: admin
  adminPassword: Admin@123

proxy: edge

ingress:
  enabled: true
  ingressClassName: nginx

  hostname: keycloak.example.com

  tls: true

  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-cloudflare

postgresql:
  enabled: true

  auth:
    username: keycloak
    password: Password@123
    database: keycloak
```

> **Note:** For production environments, store sensitive values such as passwords in Kubernetes Secrets instead of directly in the `values.yaml` file.

---

# 4. Install Keycloak

```bash
helm install keycloak bitnami/keycloak \
  -n keycloak \
  -f values.yaml
```

---

# 5. Verify the Deployment

Check the pods:

```bash
kubectl get pods -n keycloak
```

Example:

```text
NAME                             READY   STATUS
keycloak-0                       1/1     Running
keycloak-postgresql-0            1/1     Running
```

Check the services:

```bash
kubectl get svc -n keycloak
```

---

# 6. Verify the Ingress

```bash
kubectl get ingress -n keycloak
```

Example:

```text
NAME        HOSTS
keycloak    keycloak.example.com
```

---

# 7. Verify the TLS Certificate

```bash
kubectl get certificate -A
```

Verify the secret:

```bash
kubectl get secret -n keycloak
```

---

# 8. Access Keycloak

Open the following URL:

```text
https://keycloak.example.com
```

Login using:

| Username | Password |
|----------|----------|
| admin | Admin@123 |

---

# 9. Create a Realm

1. Log in to the Keycloak Admin Console.
2. Click **Create Realm**.
3. Enter the realm name.
4. Click **Create**.

Example:

```text
devops
```

---

# 10. Create a Client

Navigate to:

```
Clients
    └── Create Client
```

Configure:

| Field | Value |
|--------|-------|
| Client ID | demo-app |
| Client Type | OpenID Connect |
| Authentication | Enabled |

Example Redirect URI:

```text
https://app.example.com/*
```

Example Web Origin:

```text
https://app.example.com
```

Save the client.

---

# 11. Create a User

Navigate to:

```
Users
    └── Add User
```

Configure:

| Field | Value |
|--------|-------|
| Username | demo |
| Email | demo@example.com |

Save the user.

Set a password:

```
Credentials
    └── Set Password
```

Disable **Temporary Password**.

---

# 12. Retrieve the Client Secret

Navigate to:

```
Clients
    └── demo-app
          └── Credentials
```

Copy the generated **Client Secret**.

---

# 13. Verify OpenID Configuration

Open:

```text
https://keycloak.example.com/realms/devops/.well-known/openid-configuration
```

A JSON response confirms that OpenID Connect is configured correctly.

---

# 14. Useful Commands

Check pods:

```bash
kubectl get pods -n keycloak
```

Check services:

```bash
kubectl get svc -n keycloak
```

Check ingress:

```bash
kubectl get ingress -n keycloak
```

View logs:

```bash
kubectl logs -n keycloak deploy/keycloak
```

Describe the ingress:

```bash
kubectl describe ingress -n keycloak
```

---

# Troubleshooting

## Pod Not Starting

```bash
kubectl describe pod <pod-name> -n keycloak
```

---

## Ingress Not Working

```bash
kubectl describe ingress -n keycloak
```

Verify:

- DNS record
- Ingress Controller
- TLS Secret

---

## TLS Certificate Pending

```bash
kubectl describe certificate -A

kubectl describe challenge -A
```

---

## Database Connection Issues

Verify PostgreSQL:

```bash
kubectl get pods -n keycloak

kubectl logs -n keycloak keycloak-postgresql-0
```

---

# Security Best Practices

- Use Kubernetes Secrets for passwords.
- Enable HTTPS using Cert-Manager.
- Disable the default admin account after creating administrator users.
- Use strong passwords and rotate them regularly.
- Restrict access to the Keycloak Admin Console.
- Back up the PostgreSQL database regularly.
- Enable role-based access control (RBAC).

---

# Verification Checklist

| Task | Status |
|------|--------|
| Namespace created | ☐ |
| Helm repository added | ☐ |
| Keycloak installed | ☐ |
| PostgreSQL deployed | ☐ |
| Ingress created | ☐ |
| TLS certificate issued | ☐ |
| Admin login successful | ☐ |
| Realm created | ☐ |
| Client created | ☐ |
| User created | ☐ |
| OpenID configuration verified | ☐ |

---

# Conclusion

You have successfully deployed **Keycloak** on Kubernetes with:

- ✅ PostgreSQL backend
- ✅ NGINX Ingress
- ✅ TLS using Cert-Manager
- ✅ OpenID Connect (OIDC)
- ✅ OAuth 2.0 support
- ✅ Secure HTTPS access

Your Keycloak instance is now ready to integrate with applications requiring centralized authentication and authorization.
