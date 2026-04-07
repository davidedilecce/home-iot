# ===== Stage 1: Build =====
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /workspace

# Gradle version catalog
COPY gradle/libs.versions.toml gradle/libs.versions.toml

# Common library
COPY home-iot-common/gradle/wrapper home-iot-common/gradle/wrapper
COPY home-iot-common/gradlew home-iot-common/gradlew
COPY home-iot-common/build.gradle.kts home-iot-common/build.gradle.kts
COPY home-iot-common/settings.gradle.kts home-iot-common/settings.gradle.kts
COPY home-iot-common/src home-iot-common/src

# Service module
COPY home-iot-ms-telemetry-writer/gradle/wrapper home-iot-ms-telemetry-writer/gradle/wrapper
COPY home-iot-ms-telemetry-writer/gradlew home-iot-ms-telemetry-writer/gradlew
COPY home-iot-ms-telemetry-writer/build.gradle.kts home-iot-ms-telemetry-writer/build.gradle.kts
COPY home-iot-ms-telemetry-writer/settings.gradle.kts home-iot-ms-telemetry-writer/settings.gradle.kts
COPY home-iot-ms-telemetry-writer/src home-iot-ms-telemetry-writer/src

# Build
WORKDIR /workspace/home-iot-ms-telemetry-writer
RUN chmod +x gradlew && ./gradlew bootJar --no-daemon -x test

# ===== Stage 2: Runtime =====
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

COPY --from=build /workspace/home-iot-ms-telemetry-writer/build/libs/*.jar app.jar

# No web server (web-application-type: none).
# EXPOSE 8082 = Spring Boot Actuator management port only (health check)
EXPOSE 8082
ENV JAVA_OPTS=""
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]

