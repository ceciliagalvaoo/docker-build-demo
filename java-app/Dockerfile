# syntax=docker/dockerfile:1

# Global ARG: declared before the first FROM, so it can only be used in FROM lines.
ARG JAVA_VERSION=21

############ Stage 1: build ############
FROM maven:3-eclipse-temurin-${JAVA_VERSION} AS build
WORKDIR /build

# 1. dependency definition only
COPY pom.xml .
# Cache mount: the Maven local repository lives outside the image layers.
RUN --mount=type=cache,target=/root/.m2 \
    mvn -B -ntp dependency:go-offline

# 2. source code
COPY src ./src
RUN --mount=type=cache,target=/root/.m2 \
    mvn -B -ntp package -DskipTests

############ Test stage (only runs with --target test) ############
FROM build AS test
RUN --mount=type=cache,target=/root/.m2 \
    mvn -B -ntp test

############ Stage 2: runtime ############
FROM eclipse-temurin:${JAVA_VERSION}-jre AS final
WORKDIR /app

RUN groupadd --system app && useradd --system --gid app --no-create-home app

COPY --from=build /build/target/*.jar app.jar

# ARGs are scoped to a stage: APP_VERSION is declared in the stage that uses it,
# after the RUN above so that a new version does not invalidate it.
ARG APP_VERSION=0.0.0
LABEL org.opencontainers.image.version=${APP_VERSION} \
      org.opencontainers.image.source="https://github.com/ceciliagalvaoo/docker-build-demo" \
      org.opencontainers.image.description="Demo Spring Boot app" \
      org.opencontainers.image.licenses="MIT"

USER app
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
