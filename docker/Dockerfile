# ──────────────────────────────────────────────────────────────
# Stage 1 – dependency cache
# Only re-runs when pom.xml changes; Maven deps are cached.
# ──────────────────────────────────────────────────────────────
FROM eclipse-temurin:25-jdk-alpine AS deps

WORKDIR /build

COPY mvnw .
COPY .mvn .mvn
COPY pom.xml .

# Strip Windows CRLF line endings from mvnw (safe no-op if already LF)
RUN sed -i 's/\r$//' mvnw && \
    chmod +x mvnw && \
    ./mvnw dependency:go-offline -B

# ──────────────────────────────────────────────────────────────
# Stage 2 – compile and package
# Only re-runs when source changes.
# ──────────────────────────────────────────────────────────────
FROM deps AS build

COPY src ./src

RUN ./mvnw clean package -DskipTests -B

# ──────────────────────────────────────────────────────────────
# Stage 3 – extract Spring Boot layers
# Splits the fat JAR into 4 independent Docker layers so that
# library changes don't invalidate the application-code layer.
# ──────────────────────────────────────────────────────────────
FROM eclipse-temurin:25-jdk-alpine AS layers

WORKDIR /build

COPY --from=build /build/target/*.jar app.jar

RUN java -Djarmode=tools -jar app.jar extract \
      --layers \
      --launcher \
      --destination extracted

# ──────────────────────────────────────────────────────────────
# Stage 4 – minimal runtime image (JRE only, no JDK)
# ──────────────────────────────────────────────────────────────
FROM eclipse-temurin:25-jre-alpine AS runtime

RUN apk update && apk upgrade --no-cache

RUN addgroup -S appgroup && \
    adduser -S appuser -G appgroup

WORKDIR /app

# Most-stable layers first → best Docker cache hit rate
COPY --from=layers /build/extracted/dependencies/          ./
COPY --from=layers /build/extracted/spring-boot-loader/    ./
COPY --from=layers /build/extracted/snapshot-dependencies/ ./
COPY --from=layers /build/extracted/application/           ./

RUN chown -R appuser:appgroup /app

USER appuser

EXPOSE 8080

ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
