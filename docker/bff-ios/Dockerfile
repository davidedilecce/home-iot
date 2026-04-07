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
COPY home-iot-bff-ios/gradle/wrapper                  home-iot-bff-ios/gradle/wrapper
COPY home-iot-bff-ios/gradlew                         home-iot-bff-ios/gradlew
COPY home-iot-bff-ios/build.gradle.kts                home-iot-bff-ios/build.gradle.kts
COPY home-iot-bff-ios/settings.gradle.kts             home-iot-bff-ios/settings.gradle.kts
COPY home-iot-bff-ios/src                             home-iot-bff-ios/src

# Build
WORKDIR /workspace/home-iot-bff-ios
RUN chmod +x gradlew && ./gradlew bootJar --no-daemon -x test

# ===== Stage 2: Runtime =====
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

COPY --from=build /workspace/home-iot-bff-ios/build/libs/*.jar app.jar

EXPOSE 8080
ENV JAVA_OPTS=""
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]

