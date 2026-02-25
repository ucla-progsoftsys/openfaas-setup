# ── Stage 1: build fat jar ────────────────────────────────────────────────────
FROM maven:3.9-eclipse-temurin-11 AS builder

WORKDIR /build
COPY pom.xml .
# Pre-fetch deps in a cacheable layer
RUN mvn dependency:go-offline -q
COPY src ./src
RUN mvn package -q -DskipTests

# ── Stage 2: minimal runtime ──────────────────────────────────────────────────
FROM eclipse-temurin:11-jre-alpine

# redis-cli for wrapper pull/push; jq for JSON manipulation in entrypoint.sh
RUN apk add --no-cache bash redis jq

WORKDIR /app
COPY --from=builder /build/target/function.jar .
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

# Profile dir; normally backed by emptyDir in Phase 2
RUN mkdir -p /profiles

EXPOSE 8080
ENTRYPOINT ["/entrypoint.sh"]
