## KUBEADM CLUSTER INSTALATION DOCS
---
### Objective
<small>
To set up a Kubernetes cluster using kubeadm, with containerd as the container runtime.
</small>

---
### Prerequisites

| Requirement | Details |
|-------------|---------|
| Operating System | Debian/Ubuntu |
| Privileges | Root or sudo access |
| CPU | Minimum 2 Cores |
| Memory | Minimum 2 GB RAM |
| Network | Connectivity between all nodes |
| Hostname | Unique hostname for each node |

---
### System Configuration (All Nodes)

Allows the Linux kernel to forward IPv4 packets between network interfaces, enabling communication between Kubernetes pods and nodes.

#### Enable IP Forwarding
```bash
sudo -i
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF
sysctl --system
sysctl net.ipv4.ip_forward
```
#### Disable Swap

Kubernetes requires swap to be disabled to ensure predictable memory management and proper scheduling by the kubelet.

```bash
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab
```

---
### Install Container Runtime (containerd)

Installs essential utilities required to download repositories, import GPG keys, and install containerd.
Adds Docker's official APT repository and GPG key so the latest version of containerd can be installed securely.
```bash
apt update && apt install -y ca-certificates curl
```
```bash
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
```
```bash
tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```
Installs containerd, the container runtime responsible for creating and managing containers on each Kubernetes node.

```bash
 apt update && apt install -y containerd.io
```

Generates the default containerd configuration and enables the SystemdCgroup setting, which is recommended for kubeadm-based Kubernetes clusters.

```bash
containerd config default > /etc/containerd/config.toml
# Edit file and set:
SystemdCgroup = true
systemctl restart containerd
systemctl enable containerd
```
---

### Install Kubernetes Components

Adds the official Kubernetes package repository and imports its signing key to install Kubernetes components securely.

```bash
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```
```bash
# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
```bash
# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
Installs the core Kubernetes tools:

kubeadm – Bootstraps and manages the Kubernetes cluster.
kubelet – Runs on every node and manages pods.
kubectl – Command-line tool used to interact with the cluster.

```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
---

### Initialize Control Plane (Master Node Only)

Creates the Kubernetes control plane by generating certificates, configuring the API server, and starting all required control plane components.

```bash
kubeadm init --apiserver-advertise-address=<MASTER_IP> --pod-network-cidr=10.244.0.0/16 --cri-socket unix:///var/run/containerd/containerd.sock
```
### Configure kubectl (Non-root User)

Configures kubectl for the current user, allowing interaction with the Kubernetes cluster without requiring root privileges.

```bash
exit
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
---

### Join Worker Nodes

Adds worker nodes to the Kubernetes cluster using the secure join token generated during control plane initialization.

```bash
kubeadm join <MASTER_IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

---

### Install Pod Network (CNI)

Deploys the Weave Net CNI plugin, enabling pod-to-pod communication across all nodes in the cluster.

```bash
kubectl apply -f https://reweave.azurewebsites.net/k8s/v1.33/net.yaml
```
---

### Verification

Confirms that all cluster nodes are in the Ready state and that system pods are running successfully.

```bash
kubectl get nodes
kubectl get pods -A
```
---

### Troubleshooting Tips

Provides commonly used commands to diagnose issues related to the container runtime and Kubernetes services.

```bash
systemctl status containerd
journalctl -u kubelet -f
```
