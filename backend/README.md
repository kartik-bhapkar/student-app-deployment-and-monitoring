backend of the student-deployment application

---

Step 1 - Add Required Dependencies

Open:

## pom.xml

```Add Spring Boot Actuator:

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```Add Prometheus Registry:

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

