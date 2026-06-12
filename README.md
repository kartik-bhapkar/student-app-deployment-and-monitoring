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

## Install Driver

```bash
helm upgrade --install aws-ebs-csi-driver \
aws-ebs-csi-driver/aws-ebs-csi-driver \
-n kube-system
```

Verify:

```bash
kubectl get pods -n kube-system
```

---

# Phase 8 - Install NGINX Ingress Controller

```bash
helm repo add ingress-nginx \
https://kubernetes.github.io/ingress-nginx

helm repo update

helm install ingress-nginx \
ingress-nginx/ingress-nginx \
--namespace ingress-nginx \
--create-namespace
```

Verify:

```bash
kubectl get svc -n ingress-nginx
```

Copy the AWS LoadBalancer DNS.

---

# Phase 9 - Database Deployment

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

Verify:

```bash
kubectl get pvc
kubectl get pods
kubectl get svc
```

---

# Phase 10 - Backend Deployment

## application.properties

```properties
server.port=8080

spring.datasource.url=jdbc:mariadb://student-db-svc:3306/student_db
spring.datasource.username=root
spring.datasource.password=redhat

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true

management.endpoints.web.exposure.include=*
management.endpoint.prometheus.enabled=true
management.metrics.tags.application=student-app
```

---

## Backend Dockerfile

```dockerfile
FROM maven:3.8.3-openjdk-17 AS builder

WORKDIR /app

COPY . .

RUN mvn clean package -DskipTests

FROM eclipse-temurin:17-jre

WORKDIR /app

COPY --from=builder \
/app/target/student-registration-backend-0.0.1-SNAPSHOT.jar .

EXPOSE 8080

CMD ["java","-jar","student-registration-backend-0.0.1-SNAPSHOT.jar"]
```

---

## Build Image

```bash
docker build -t kartikbhapkar07/monitor-backend:latest .
```

## Push Image

```bash
docker push kartikbhapkar07/monitor-backend:latest
```

---

## Backend Deployment

```bash
kubectl apply -f deploy.yml
kubectl apply -f svc.yml
kubectl apply -f hpa.yml
```

Verify:

```bash
kubectl get deploy
kubectl get svc
kubectl get hpa
```

---

# Phase 11 - Frontend Deployment

## .env

```env
VITE_API_URL=/api
```

---

## Frontend Dockerfile

```dockerfile
FROM node AS builder

WORKDIR /app

COPY . .

RUN npm install

RUN npm run build

FROM nginx:alpine

COPY --from=builder /app/dist /usr/share/nginx/html

EXPOSE 80

CMD ["nginx","-g","daemon off;"]
```

---

## Build Image

```bash
docker build -t kartikbhapkar07/monitor-frontend:latest .
```

## Push Image

```bash
docker push kartikbhapkar07/monitor-frontend:latest
```

---

## Deploy Frontend

```bash
kubectl apply -f deploy.yml
kubectl apply -f svc.yml
kubectl apply -f ingress.yml
```

Verify:

```bash
kubectl get ingress
```

---

# Phase 12 - Monitoring Setup

## Install Prometheus + Grafana

```bash
helm repo add prometheus-community \
https://prometheus-community.github.io/helm-charts

helm repo update

helm install monitoring \
prometheus-community/kube-prometheus-stack \
-n monitoring \
--create-namespace
```

Verify:

```bash
kubectl get pods -n monitoring
```

---

# Phase 13 - Install Promtail

```bash
helm repo add grafana \
https://grafana.github.io/helm-charts

helm repo update

helm install promtail \
grafana/promtail \
-n monitoring
```

Verify:

```bash
kubectl get pods -n monitoring
```

---

# Phase 14 - ServiceMonitor

Apply:

```bash
kubectl apply -f servicemonitor.yml
```

Verify:

```bash
kubectl get servicemonitor -A
```

---

# Phase 15 - Verify Metrics Collection

Port Forward Prometheus:

```bash
kubectl port-forward \
svc/monitoring-kube-prometheus-prometheus \
9090:9090 \
-n monitoring
```

Open:

```text
http://localhost:9090
```

Check:

```text
Status → Targets
```

Student Application should be:

```text
UP
```

---

# Phase 16 - Access Grafana

Get Password:

```bash
kubectl get secret monitoring-grafana \
-n monitoring \
-o jsonpath="{.data.admin-password}" \
| base64 -d
```

Port Forward:

```bash
kubectl port-forward \
svc/monitoring-grafana \
3000:80 \
-n monitoring
```

Open:

```text
http://localhost:3000
```

Login:

```text
Username : admin
Password : <retrieved-password>
```

---

# Phase 17 - Create Grafana Dashboard

Create Dashboard → Add Visualization

Datasource:

```text
Prometheus
```

Useful Metrics:

### JVM Memory

```promql
jvm_memory_used_bytes
```

### CPU Usage

```promql
process_cpu_usage
```

### HTTP Requests

```promql
http_server_requests_seconds_count
```

### Active Threads

```promql
jvm_threads_live_threads
```

### Application Uptime

```promql
process_uptime_seconds
```

---

# Phase 18 - Test Autoscaling

Generate CPU Load:

```bash
kubectl exec -it <pod-name> -- sh
```

Install Stress Tool:

```bash
apk add stress-ng
```

Generate Load:

```bash
stress-ng --cpu 2 --timeout 300s
```

Watch HPA:

```bash
kubectl get hpa -w
```

Observe new pods being created.

---

# Troubleshooting

## CrashLoopBackOff

```bash
kubectl logs <pod-name>
```

```bash
kubectl describe pod <pod-name>
```

---

## ImagePullBackOff

Verify:

```bash
docker push <image-name>
```

Check:

```bash
kubectl describe pod <pod-name>
```

---

## 502 Bad Gateway

Verify:

```bash
kubectl get endpoints
```

Backend service must have endpoints.

---

## Empty Reply From Server

Verify:

```bash
kubectl get ingress
kubectl get svc -A
```

Check ingress routing.

---

## Grafana Datasource Errors

Verify:

```bash
kubectl logs -n monitoring deployment/monitoring-grafana
```

Check datasource configuration.

---

# Final Result

Application URL:

```text
http://<AWS-LOADBALANCER-DNS>
```

Monitoring Stack:

```text
Spring Boot Actuator
        ↓
ServiceMonitor
        ↓
Prometheus
        ↓
Grafana
```

Autoscaling:

```text
CPU > 50%
       ↓
HPA
       ↓
New Backend Pods
```

Persistent Storage:

```text
MariaDB
    ↓
PVC
    ↓
AWS EBS Volume
```
