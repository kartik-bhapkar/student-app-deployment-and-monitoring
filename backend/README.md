backend of the student-deployment application

---

Step 1 - Add Required Dependencies

Open:

## pom.xml

1. Add Spring Boot Actuator:
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

2. Add Prometheus Registry:
```
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

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
# ----stage-1---
FROM maven:3.8.3-openjdk-17 AS builder
WORKDIR /app
COPY . .
RUN mvn clean package -DskipTests
# ---build complete artifact ready---

# ----stage-2---
FROM eclipse-temurin:17-jre
WORKDIR /app
COPY --from=builder /app/target/student-registration-backend-0.0.1-SNAPSHOT.jar .
EXPOSE 8080

CMD ["java", "-jar", "student-registration-backend-0.0.1-SNAPSHOT.jar"]
```

---

## Build Image

```bash
docker build -t kartikbhapkar07/monitor-be:latest .
```

## Push Image

```bash
docker push kartikbhapkar07/monitor-be:latest
```

---

Step 6 - Backend Deployment

deploy.yml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: student-app-deploy

spec:
  replicas: 2

  selector:
    matchLabels:
      app: student-app

  template:
    metadata:
      labels:
        app: student-app

    spec:
      containers:
        - name: student-app-container

          image: kartikbhapkar07/monitor-backend:latest

          ports:
            - containerPort: 8080

          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"

            limits:
              memory: "500Mi"
              cpu: "500m"
```

Deploy:
```
kubectl apply -f yaml/deploy.yml
```
Verify:
```
kubectl get deployments
```
# Step 7 - Backend Service

svc.yml

```
apiVersion: v1
kind: Service

metadata:
  name: student-app-svc
  labels:
    app: student-app

spec:
  selector:
    app: student-app

  ports:
    - name: http
      port: 8080
      targetPort: 8080

  type: ClusterIP
```

Important:

The port name:

name: http

is required for ServiceMonitor.

Deploy:
```
kubectl apply -f yaml/svc.yml
```
Verify:
```
kubectl get svc
```
Check endpoints:
```
kubectl get endpoints
```
Step 8 - Configure HPA

hpa.yml
```
apiVersion: autoscaling/v2

kind: HorizontalPodAutoscaler

metadata:
  name: student-app-hpa

spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: student-app-deploy

  minReplicas: 2
  maxReplicas: 5

  metrics:
    - type: Resource

      resource:
        name: cpu

        target:
          type: Utilization
          averageUtilization: 50
```
Deploy:
```
kubectl apply -f yaml/hpa.yml
```
Verify:
```
kubectl get hpa
```
# Step 9 - Configure ServiceMonitor

servicemonitor.yml
```
apiVersion: monitoring.coreos.com/v1

kind: ServiceMonitor

metadata:
  name: student-app-monitor
  namespace: monitoring

spec:
  selector:
    matchLabels:
      app: student-app

  namespaceSelector:
    any: true

  endpoints:
    - port: http
      path: /actuator/prometheus
      interval: 15s
```
Deploy:
```
kubectl apply -f yaml/servicemonitor.yml
```
Verify:
```
kubectl get servicemonitor -A
```
