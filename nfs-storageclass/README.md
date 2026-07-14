# NFS-Backed Dynamic StorageClass Setup in Kubernetes (kubeadm on GCP)

## Overview

This guide explains how to configure **dynamic persistent storage** in a **kubeadm-based Kubernetes cluster** running on **Google Cloud Platform (GCP)** using:

- NFS Server
- Kubernetes NFS Subdir External Provisioner
- Dynamic StorageClass
- PersistentVolumeClaims (PVCs)

After completing this guide, Kubernetes will automatically provision PersistentVolumes (PVs) whenever a PersistentVolumeClaim (PVC) is created.

---

# Architecture

```text
                   +---------------------------+
                   |     GCP VM (NFS Server)   |
                   |---------------------------|
                   | /srv/nfs/kubedata         |
                   +------------+--------------+
                                |
                         NFS (TCP 2049)
                                |
        -------------------------------------------------
        |                 Kubernetes Cluster            |
        |                                               |
        |  +-------------------------------+            |
        |  | NFS External Provisioner      |            |
        |  +---------------+---------------+            |
        |                  |                            |
        |          StorageClass (nfs-client)            |
        |                  |                            |
        |      +-----------+-----------+                |
        |      |                       |                |
        |     PVC                    PVC               |
        |      |                       |                |
        |     Pod                     Pod              |
        +----------------------------------------------+
```

---

# Prerequisites

- Kubernetes cluster created using **kubeadm**
- Google Cloud Platform (GCP)
- Helm 3 installed
- `kubectl` configured
- Ubuntu-based Kubernetes nodes
- Dedicated Ubuntu VM for NFS Server
- All resources deployed within the same VPC

---

# 1. Configure GCP Firewall Rules

Allow the following ports from your Kubernetes subnet.

| Protocol | Port | Purpose |
|----------|------|---------|
| TCP | 2049 | NFS |
| TCP | 111 | rpcbind |
| UDP | 111 | rpcbind |

Example source subnet:

```text
10.128.0.0/20
```

---

# 2. Create the NFS Server VM

Recommended configuration:

| Parameter | Value |
|-----------|-------|
| Operating System | Ubuntu 22.04 LTS |
| Machine Type | e2-micro |
| Network | Same VPC as Kubernetes Cluster |
| Disk | Standard Persistent Disk |

Verify the internal IP address:

```bash
hostname -I
```

Example:

```text
10.128.0.10
```

---

# 3. Install the NFS Server

Update the package repository:

```bash
sudo apt update
```

Install the NFS server package:

```bash
sudo apt install -y nfs-kernel-server
```

Verify the service:

```bash
sudo systemctl status nfs-kernel-server
```

---

# 4. Create the NFS Export Directory

Create the shared directory:

```bash
sudo mkdir -p /srv/nfs/kubedata
```

Assign ownership:

```bash
sudo chown nobody:nogroup /srv/nfs/kubedata
```

Set permissions:

```bash
sudo chmod 777 /srv/nfs/kubedata
```

---

# 5. Configure NFS Exports

Edit the exports file:

```bash
sudo nano /etc/exports
```

Add the following entry:

```text
/srv/nfs/kubedata 10.128.0.0/20(rw,sync,no_subtree_check,no_root_squash)
```

Apply the configuration:

```bash
sudo exportfs -rav

sudo systemctl restart nfs-kernel-server

sudo exportfs -v
```

Expected output:

```text
/srv/nfs/kubedata
    10.128.0.0/20
```

---

# 6. Install the NFS Client on All Kubernetes Nodes

Run the following commands on **every Kubernetes node**.

Update packages:

```bash
sudo apt update
```

Install the NFS client:

```bash
sudo apt install -y nfs-common
```

Verify connectivity to the NFS server:

```bash
showmount -e 10.128.0.10
```

Expected output:

```text
Export list for 10.128.0.10:
/srv/nfs/kubedata
```

---

# 7. Create the Namespace

```bash
kubectl create namespace nfs-provisioner
```

Verify:

```bash
kubectl get namespace
```

---

# 8. Add the Helm Repository

```bash
helm repo add nfs-subdir-external-provisioner \
https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner

helm repo update
```

---

# 9. Install the NFS External Provisioner

```bash
helm install nfs-storage \
nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
--namespace nfs-provisioner \
--set nfs.server=10.128.0.10 \
--set nfs.path=/srv/nfs/kubedata \
--set storageClass.name=nfs-client \
--set storageClass.defaultClass=true
```

---

# 10. Verify the Installation

Verify the provisioner pod:

```bash
kubectl get pods -n nfs-provisioner
```

Example:

```text
NAME                                         READY   STATUS
nfs-storage-nfs-subdir-external-provisioner  1/1     Running
```

Verify the StorageClass:

```bash
kubectl get storageclass
```

Expected:

```text
NAME           PROVISIONER                                     DEFAULT
nfs-client     cluster.local/nfs-subdir-external-provisioner    Yes
```

---

# 11. Test Dynamic PVC Provisioning

Create `test-pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim

metadata:
  name: test-pvc

spec:
  accessModes:
    - ReadWriteMany

  storageClassName: nfs-client

  resources:
    requests:
      storage: 1Gi
```

Apply:

```bash
kubectl apply -f test-pvc.yaml
```

Verify:

```bash
kubectl get pvc
```

Expected:

```text
NAME       STATUS   VOLUME
test-pvc   Bound    pvc-xxxx
```

---

# 12. Test the PVC Using a Pod

Create `test-pod.yaml`

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: test-pod

spec:
  containers:
    - name: app
      image: busybox

      command:
        - sh
        - -c
        - |
          echo "hello" > /data/test.txt
          sleep 3600

      volumeMounts:
        - mountPath: /data
          name: storage

  volumes:
    - name: storage
      persistentVolumeClaim:
        claimName: test-pvc
```

Apply:

```bash
kubectl apply -f test-pod.yaml
```

Verify:

```bash
kubectl get pods
```

---

# 13. Validate Persistent Storage

Verify the data inside the pod:

```bash
kubectl exec -it test-pod -- sh
```

Read the file:

```bash
cat /data/test.txt
```

Expected output:

```text
hello
```

---

# 14. Verify Data on the NFS Server

On the NFS server:

```bash
ls -R /srv/nfs/kubedata
```

You should see a directory similar to:

```text
pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

Verify the file:

```bash
cat /srv/nfs/kubedata/<PVC_DIRECTORY>/test.txt
```

Expected output:

```text
hello
```

---

# Verification Checklist

| Task | Status |
|------|--------|
| NFS Server installed | ☐ |
| Firewall rules configured | ☐ |
| NFS export created | ☐ |
| NFS client installed on all nodes | ☐ |
| Helm repository added | ☐ |
| NFS External Provisioner installed | ☐ |
| StorageClass created | ☐ |
| PVC created successfully | ☐ |
| PVC status is Bound | ☐ |
| Pod created successfully | ☐ |
| Data persisted on NFS server | ☐ |

---

# Troubleshooting

## PVC Stuck in Pending

Check the provisioner:

```bash
kubectl get pods -n nfs-provisioner
```

Check PVC events:

```bash
kubectl describe pvc test-pvc
```

---

## NFS Mount Failure

Verify exports:

```bash
showmount -e 10.128.0.10
```

---

## Permission Issues

Update permissions:

```bash
sudo chmod 777 /srv/nfs/kubedata
```

Restart the NFS server:

```bash
sudo systemctl restart nfs-kernel-server
```

---

## Verify the StorageClass

```bash
kubectl get storageclass
```

---

# Security Best Practices

- Restrict NFS exports to trusted Kubernetes node subnets.
- Avoid using `no_root_squash` in production unless required.
- Use dedicated firewall rules to limit NFS access.
- Regularly back up NFS data.
- Monitor storage utilization on the NFS server.
- Use appropriate file permissions for shared directories.

---

# Conclusion

You have successfully configured:

- ✅ NFS Server on GCP
- ✅ NFS Subdir External Provisioner
- ✅ Dynamic Kubernetes StorageClass
- ✅ Automatic PersistentVolume provisioning
- ✅ ReadWriteMany (RWX) shared storage
- ✅ Dynamic PersistentVolumeClaim (PVC) provisioning

Applications can now request persistent storage dynamically without manually creating PersistentVolumes.
