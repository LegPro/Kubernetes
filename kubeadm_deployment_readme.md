# Spring Boot Project Deployment on Kubernetes using Kubeadm

This guide explains the complete steps to build and deploy a Java 17 Spring Boot application on a Kubernetes cluster initialized with `kubeadm`. It covers prerequisites, Maven setup, Docker image creation, Kubernetes manifest creation, and deployment steps.

---

## Prerequisites

- Root access on EC2 instances (Amazon Linux or RHEL-based)
- EC2 instances with a Kubernetes cluster setup using `kubeadm`
- Java 17 installed
- Git installed
- Docker installed and running
- Kubernetes `kubectl` installed and configured
- Maven not yet installed (covered below)

---

## Step-by-Step Deployment

### 1. Install Maven
```bash
mkdir -p /mnt/build-tools
cd /mnt
wget https://dlcdn.apache.org/maven/maven-3/3.9.10/binaries/apache-maven-3.9.10-bin.zip
unzip apache-maven-3.9.10-bin.zip
rm -rf apache-maven-3.9.10-bin.zip
```

Update environment variables:
```bash
vi ~/.bash_profile
```
Add the following:
```bash
export MAVEN_HOME=/mnt/apache-maven-3.9.10
export PATH=$MAVEN_HOME/bin:$PATH
```
Then:
```bash
source ~/.bash_profile
```

---

### 2. Clone Git Repository and Build Project
```bash
cd /mnt
git clone https://github.com/Shantanumajan6/SprintBootService-1.git
cd SprintBootService-1
mvn clean package
```

---

### 3. Prepare for Docker Image Build
```bash
mkdir /deploy
cp target/SpringBootExecutableJarFileDemo-0.0.1-SNAPSHOT.jar /deploy
cd /deploy
```

Create `Dockerfile`:
```Dockerfile
FROM openjdk:17
WORKDIR /
COPY SpringBootExecutableJarFileDemo-0.0.1-SNAPSHOT.jar SpringBootExecutableJarFileDemo-0.0.1-SNAPSHOT.jar
EXPOSE 8080
CMD java -jar SpringBootExecutableJarFileDemo-0.0.1-SNAPSHOT.jar
```

Build and push Docker image:
```bash
docker build -t legpro/spring:1.0 .
docker images
docker run -dp 8080:8080 --name demo legpro/spring:1.0
# Test at http://<EC2_PUBLIC_IP>:8080/getData
docker login
docker push legpro/spring:1.0
```

---

### 4. Create Kubernetes Deployment and Service YAML

Create `deploy.yaml`:
```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: veldeploy
spec:
  replicas: 1
  selector:
    matchLabels:
      name: velpod
  template:
    metadata:
      name: velpod
      labels:
        name: velpod
    spec:
      containers:
        - name: vel-1
          image: legpro/spring:1.0
          ports:
            - containerPort: 8080
---
kind: Service
apiVersion: v1
metadata:
  name: velservice
spec:
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    name: velpod
  type: NodePort
```

Apply deployment:
```bash
kubectl apply -f deploy.yaml
kubectl get svc
```

Access service at:
```
http://<WORKER_NODE_PUBLIC_IP>:<NODE_PORT>/getData
```

---

##  Build Script (build.sh)
```bash
#!/bin/bash
set -e
echo ">>> Building Project"
git clone https://github.com/Shantanumajan6/SprintBootService-1.git /mnt/SprintBootService-1
cd /mnt/SprintBootService-1
mvn clean package
mkdir -p /deploy
cp target/SpringBootExecutableJarFileDemo-0.0.1-SNAPSHOT.jar /deploy
cd /deploy
echo ">>> Building Docker Image"
docker build -t legpro/spring:1.0 .
docker login
docker push legpro/spring:1.0
```

---

##  Deploy Script (deploy.sh)
```bash
#!/bin/bash
set -e
echo ">>> Applying Kubernetes Deployment"
kubectl apply -f /deploy/deploy.yaml
echo ">>> Kubernetes Service Info:"
kubectl get svc
```

---

## Notes
- Ensure all Kubernetes nodes can pull from the Docker registry.
- Open required `NodePort` in the AWS EC2 security group.
- You can also run `kubectl describe svc velservice` to get full endpoint info.

---

Deployment complete!

