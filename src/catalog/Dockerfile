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


# Build the go binary
FROM golang:1.25-alpine AS builder

WORKDIR /app



# Copies only the module definition and dependency checksum files into the container image layer, leaving all application source code (*.go) behind on the host machine
COPY go.mod go.sum ./
# Reads those two files and downloads all external third-party libraries into the container's local Go module cache ($GOPATH/pkg/mod
RUN go mod download

# Copy source code and compile statically
# CGO_ENABLED=0: Disables cgo, forcing the Go compiler to avoid linking against the host system's C runtime libraries (like glibc or musl). This produces a 100% statically linked binary that requires no shared libraries and can run inside completely empty or minimal runtime containers (like alpine or scratch).
# GOOS=linux: Tells the compiler to target the Linux operating system, ensuring the generated executable works inside the Linux container regardless of whether you are building on Windows, macOS, or Linux.
# go build: The standard Go command that compiles packages and dependencies into an executable binary.
# -ldflags="-w -s": Passes optimization flags directly to the internal linker:
# -w: Strips DWARF debug symbols from the binary.
# -s: Strips the symbol table and debugging information.
# Together, these two flags reduce the final binary size by roughly 20%–30% without changing performance or runtime behavior.
# -o /app/catalog: Specifies the output path and filename for the compiled executable.
# main.go: The entry point file containing package main and the func main() function to compile.

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-w -s" -o /app/catalog main.go

# Stage artifact for upload
RUN mkdir -p /app/dist && cp /app/catalog /app/dist/catalog

# Publish directly to Azure Artifacts using secret mount
# RUN --mount=type=secret,id=AZURE_PAT \
#     export AZURE_PAT=$(cat /run/secrets/AZURE_PAT) && \
#     echo "$AZURE_PAT" | az devops login --organization https://dev.azure.com/my-learning2026 && \
#     az artifacts universal publish \
#       --organization "https://dev.azure.com/my-learning2026" \
#       --project "DevOpsLearning" \
#       --scope project \
#       --feed "store-app" \
#       --name "catalog-service" \
#       --version "0.0.2" \
#       --description "Catalog microservice binary" \
#       --path "/app/dist"

# Stage 2: Minimal runtime image
FROM alpine:3.20

# Create the system group and user
RUN adduser -D -u 1000 appuser \
    && addgroup appuser appuser

# Set working directory and assign ownership
WORKDIR /app
RUN chown -R appuser:appuser /app

# Switch to the non-root user
USER appuser

COPY --from=builder /app/catalog /app/catalog

ENV GIN_MODE=release

ENTRYPOINT ["/app/catalog"]