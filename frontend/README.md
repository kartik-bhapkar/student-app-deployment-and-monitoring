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

# Phase 10 - Ingress Configuration

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

# Phase 11 - Application Validation

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
