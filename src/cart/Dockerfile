# ==============================================================================================
# DEVOPS CHEAT SHEET: DEPENDENCY CACHING & ARTIFACT TARGETS
# ==============================================================================================
# Stack             | Copy First (Manifests & Config)                                                            | Runtime Target (Final Image)
# ------------------+--------------------------------------------------------------------------------------------+-----------------------------
# Node (Yarn Berry) | package.json yarn.lock* .yarnrc.yml* .yarn*                                                | dist/ + production node_modules
# Node (npm)        | package.json package-lock.json*                                                            | dist/ + production node_modules
# Go                | go.mod go.sum                                                                              | Compiled binary (e.g. /app/server)
# Python (pip)      | requirements.txt                                                                           | /root/.local or venv
# Python (Poetry)   | pyproject.toml poetry.lock                                                                 | .venv
# Java (Maven)      | pom.xml .mvn/ - only if .mvn folder contains jvm.config and extension.xml                  | target/*.jar
# Java (Gradle)     | build.gradle* settings.gradle* gradlew gradle/                                             | build/libs/*.jar
# .NET              | *.sln *.csproj                                                                             | out/ (published binaries)
# Rust              | Cargo.toml Cargo.lock                                                                      | target/release/<binary>
# ==============================================================================================


# $env:AZURE_PAT="xx"
# docker build --secret id=AZURE_PAT,env=AZURE_PAT -t my-cart-service:latest . 
# --secret id=AZURE_PAT,env=AZURE_PAT (Build command): Tells BuildKit to expose the host machine's AZURE_PAT environment variable as a build secret with the identifier AZURE_PAT.
# --mount=type=secret,id=AZURE_PAT (Dockerfile RUN instruction): Temporarily mounts that secret into the build container during that specific step. By default, BuildKit places secrets at /run/secrets/<id>, so the PAT becomes readable at /run/secrets/AZURE_PAT
# export AZURE_PAT=$(cat /run/secrets/AZURE_PAT): Reads the mounted secret file into a temporary environment variable named AZURE_PAT for the lifetime of that single shell execution.
# mvn -s settings-docker.xml ...: Runs Maven using your custom settings file to download dependencies offline. Once the RUN layer finishes executing, the /run/secrets/ directory is unmounted, ensuring the token is never saved in intermediate or final container layers.
# The settings-docker.xml: Maven uses this configuration file to authenticate against remote repositories
# <id>store-app</id>: Matches the repository ID defined in project’s pom.xml. 
# When Maven attempts to fetch artifacts from store-app, it looks here for credentials.
#  <username>my-learning2026</username>: The user identity supplied to Azure Artifacts (often organization account or an arbitrary placeholder, depending on the feed configuration).
# <password>${env.AZURE_PAT}</password>: Maven variable interpolation. Maven reads the AZURE_PAT environment variable you exported in the RUN step and passes its value as the authentication password.

# Stage 1: Build the application JAR
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /app

# Copy dependency definitions and download dependencies first (optimizes caching)
COPY pom.xml settings-docker.xml .


# Mount the secret, export it as an env variable, and download dependencies
# || true on go-offline: Ensures the build does not halt if dependency:go-offline hits transient metadata resolution issues, while still populating Docker's cache layer with the majority of heavy dependencies.
# -B stands for --batch-mode.
# It prevents Maven from pausing to ask for user input (e.g., when a plugin asks for confirmation).
# Cleans up CI/CD and Docker logs: By default, Maven continuously redraws terminal lines to show download progress bars (e.g., 10KB/100KB...). In headless environments like Docker builds or CI/CD pipelines, this floods the console with thousands of redundant progress lines. -B disables the download animations and outputs simple, sequential log entries instead.
RUN --mount=type=secret,id=AZURE_PAT \
    export AZURE_PAT=$(cat /run/secrets/AZURE_PAT) && \
    mvn -s settings-docker.xml -f pom.xml dependency:go-offline -B

# Maven needs src/main/java present on disk before running package.
COPY src ./src


# It tells Maven to download all the project's dependencies, plugins, 
# and extensions specified in the pom.xml file ahead of time, preparing the environment to run completely "offline."
# By copying only the pom.xml and running go-offline before copying your actual Java source files,
# Docker caches all downloaded libraries into its own isolated layer
# clean: Deletes the target/ directory where previous build outputs, compiled .class files, and cached artifacts are stored. This prevents stale files or outdated code from lingering in your new build.
# package: Executes the standard Maven build lifecycle sequentially up through the packaging phase.
# deploy: Deploys the built package to the repository (local or remote, defined in pom.xml file), which is then used to resolve dependencies for other modules.
# -DskipTests: Bypasses running unit and integration tests during the build. This is standard practice in Docker builds because container build environments lack live databases or external services required by tests, and skipping them speeds up the image creation process.

RUN --mount=type=secret,id=AZURE_PAT \
    export AZURE_PAT=$(cat /run/secrets/AZURE_PAT) && \
   # mvn -s settings-docker.xml -f pom.xml clean package deploy -DskipTests
     mvn -s settings-docker.xml -f pom.xml clean package -DskipTests

# Stage 2: Run the application on a lean runtime image
FROM eclipse-temurin:21-jre-jammy

# Create the system group and user
RUN groupadd --gid 1000 appuser \
    && useradd --uid 1000 --gid 1000 -m -d /home/appuser -s /bin/bash appuser

# Set working directory and assign ownership
WORKDIR /app

# Look inside each service's src/main/resources/ directory
# If you see only application.properties (or application.yml):The service has no profile-specific configurations.
# Defining SPRING_PROFILES_ACTIVE does nothing harmful, but it is not strictly required.
# If application.yml or application-prod.yml, ENV SPRING_PROFILES_ACTIVE=prod is required.Without it, Spring Boot will ignore application-prod.yml and run on base defaults
ENV SPRING_PROFILES_ACTIVE=prod


RUN chown -R appuser:appuser /app

# Switch to the non-root user
USER appuser

# Copy only the built executable jar from the builder stage
COPY --from=builder /app/target/*.jar app.jar

# Expose the default Spring Boot port
# EXPOSE 8080

# Run the jar file
ENTRYPOINT ["java", "-jar", "app.jar"]