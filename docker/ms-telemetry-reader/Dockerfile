# ===== Stage 1: Build =====
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /workspace

# Gradle version catalog
COPY gradle/libs.versions.toml                        gradle/libs.versions.toml

# Common library
COPY home-iot-common/gradle/wrapper                   home-iot-common/gradle/wrapper
COPY home-iot-common/gradlew                          home-iot-common/gradlew
COPY home-iot-common/build.gradle.kts                 home-iot-common/build.gradle.kts
COPY home-iot-common/settings.gradle.kts              home-iot-common/settings.gradle.kts
COPY home-iot-common/src                              home-iot-common/src

# Service module — dependency layer (cached if build.gradle.kts unchanged)
COPY home-iot-ms-telemetry-reader/gradle/wrapper      home-iot-ms-telemetry-reader/gradle/wrapper
COPY home-iot-ms-telemetry-reader/gradlew             home-iot-ms-telemetry-reader/gradlew
COPY home-iot-ms-telemetry-reader/build.gradle.kts    home-iot-ms-telemetry-reader/build.gradle.kts
COPY home-iot-ms-telemetry-reader/settings.gradle.kts home-iot-ms-telemetry-reader/settings.gradle.kts

WORKDIR /workspace/home-iot-ms-telemetry-reader
RUN chmod +x gradlew && ./gradlew dependencies --no-daemon

# Service module — source & build (re-runs only when src changes)
COPY home-iot-ms-telemetry-reader/src                 /workspace/home-iot-ms-telemetry-reader/src
RUN ./gradlew bootJar --no-daemon -x test

# ===== Stage 2: Runtime =====
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

COPY --from=build /workspace/home-iot-ms-telemetry-reader/build/libs/*.jar app.jar
RUN chown appuser:appgroup app.jar

USER appuser

EXPOSE 8080
ENV JAVA_OPTS="-XX:MaxRAMPercentage=75.0 -XX:+UseG1GC"
ENTRYPOINT ["sh", "-c", "exec java $JAVA_OPTS -jar app.jar"]
