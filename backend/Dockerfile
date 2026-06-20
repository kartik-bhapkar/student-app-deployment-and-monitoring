FROM maven:3.8.3-openjdk-17 AS builder

WORKDIR /app

COPY . .

RUN mvn clean package -DskipTests

FROM eclipse-temurin:17-jre

WORKDIR /app

COPY --from=builder \
/app/target/student-registration-backend-0.0.1-SNAPSHOT.jar \
app.jar

EXPOSE 8080

ENTRYPOINT ["java","-jar","app.jar"]
