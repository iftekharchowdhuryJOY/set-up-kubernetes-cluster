# Project: Bare Metal Kubernetes Cluster on KVM

**Author:** Joy (iftekharchowdhury)  
**Date:** December 2025  
**Environment:** Home Lab (HP EliteDesk running Ubuntu Server)

## 1. Project Overview
This project demonstrates the deployment of a production-grade Kubernetes cluster (v1.30) from scratch on virtual machines managed by KVM/Libvirt. Unlike cloud-managed solutions (EKS, GKE) or packaged installers (Minikube), this cluster is bootstrapped manually using `kubeadm` to simulate a real-world bare metal environment.

### Infrastructure Architecture
* **Host:** Ubuntu 24.04 LTS (Physical Server)
* **Hypervisor:** KVM/QEMU with Libvirt
* **Nodes:**
    * `k8-master` (Control Plane): 2 vCPU, 4GB RAM, 20GB Disk
    * `k8-worker1` (Worker): 2 vCPU, 4GB RAM, 20GB Disk
    * `k8-worker2` (Worker): 2 vCPU, 4GB RAM, 20GB Disk

---

## 2. VM Provisioning (The "Hard Way")
Due to specific issues with the Ubuntu 24.04 ISO installer on KVM console, the VMs were provisioned using a manual kernel boot method and a post-install configuration fix.

### Step 2.1: Extracting Kernel Files
*Prerequisite:* The ISO file is located at `/var/lib/libvirt/images/ubuntu-24.04.3-live-server-amd64.iso`.
We mounted the ISO and extracted `casper/vmlinuz` and `casper/initrd` to the images directory to allow direct kernel booting.

### Step 2.2: Creating the VMs
Run the following commands for **each node** (Master, Worker1, Worker2), changing the `--name` and disk path accordingly.
*Note: We use 4096MB RAM to prevent the Ubuntu installer from freezing during profile configuration.*

**Command to Install (Example for Master):**
```bash
sudo virt-install \
  --name k8-master \
  --ram 4096 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/k8-master.qcow2,size=20 \
  --os-variant ubuntu22.04 \
  --network network=default \
  --graphics none \
  --console pty,target_type=serial \
  --cdrom '/var/lib/libvirt/images/ubuntu-24.04.3-live-server-amd64.iso' \
  --boot kernel=/var/lib/libvirt/images/kernel-24.04,initrd=/var/lib/libvirt/images/initrd-24.04,kernel_args="console=ttyS0"

```

### Step 2.3: The "Boot Loop" Fix

After installation, the VM fails to reboot properly (error: `/init: can't open /dev/sr0`). We fix this by removing the installer config and importing the disk as a fresh machine.

**Fix Procedure (Run after install finishes):**

```bash
# 1. Force stop the VM
sudo virsh destroy k8-master

# 2. Delete the installer configuration
sudo virsh undefine k8-master

# 3. Re-import the disk to boot normally
sudo virt-install \
  --name k8-master \
  --ram 4096 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/k8-master.qcow2 \
  --network network=default \
  --graphics none \
  --console pty,target_type=serial \
  --os-variant ubuntu22.04 \
  --import

```

---

## 3. Network Configuration

To allow nodes to communicate by hostname, we configured local DNS resolution.

**1. Retrieve IPs from Host:**

```bash
sudo virsh net-dhcp-leases default

```

**2. Configure `/etc/hosts`:**
Added the following mapping to the Host machine and **all 3 VM nodes**:

```text
192.168.122.194  k8-master
192.168.122.101  k8-worker1
192.168.122.10   k8-worker2

```

---

## 4. Kubernetes Bootstrap (System Prep)

*The following commands were run on **ALL 3 NODES** to prepare them for Kubernetes.*

### Step 4.1: Kernel & Sysctl Config

Kubernetes requires swap disabled and bridged traffic allowed.

```bash
# Disable Swap
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Load Modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Configure Networking
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

```

### Step 4.2: Container Runtime (Containerd)

We use `containerd` as the CRI.

```bash
# Install
sudo apt-get update
sudo apt-get install -y containerd

# Generate Config
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable SystemdCgroup (Critical for stability)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

# Restart
sudo systemctl restart containerd

```

### Step 4.3: Installing Kubeadm, Kubelet, Kubectl

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# Add K8s v1.30 Repo
curl -fsSL [https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key](https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key) | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] [https://pkgs.k8s.io/core:/stable:/v1.30/deb/](https://pkgs.k8s.io/core:/stable:/v1.30/deb/) /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

```

---

## 5. Cluster Initialization

### Step 5.1: Initialize Control Plane (Master Node)

Ran on `k8-master`:

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.122.194 --node-name k8-master

```

### Step 5.2: Configure Admin Access

Ran on `k8-master`:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

```

### Step 5.3: Install CNI (Networking)

We used **Flannel** for the pod network.

```bash
kubectl apply -f [https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml](https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml)

```

### Step 5.4: Join Workers

Ran on `k8-worker1` and `k8-worker2` using the token generated by the init command:

```bash
sudo kubeadm join 192.168.122.194:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>

```

---

## 6. Verification & Testing

### Step 6.1: Check Nodes

```bash
kubectl get nodes

```

*Result:* All 3 nodes confirmed `Ready`.

### Step 6.2: Smoke Test (Nginx Deployment)

To verify the cluster can schedule workloads and route traffic:

1. **Deploy App:** `kubectl create deployment nginx-app --image=nginx --replicas=2`
2. **Expose Port:** `kubectl expose deployment nginx-app --port=80 --type=NodePort`
3. **Verify Access:** Checked via `curl` from the host machine to the Worker IP.

```bash
curl [http://192.168.122.101](http://192.168.122.101):<NodePort>
# Output: "Welcome to nginx!"

```

## Conclusion

The cluster is fully operational. This setup serves as the foundation for further DevOps learning, including CI/CD pipelines, Helm charts, and monitoring with Prometheus/Grafana.

