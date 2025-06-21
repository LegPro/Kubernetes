## README: Kubernetes Minikube Setup on AWS EC2 (Amazon Linux 2)

This document provides step-by-step instructions to set up **Kubernetes Minikube** on an Amazon EC2 instance using **Amazon Linux 2**, along with a verification script.

---

### Prerequisites

- AWS EC2 instance with the following:
  - **Type:** `t2.medium` (2 vCPU, 4 GB RAM)
  - **OS:** Amazon Linux 2
  - **Ports Open:** 22 (SSH)

---

### Setup Instructions

#### 1. Launch EC2 Instance

- Launch a `t2.medium` EC2 instance using Amazon Linux 2 AMI.

#### 2. SSH into EC2

```bash
ssh -i your-key.pem ec2-user@<public-ip>
```

#### 3. Switch to Root

```bash
sudo -i
```

#### 4. Create a User: `velocity`

```bash
useradd velocity
passwd velocity   # Set password interactively
```

#### 5. Grant Root Access to Velocity User

```bash
visudo
```

> Add this line at the end:

```bash
velocity ALL=(ALL)   ALL
```

#### 6. Logout and Login Again

```bash
exit
# Reconnect and switch to root again
ssh -i your-key.pem ec2-user@<public-ip>
sudo -i
```

#### 7. Install Docker

```bash
yum install docker -y
systemctl start docker
```

#### 8. Add Velocity User to Docker Group

```bash
gpasswd -a velocity docker
```

#### 9. Install `kubectl`

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

#### 10. Install Minikube

```bash
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64
```

#### 11. Start Minikube Cluster

```bash
minikube start --driver=docker
```

---

### Verify Installations

- **Kubectl Version:**

  ```bash
  kubectl version
  ```

- **Minikube Status:**

  ```bash
  minikube status
  ```

---

## Automation Script (Shell)

Save this as `install_minikube.sh` and run with root privilege on the EC2 machine:

```bash
#!/bin/bash

echo "Switching to root user..."
sudo -i

echo "Creating user 'velocity'..."
useradd velocity
echo "velocity" | passwd --stdin velocity

echo "Granting sudo access to 'velocity'..."
echo "velocity ALL=(ALL)   ALL" >> /etc/sudoers

echo "User 'velocity' setup complete."

echo "Installing Docker..."
yum install -y docker
systemctl start docker
echo "Docker installed and started."

echo "Adding 'velocity' to docker group..."
gpasswd -a velocity docker

echo "Installing kubectl..."
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
echo "kubectl installed."

echo "Installing Minikube..."
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
install minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64
echo "Minikube installed."

echo "Starting Minikube cluster..."
minikube start --driver=docker

echo "Verifying installations..."
kubectl version
minikube status

echo "Setup complete!"
```

> **Make it executable**: `chmod +x install_minikube.sh`\
> **Run it**: `sudo ./install_minikube.sh`

