# Phase 18 - Frontend Deployment

## Update .env

Before:

```env
VITE_API_URL=http://BACKEND_IP:8080/api
```

After:

```env
VITE_API_URL=/api
```

Reason:

Ingress will handle routing to backend services.

---
## Frontend Dockerfile

```dockerfile
FROM node:20 AS builder

WORKDIR /app

COPY . .

RUN npm install

RUN npm run build

FROM nginx:alpine

COPY --from=builder /app/dist/. /usr/share/nginx/html/

EXPOSE 80

CMD ["nginx","-g","daemon off;"]
```

---

## Build Frontend Image

```bash
docker build -t <dockerhub-username>/frontend:latest .
```

---

## Push Frontend Image

```bash
docker push <dockerhub-username>/frontend:latest
```

---

## Frontend Kubernetes Resources

text
deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: student-frontend-deploy
spec:
  replicas: 2
  selector:
    matchExpressions:
      - {key: app, operator: In, values: [student-frontend]}
  template:
    metadata:
      name: pod-01
      labels:
        app: student-frontend
    spec:
      containers:
        - name: student-frontend-container
          image: kartikbhapkar07/monitor-fe:latest
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "500Mi"
              cpu: "500m"
```

service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: student-frontend-svc
spec:
  selector:
    app: student-frontend
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  type: ClusterIP
```

Apply:

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

Verify:

```bash
kubectl get pods
kubectl get svc
```

---

# Phase 19 - Ingress Configuration

Create:


ingress.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: student-app-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: <YOUR-ALB-DNS>
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: student-frontend-svc
              port:
                number: 80

        - path: /api
          pathType: Prefix
          backend:
            service:
              name: student-app-svc
              port:
                number: 8080
```

Ingress Routes:

```text
/      -> student-frontend-svc:80
/api   -> student-app-svc:8080
```

Apply:

```bash
kubectl apply -f ingress.yaml
```

Verify:

```bash
kubectl describe ingress student-app-ingress
```

Expected:

```text
/      student-frontend-svc:80
/api   student-app-svc:8080
```

---

# Phase 20 - Application Validation

Check all resources:

```bash
kubectl get nodes
kubectl get pods
kubectl get svc
kubectl get pvc
kubectl get hpa
kubectl get ingress
```

Retrieve Load Balancer DNS:

```bash
kubectl get svc -n ingress-nginx
```

Open:

```text
http://<LOAD_BALANCER_DNS>
```

Expected:

* Frontend loads successfully
* User registration works
* Backend APIs respond successfully
* Data is stored in MariaDB
* Persistent storage attached through EBS

---

# Monitoring Verification & Dashboard Access

The monitoring stack (Prometheus + Grafana) has already been installed in the cluster.

The Spring Boot application exposes metrics through:

```text
/actuator/prometheus
```

Prometheus scrapes these metrics using the configured ServiceMonitor.

---

## Verify ServiceMonitor

Check whether the ServiceMonitor is deployed:

```bash
kubectl get servicemonitor -A
```

Expected Output:

```text
monitoring   student-app-monitor
```

---

## Verify Backend Service Endpoints

Check whether the backend service is discovering application pods:

```bash
kubectl get endpoints student-app-svc
```

Expected Output:

```text
student-app-svc   <pod-ip>:8080
```

---

## Verify Spring Boot Metrics Endpoint

Port-forward the backend service:

```bash
kubectl port-forward svc/student-app-svc 8080:8080
```

Open:

```text
http://localhost:8080/actuator/prometheus
```

Expected Output:

```text
jvm_memory_used_bytes
process_cpu_usage
system_cpu_usage
http_server_requests_seconds_count
...
```

---

# Access Prometheus

Port-forward Prometheus:

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

---

## Verify Prometheus Targets

Navigate to:

```text
Status → Targets
```

Verify that:

```text
student-app-monitor
```

appears with status:

```text
UP
```

---

# Access Grafana

Get Grafana Admin Password:

```bash
kubectl get secret monitoring-grafana \
-n monitoring \
-o jsonpath="{.data.admin-password}" | base64 -d && echo
```

Default Username:

```text
admin
```

Port-forward Grafana:

```bash
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring
```

Open:

```text
http://localhost:3000
```

Login using:

```text
Username: admin
Password: <password-from-previous-command>
```

---

# Configure Prometheus Datasource

If Prometheus datasource is not automatically available, add it manually.

Datasource URL:

```text
http://monitoring-kube-prometheus-prometheus.monitoring:9090
```

Grafana Navigation:

```text
Connections
 └── Data Sources
      └── Add Data Source
           └── Prometheus
```

Click:

```text
Save & Test
```

Expected Result:

```text
Datasource is working
```

---

# Useful Prometheus Queries

## JVM Memory Usage

```promql
jvm_memory_used_bytes
```

## CPU Usage

```promql
process_cpu_usage
```

## System CPU Usage

```promql
system_cpu_usage
```

## Total HTTP Requests

```promql
http_server_requests_seconds_count
```

## Request Rate

```promql
rate(http_server_requests_seconds_count[1m])
```

## Running Pods

```promql
count(kube_pod_status_phase{phase="Running"})
```

## HPA Current Replicas

```promql
kube_horizontalpodautoscaler_status_current_replicas
```

## Node CPU Usage

```promql
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

## Node Memory Usage

```promql
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes)
/
node_memory_MemTotal_bytes * 100
```

---

# Monitoring Flow

```text
Spring Boot Application
          │
          ▼
 /actuator/prometheus
          │
          ▼
     ServiceMonitor
          │
          ▼
      Prometheus
          │
          ▼
        Grafana
          │
          ▼
     Dashboards
```

---

# Monitoring Health Check Commands

```bash
kubectl get pods -n monitoring

kubectl get servicemonitor -A

kubectl get svc -n monitoring

kubectl get endpoints student-app-svc

kubectl logs -n monitoring deployment/monitoring-grafana

kubectl logs -n monitoring prometheus-monitoring-kube-prometheus-prometheus-0
```
# Troubleshooting

## Node Group Stuck in Creating

Cause:

```text
UnauthorizedOperation
ec2:DescribeInstances
```

Fix:

Attach:

```text
AmazonEKSComputePolicy
```

to Node IAM Role.

---

## ImagePullBackOff

Cause:

Docker image not pushed.

Fix:

```bash
docker push <image>
```

---

## CrashLoopBackOff

Cause:

Database configuration or Secret issues.

Fix:

Verify Secret references and environment variables.

---

## Frontend White Screen

Cause:

Incorrect Docker COPY command.

Incorrect:

```dockerfile
COPY --from=builder /app/dist/* /usr/share/nginx/html
```

Correct:

```dockerfile
COPY --from=builder /app/dist/. /usr/share/nginx/html/
```

---

## API Connection Refused

Cause:

Frontend using backend node IP.

Incorrect:

```env
VITE_API_URL=http://BACKEND_IP:8080/api
```

Correct:

```env
VITE_API_URL=/api
```

with Ingress routing.

---

# Final Outcome

Successfully deployed a production-style application on AWS EKS using:

* Amazon EKS
* Managed Node Groups
* Kubernetes Deployments
* StatefulSets
* Persistent Volumes
* EBS CSI Driver
* Kubernetes Secrets
* Horizontal Pod Autoscaler
* NGINX Ingress Controller
* Docker
* React
* Spring Boot
* MariaDB

This project demonstrates end-to-end Kubernetes deployment, storage management, networking, autoscaling, and troubleshooting on AWS.
