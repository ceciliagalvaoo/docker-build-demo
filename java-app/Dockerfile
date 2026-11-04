# syntax=docker/dockerfile:1

############ Stage 1: build ############
FROM maven:3-eclipse-temurin-21 AS build
WORKDIR /build

# 1. dependency definition only
COPY pom.xml .
RUN mvn -B -ntp dependency:go-offline

# 2. source code
COPY src ./src
RUN mvn -B -ntp package -DskipTests

############ Stage 2: runtime ############
FROM eclipse-temurin:21-jre AS final
WORKDIR /app

RUN groupadd --system app && useradd --system --gid app --no-create-home app

COPY --from=build /build/target/*.jar app.jar

USER app
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
