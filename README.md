# KUBEADM CLUSTER INSTALATION
## SACHIN
#### SACHIN

```bash
#pod is in crashloopbackoff state
kubectl get pods -n test-uat
kubectl get logs <pod name> -n test-uat | grep "running"
```

`kubectl get nodes
kubectl get top nodes`

> [!important]
> Exit code 137 usually indicates OOMKilled.


---

#### kubelet installation
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
```

```bash
kubectl get pods
kubectl describe pod nginx
```

```json
{
  "status": "Running"
}
```

| Exit Code | Meaning                        |
| --------- | ------------------------------ |
| 0         | Process completed successfully |
| 1         | Application error              |
| 126       | Permission denied              |
| 127       | Command not found              |
| 137       | OOMKilled                      |
| 139       | Segmentation Fault             |


## 📌 Symptoms

## 🔍 Diagnosis

## 🛠 Resolution

## ✅ Verification

## 🚀 Prevention

---

- [ ] Check pod status
- [ ] Describe the pod
- [ ] View logs
- [ ] Check events

<details>

<summary>View Complete Command Output</summary>

```bash
kubectl describe pod nginx
...

![CrashLoopBackOff](images/crashloopbackoff.png)

> Kubernetes is restarting your container because it keeps exiting unexpectedly.

# Issue Name

Overview

Symptoms

Quick Diagnosis

First Response

Diagnosis

Common Causes

Resolution

Verification

Prevention

References

### Check Pod Status



```bash
kubectl get pods
```

```bash
# Check pod status
kubectl get pod <pod-name> -n <namespace>

# Check container exit code and restart count
kubectl describe pod <pod-name> -n <namespace> | grep -A 10 "State\|Last State"

# Get logs from the crashing container
kubectl logs <pod-name> -n <namespace>

# If the current container already crashed, get previous logs
kubectl logs <pod-name> -n <namespace> --previous

# check the logs
kubectl logs <podname> -n demo
```
