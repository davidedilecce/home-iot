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

# Service module
COPY home-iot-auth-server/gradle/wrapper              home-iot-auth-server/gradle/wrapper
COPY home-iot-auth-server/gradlew                     home-iot-auth-server/gradlew
COPY home-iot-auth-server/build.gradle.kts            home-iot-auth-server/build.gradle.kts
COPY home-iot-auth-server/settings.gradle.kts         home-iot-auth-server/settings.gradle.kts
COPY home-iot-auth-server/src                         home-iot-auth-server/src

# Build
WORKDIR /workspace/home-iot-auth-server
RUN chmod +x gradlew && ./gradlew bootJar --no-daemon -x test

# ===== Stage 2: Runtime =====
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

COPY --from=build /workspace/home-iot-auth-server/build/libs/*.jar app.jar

EXPOSE 8080
ENV JAVA_OPTS=""
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]

