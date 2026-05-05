# ============================================
# Stage 1: Maven Build
# ============================================
FROM maven:3.9-eclipse-temurin-21-alpine AS builder
WORKDIR /app

# 의존성 캐싱을 위해 pom.xml 먼저 복사
COPY pom.xml .
RUN mvn dependency:go-offline -B

# 소스 복사 후 빌드
COPY src ./src
RUN mvn package -DskipTests -B

# ============================================
# Stage 2: Runtime
# ============================================
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

COPY --from=builder /app/target/*.jar app.jar

ENV TZ=Asia/Seoul
EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar", "--spring.profiles.active=prod"]
