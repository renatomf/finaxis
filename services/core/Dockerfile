# syntax=docker/dockerfile:1
# Container image of the core service. Build it from the repository root:
#   just image
#
# Two stages: `build` compiles the service with the full JDK; the final stage
# copies only the packaged application onto a minimal runtime image. The
# compiler, Gradle and the sources never reach the image that runs.
#
# Base images are pinned by digest; Dependabot bumps tag and digest together.

FROM eclipse-temurin:25-jdk-noble@sha256:80f67fb3d6f76e2d817d98a4cc2a9cf05f054d4e6adc5e2144bde671d722e63f AS build
WORKDIR /workspace
COPY . .
# Tests run in CI before an image is built, so this stage only packages the
# JAR. The cache mount keeps Gradle's downloads between builds without
# storing them in any image layer.
RUN --mount=type=cache,target=/root/.gradle \
    ./gradlew --no-daemon bootJar
# Spring Boot's layertools split the JAR into directories ordered by how often
# they change: third-party libraries, Spring Boot's loader, snapshot
# libraries, then the service's own code. Each becomes one image layer below,
# so a change to the service's code rebuilds and ships only the last, small
# layer and reuses the others from the cache.
RUN java -Djarmode=tools -jar build/libs/finaxis-core.jar \
    extract --layers --destination extracted

# Distroless: the Java runtime and nothing else, no shell and no package
# manager, so an attacker who gets in finds no tools. The `nonroot` variant
# runs as user 65532, never as root.
FROM gcr.io/distroless/java25-debian13:nonroot@sha256:b0a31b9a063a6fb8132ec93da4cfd8515421ad35e40d7ac79fbebbaaae98227f
WORKDIR /app
COPY --from=build /workspace/extracted/dependencies/ ./
COPY --from=build /workspace/extracted/spring-boot-loader/ ./
COPY --from=build /workspace/extracted/snapshot-dependencies/ ./
COPY --from=build /workspace/extracted/application/ ./
USER nonroot
EXPOSE 8080
# The extracted JAR finds its libraries in lib/ through its manifest.
ENTRYPOINT ["java", "-jar", "finaxis-core.jar"]
