# Kubernetes Installation using kubeadm on AWS EC2

This guide walks you through setting up a Kubernetes cluster using `kubeadm` version `v1.32` on AWS EC2 Linux (Amazon Linux 2 / CentOS) machines. It includes scripts for both **master** and **worker** node setup.

---

## 1. EC2 Requirements

- **Instance Type**: t2.medium (2 vCPU, 4 GB RAM)
- **OS**: Amazon Linux 2 / CentOS 7+
- **Nodes**: 2 (1 Master, 1 Worker)
- **Access**: SSH to each node

---

## 2. Prerequisites for Each Node

Ensure the following system requirements:

- 2 GB+ RAM
- 2 vCPUs
- Unique hostname, MAC address, and product\_uuid
- Full network connectivity between nodes
- Required ports open (TCP: 6443, 2379-2380, 10250-10252, etc.)

---

## 3. Common Setup (Run on both Master and Worker)

```bash
# Install Docker
sudo yum install docker -y
sudo systemctl start docker
sudo systemctl enable docker

# Disable SELinux
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# Add Kubernetes repo
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

# Install Kubernetes components
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```

---

## 4. Master Node Setup

```bash
# Initialize the cluster
sudo kubeadm init

# Setup kubeconfig for kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
export KUBECONFIG=/etc/kubernetes/admin.conf
```

Copy the `kubeadm join` command from output and run it on the worker node.

---

## 5. Worker Node Setup

Run the `kubeadm join` command received from the master node.

---

## 6. Networking - Calico Setup (Master Node)

Use the provided `calico.yaml` file or download a fresh one from the official Calico repo.

```bash
kubectl apply -f calico.yaml
```

---

## 7. Verification (Master Node)

```bash
kubectl get nodes
```

Ensure both `master` and `worker` nodes show `Ready` status.

You may also label the worker node:

```bash
kubectl label node <worker-node-name> node-role.kubernetes.io/worker=worker
```

---

## Shell Script: Master Node Setup

```bash
#!/bin/bash
set -e

echo "> Installing Docker..."
sudo yum install docker -y
sudo systemctl start docker
sudo systemctl enable docker

echo "> Disabling SELinux..."
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

echo "> Adding Kubernetes repo..."
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

echo "> Installing kubeadm, kubelet, and kubectl..."
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet

echo "> Initializing master..."
sudo kubeadm init

echo "> Configuring kubectl access..."
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
export KUBECONFIG=/etc/kubernetes/admin.conf

echo "> Applying Calico network plugin..."
kubectl apply -f calico.yaml

echo "> Master setup complete. Save the join command from above logs for the worker node."
```

---

## Shell Script: Worker Node Setup

```bash
#!/bin/bash
set -e

echo "> Installing Docker..."
sudo yum install docker -y
sudo systemctl start docker
sudo systemctl enable docker

echo "> Disabling SELinux..."
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

echo "> Adding Kubernetes repo..."
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

echo "> Installing kubeadm, kubelet, and kubectl..."
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet

echo "> Paste your kubeadm join command below to join the master node:"
echo "Example: sudo kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>"
```

---

## Notes:

- Ensure firewall/ports are open (e.g., 6443, 10250).
- You may disable `swap` if required by `kubeadm`.
- Monitor logs using `journalctl -xeu kubelet` if issues arise.
---

## Enabling All Traffic on EC2 Security Group

To ensure full connectivity between the Kubernetes nodes (master and worker), update the **EC2 Security Group**:

### Steps to Allow All Traffic:
1. Go to your AWS EC2 Dashboard.
2. In the left pane, click **"Security Groups"**.
3. Select the security group associated with both your master and worker EC2 instances.
4. Click the **Inbound rules** tab → **Edit inbound rules**.
5. Click **Add Rule**:
   - **Type**: All traffic
   - **Protocol**: All
   - **Port Range**: All
   - **Source**: Custom → **Your Security Group ID** (to allow internal traffic)
6. Click **Save rules**.

You may optionally:
- Add an additional rule to allow **SSH (port 22)** from **My IP** to maintain admin access.
- Restrict public access and use a bastion or VPN for production environments.

---
