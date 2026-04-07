# ===== Stage 1: Build =====
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /workspace
COPY gradle/libs.versions.toml gradle/libs.versions.toml
COPY home-iot-common/gradle/wrapper home-iot-common/gradle/wrapper
COPY home-iot-common/gradlew home-iot-common/gradlew
COPY home-iot-common/build.gradle.kts home-iot-common/build.gradle.kts
COPY home-iot-common/settings.gradle.kts home-iot-common/settings.gradle.kts
COPY home-iot-common/src home-iot-common/src
COPY home-iot-ms-events/gradle/wrapper home-iot-ms-events/gradle/wrapper
COPY home-iot-ms-events/gradlew home-iot-ms-events/gradlew
COPY home-iot-ms-events/build.gradle.kts home-iot-ms-events/build.gradle.kts
COPY home-iot-ms-events/settings.gradle.kts home-iot-ms-events/settings.gradle.kts
COPY home-iot-ms-events/src home-iot-ms-events/src
WORKDIR /workspace/home-iot-ms-events
RUN chmod +x gradlew && ./gradlew bootJar --no-daemon -x test
# ===== Stage 2: Runtime =====
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=build /workspace/home-iot-ms-events/build/libs/*.jar app.jar
EXPOSE 8080
ENV JAVA_OPTS=""
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
