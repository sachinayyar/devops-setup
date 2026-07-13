# Jenkins Kubernetes Agent Integration

## Overview

This document describes the process of integrating Jenkins with a Kubernetes cluster using the **Jenkins Kubernetes Cloud Plugin**.

The integration enables Jenkins to dynamically provision build agents as Kubernetes pods for executing CI/CD pipelines.

## Architecture

<p align="center">
  <img src="https://github.com/sachinayyar/devops-setup/blob/main/CICD/Jenkins-agent-integration/jenkins-agent-architecture.png" alt="Kubernetes Cluster Architecture" width="800">
</p>

---

## Environment Details

| Component | Configuration |
|-----------|--------------|
| Jenkins Namespace | `jenkins` |
| Jenkins Agent Namespace | `jenkins` |
| Kubernetes Cloud Name | `kubernetes` |
| ServiceAccount | `jenkins-agent-sa` |
| Agent Pod Label | `java21` |
| Agent Container | `jnlp` |

> Jenkins controller and Jenkins agent pods are deployed in the same `jenkins` namespace.

---

# Prerequisites

## Jenkins Requirements

- Jenkins server is running and accessible.
- Jenkins user has administrative privileges.
- Jenkins Kubernetes Plugin is installed.

Plugin:
- Kubernetes Plugin

## Kubernetes Requirements

- Kubernetes cluster access.
- `kubectl` configured.
- Required RBAC permissions.
- `jenkins` namespace available.

Verify namespace:

```bash
kubectl get namespace jenkins
```

Expected:

```
NAME       STATUS
jenkins    Active
```

---

# 1. Create Kubernetes ServiceAccount

Create a ServiceAccount that Jenkins will use to communicate with Kubernetes.

File:

`jenkins-agent-serviceaccount.yaml`

```yaml
apiVersion: v1
kind: ServiceAccount

metadata:
  name: jenkins-agent-sa
  namespace: jenkins
```

Apply:

```bash
kubectl apply -f jenkins-agent-serviceaccount.yaml
```

Verify:

```bash
kubectl get serviceaccount -n jenkins
```

---

# 2. Create Kubernetes Role

The Role provides permissions required by Jenkins to manage Kubernetes agent pods.

File:

`jenkins-agent-role.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role

metadata:
  name: jenkins-agent-role
  namespace: jenkins

rules:

- apiGroups:
  - ""

  resources:
  - pods
  - pods/exec
  - pods/log
  - secrets
  - configmaps

  verbs:
  - get
  - list
  - watch
  - create
  - delete
  - patch
  - update
```

Apply:

```bash
kubectl apply -f jenkins-agent-role.yaml
```

Verify:

```bash
kubectl get role -n jenkins
```

---

# 3. Create RoleBinding

RoleBinding assigns the required permissions to the Jenkins ServiceAccount.

File:

`jenkins-agent-rolebinding.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding

metadata:
  name: jenkins-agent-rolebinding
  namespace: jenkins


subjects:

- kind: ServiceAccount
  name: jenkins-agent-sa
  namespace: jenkins


roleRef:

  kind: Role
  name: jenkins-agent-role
  apiGroup: rbac.authorization.k8s.io
```

Apply:

```bash
kubectl apply -f jenkins-agent-rolebinding.yaml
```

Verify:

```bash
kubectl get rolebinding -n jenkins
```

---

# 4. Validate RBAC Permissions

Verify Jenkins ServiceAccount permissions.

## Check Pod Creation Permission

```bash
kubectl auth can-i create pods \
--as=system:serviceaccount:jenkins:jenkins-agent-sa \
-n jenkins
```

Expected:

```
yes
```

## Check Pod Delete Permission

```bash
kubectl auth can-i delete pods \
--as=system:serviceaccount:jenkins:jenkins-agent-sa \
-n jenkins
```

Expected:

```
yes
```

---

# 5. Configure Jenkins Kubernetes Cloud

Navigate:

```
Jenkins Dashboard
    |
    +-- Manage Jenkins
          |
          +-- Clouds
                |
                +-- Configure Clouds
                      |
                      +-- Add a new cloud
                            |
                            +-- Kubernetes
```

## Kubernetes Cloud Configuration

### Cloud Name

```
kubernetes
```

### Kubernetes URL

For Jenkins running inside Kubernetes:

```
https://kubernetes.default.svc
```

### Kubernetes Namespace

```
jenkins
```

### Enable WebSocket

Enable:

```
[x] WebSocket
```

### Test Connection

Click:

```
Test Connection
```

Expected:

```
Connection successful
```

---

# 6. Configure Jenkins Pod Template

Navigate:

```
Manage Jenkins
    |
    +-- Clouds
          |
          +-- Kubernetes
                |
                +-- Pod Templates
                      |
                      +-- Add Pod Template
```

## Pod Template Configuration

| Field | Value |
|------|-------|
| Name | java-agent |
| Namespace | jenkins |
| Label | java21 |
| Usage | Use this node as much as possible |

---

# Container Template Configuration

| Parameter | Value |
|-----------|-------|
| Container Name | jnlp |
| Docker Image | docker.io/ayyarsachin/trivy-agent:1.0.0 |
| Image Pull Policy | Always pull image |
| Working Directory | `/home/jenkins/` |

Enable:

```
[x] Inject Jenkins agent in agent container

[x] Allocate pseudo-TTY
```

---

# 7. Pod Template YAML

Add the following YAML under **Raw YAML for the Pod**:

```yaml
spec:

  containers:

  - name: jnlp

    securityContext:
      privileged: true

    resources:

      requests:
        cpu: "500m"
        memory: "1Gi"
        ephemeral-storage: "10Gi"

      limits:
        cpu: "2"
        memory: "4Gi"
        ephemeral-storage: "20Gi"
```

---

# 8. Jenkins Pipeline Configuration

Example Jenkins pipeline using Kubernetes agent:

```groovy
pipeline {
    agent {
        kubernetes {
            label 'java21'
            defaultContainer 'jnlp'
        }
    }
    stages {
        stage('Agent Validation') {
            steps {
                sh '''
                echo "Jenkins Kubernetes Agent Running"
                hostname
                whoami
                '''
            }
        }
    }
}
```

---

# 9. Validate Jenkins Agent Pod Creation

Trigger the Jenkins pipeline.

Check Kubernetes pods:

```bash
kubectl get pods -n jenkins
```

Expected output:

```
NAME                         READY   STATUS
java21-agent-xxxxx           1/1     Running
```

After pipeline completion, the agent pod will be removed based on Jenkins Kubernetes plugin configuration.

---

# Troubleshooting

## 1. Kubernetes Cloud Connection Failed

Verify Kubernetes connectivity:

```bash
kubectl cluster-info
```

Check ServiceAccount permissions:

```bash
kubectl auth can-i create pods \
--as=system:serviceaccount:jenkins:jenkins-agent-sa \
-n jenkins
```

---

## 2. Agent Pod Stuck in Pending State

Check pod events:

```bash
kubectl describe pod <pod-name> -n jenkins
```

Common reasons:

- Insufficient node resources
- Image pull failure
- Incorrect image configuration
- Security policy restrictions

---

## 3. Jenkins Agent Disconnect Issue

Check agent logs:

```bash
kubectl logs <pod-name> \
-n jenkins \
-c jnlp
```

---

# Validation Checklist

| Task | Status |
|------|--------|
| Jenkins namespace verified | ☐ |
| ServiceAccount created | ☐ |
| Role created | ☐ |
| RoleBinding created | ☐ |
| RBAC permissions validated | ☐ |
| Kubernetes Cloud configured | ☐ |
| WebSocket enabled | ☐ |
| Connection test successful | ☐ |
| Pod Template configured | ☐ |
| Jenkins pipeline executed | ☐ |
| Dynamic agent pod created | ☐ |

---

# Conclusion

Jenkins is now integrated with Kubernetes and can dynamically provision build agent pods using the Kubernetes Cloud plugin.

This setup provides scalable Jenkins agents that are created on demand and automatically cleaned up after pipeline execution.
