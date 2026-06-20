# Student Registration Application Deployment on AWS EKS

## Project Overview

This project demonstrates the complete deployment of a full-stack Student Registration Application on Amazon EKS using Kubernetes.

### Technology Stack

* React (Frontend)
* Spring Boot (Backend)
* MariaDB (Database)
* Docker
* Kubernetes
* AWS EKS
* AWS EBS CSI Driver
* NGINX Ingress Controller
* Prometheus
* Grafana
* Promtail
* Horizontal Pod Autoscaler (HPA)

---

# Architecture

                                ┌───────────────┐
                                │     User      │
                                └───────┬───────┘
                                        │
                                        ▼
                        ┌─────────────────────────┐
                        │ AWS Load Balancer (ALB) │
                        └───────────┬─────────────┘
                                    │
                                    ▼
                    ┌─────────────────────────────┐
                    │ NGINX Ingress Controller    │
                    └───────────┬─────────────────┘
                                │
                ┌───────────────┴───────────────┐
                │                               │
                ▼                               ▼
      ┌─────────────────┐            ┌─────────────────┐
      │ Frontend Service│            │ Backend Service │
      └────────┬────────┘            └────────┬────────┘
               │                              │
               ▼                              ▼
      ┌─────────────────┐            ┌─────────────────┐
      │ React Application│           │ Spring Boot API │
      └─────────────────┘            └────────┬────────┘
                                               │
                                               ▼
                                     ┌─────────────────┐
                                     │ MariaDB         │
                                     │ StatefulSet     │
                                     └────────┬────────┘
                                              │
                                              ▼
                                     ┌─────────────────┐
                                     │ AWS EBS Volume  │
                                     └─────────────────┘
Project Structure
student-registration-app/
│
├── README.md
│
├── backend
│   ├── Dockerfile
│   ├── README.md
│   ├── mvnw
│   ├── mvnw.cmd
│   ├── pom.xml
│   ├── src
│   └── yaml
│
├── frontend
│   ├── Dockerfile
│   ├── README.md
│   ├── package.json
│   ├── package-lock.json
│   ├── src
│   ├── public
│   ├── yaml
│   └── vite.config.js
│
└── yaml
    ├── ebs-sc.yml
    ├── pvc.yml
    ├── secret.yml
    ├── sts.yml
    └── svc.yml
```text
User
 ↓
AWS Load Balancer
 ↓
NGINX Ingress Controller
 ↓
Frontend Service
 ↓
Frontend Pods
 ↓
/api Requests
 ↓
Backend Service
 ↓
Backend Pods
 ↓
MariaDB Service
 ↓
MariaDB StatefulSet
 ↓
AWS EBS Volume
```

Monitoring Flow:

```text
Spring Boot Actuator
        ↓
/actuator/prometheus
        ↓
ServiceMonitor
        ↓
Prometheus
        ↓
Grafana Dashboard
```

---

# Phase 1 - Create EKS Cluster

## Create Cluster Using AWS Console

1. Open AWS Console
2. Navigate to EKS
3. Click Create Cluster
4. Cluster Name:

```text
student-cluster
```

5. Select Kubernetes Version
6. Create IAM Role
7. Create Cluster

Wait until cluster status becomes:

```text
ACTIVE
```

---

# Phase 2 - Create Node Group

Create a Managed Node Group.

### Required IAM Policies

Attach:

```text
AmazonEKSWorkerNodePolicy
AmazonEC2ContainerRegistryReadOnly
AmazonEKS_CNI_Policy
AmazonEKSComputePolicy
```

### Node Group Configuration

```text
Instance Type : t3.medium
Desired Size  : 2
Min Size      : 2
Max Size      : 5
```

Create Node Group and wait for nodes to become Ready.

Verify:

```bash
kubectl get nodes
```

---

# Phase 3 - Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s \
https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

chmod +x kubectl

sudo mv kubectl /usr/local/bin/
```

Verify:

```bash
kubectl version --client
```

---

# Phase 4 - Install eksctl

```bash
curl --silent --location \
https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz \
| tar xz -C /tmp

sudo mv /tmp/eksctl /usr/local/bin

eksctl version
```

---

# Phase 5 - Configure kubeconfig

```bash
aws eks update-kubeconfig \
--region ap-south-1 \
--name student-cluster
```

Verify:

```bash
kubectl get nodes
```

---

# Phase 6 - Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Verify:

```bash
helm version
```

---

# Phase 7 - Install AWS EBS CSI Driver

## Add Repository

```bash
helm repo add aws-ebs-csi-driver \
https://kubernetes-sigs.github.io/aws-ebs-csi-driver

helm repo update
```

## Associate OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider \
--cluster monitor-cluster \
--approve
```

## Create IAM Service Account

```bash
eksctl create iamserviceaccount \
--name ebs-csi-controller-sa \
--namespace kube-system \
--cluster student-cluster \
--role-name AmazonEKS_EBS_CSI_DriverRole \
--attach-policy-arn \
arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
--approve
```

## Install EBS CSI Driver

```bash
helm upgrade --install aws-ebs-csi-driver \
aws-ebs-csi-driver/aws-ebs-csi-driver \
--namespace kube-system \
--set controller.serviceAccount.create=false \
--set controller.serviceAccount.name=ebs-csi-controller-sa
```

---

## Verify Installation

Check controller pods:

```bash
kubectl get pods -n kube-system | grep ebs
```

Check CSI Driver:

```bash
kubectl get csidrivers
```

Expected:

```bash
ebs.csi.aws.com
```

---


# Phase 8 - Install NGINX Ingress Controller

Add repository:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

helm repo update
```

Install:

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
--namespace ingress-nginx \
--create-namespace
```

Verify:

```bash
kubectl get pods -n ingress-nginx
```

Get LoadBalancer DNS:

```bash
kubectl get svc -n ingress-nginx
```

Save the LoadBalancer DNS because it will be used inside the Ingress resource.

---

# Phase 9 - Install Prometheus and Grafana

Add repository:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm repo update
```

Install:

```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
-n monitoring \
--create-namespace
```

Verify:

```bash
kubectl get pods -n monitoring
```

Expected:

```bash
Grafana
Prometheus
AlertManager
Kube-State-Metrics
Prometheus Operator
```

---

# Phase 10 - Database Deployment

Database resources are located in:

```text
yaml/
├── ebs-sc.yml
├── pvc.yml
├── secret.yml
├── sts.yml
└── svc.yml
```

Deploy in sequence:
---

## StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
parameters:
  type: gp2
  fsType: ext4
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

## Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
data:
  PASSWORD: cmVkaGF0
  DB_NAME: c3R1ZGVudF9kYg==
```

## PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: student-db-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 5Gi
```

## StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: student-db-sts
spec:
  replicas: 1
  selector:
    matchLabels:
      app: student-app-db
  template:
    metadata:
      labels:
        app: student-app-db
    spec:
      containers:
        - name: student-db-container
          image: mariadb:latest
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: my-secret
                  key: PASSWORD
            - name: MYSQL_DATABASE
              valueFrom:
                secretKeyRef:
                  name: my-secret
                  key: DB_NAME
          volumeMounts:
            - name: student-db-vol
              mountPath: /var/lib/mysql
      volumes:
        - name: student-db-vol
          persistentVolumeClaim:
            claimName: student-db-pvc
```

## Database Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: student-db-svc
spec:
  selector:
    app: student-app-db
  ports:
    - port: 3306
      targetPort: 3306
  type: ClusterIP
```

Deploy:

```bash
kubectl apply -f ebs-sc.yml
kubectl apply -f secret.yml
kubectl apply -f pvc.yml
kubectl apply -f sts.yml
kubectl apply -f svc.yml
```

## Verify

Check PVC:

```bash
kubectl get pvc
```

Expected:

```bash
STATUS: Bound
```

Check StatefulSet:

```bash
kubectl get sts
```

Check Pod:

```bash
kubectl get pods
```

Check Service:

```bash
kubectl get svc
```

---

# Phase 11 - Get Database Cluster IP

```bash
kubectl get svc student-db-svc
```

Example:

```bash
NAME             TYPE        CLUSTER-IP
student-db-svc   ClusterIP   10.100.241.147
```

Use this ClusterIP inside Spring Boot datasource configuration.

---

# Next Steps

After infrastructure setup is complete:

1. Build Backend Docker Image
2. Push Backend Image to DockerHub
3. Deploy Backend
4. Configure ServiceMonitor
5. Build Frontend Image
6. Deploy Frontend
7. Configure Ingress
8. Create Grafana Dashboards
9. Monitor Application Metrics

Refer to:

```text
backend/README.md
frontend/README.md
```
for further application deployment.

