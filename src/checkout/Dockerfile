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




# Stage 1: Build & Compile TypeScript
FROM node:20-alpine AS builder

WORKDIR /usr/src/app

# Enable Corepack to activate modern Yarn (v4)
RUN corepack enable

# Ensure Yarn v4 installs standard node_modules instead of PnP
ENV YARN_NODE_LINKER=node-modules

# Layer caching: copy lockfiles before source code
# Lock files (yarn.lock, package-lock.json, go.sum) pin down the exact resolved version and cryptographic checksum of every single dependency and sub-dependency in the project.
# Lock Files Prevent:
# Non-Deterministic Builds ("Works on my machine")
# Transitive Dependency Drift
# Security Vulnerabilities
# By copying only the lockfiles first, Docker caches the dependency installation layer. This means that if you change your source code but not your dependencies, Docker will reuse the cached layer and skip re
# installing dependencies, significantly speeding up the build process.
# Faster Builds

# .yarnrc.yml (The Rulebook):
# This is the primary configuration file for modern Yarn. It instructs Yarn how to behave—for instance, 
# telling it whether to use traditional node_modules (nodeLinker: node-modules) or Yarn's Zero-Install Plug'n'Play (PnP) system, which package registries to talk to, and network proxy rules. Without copying this file into Docker, Yarn reverts to default behavior (PnP), which breaks NestJS because it expects standard node_modules.
# The .yarnrc.yml file is not copied into the final image, but it is copied into the .yarn folder, which is a hidden folder in the root of your project. This folder is where Yarn stores its cache and plugins.
# .yarn/ (Yarn's Cache and Plugins):
# Inside the .yarn directory, there is usually a releases/ folder containing a file like yarn-4.11.0.cjs. Modern Yarn vendors the exact package manager binary directly into your project repo. When you ran corepack enable, Corepack read your project settings and looked into .yarn to run that bundled binary. If Docker doesn't have the .yarn folder, it cannot run the pinned Yarn 4 engine.
# .yarn/plugins/ (Extensions):
# Yarn v4 splits out features into plugins (e.g., @yarnpkg/plugin-interactive-tools or @yarnpkg/plugin-workspace-tools). If your repo relies on plugins, they reside here.
# .yarn/cache/ (Zero-Installs, if used):
# Some modern Yarn projects check compressed package archives into .yarn/cache. If enabled, Yarn installs dependencies completely offline without reaching out to the internet.
# By copying .yarnrc.yml and .yarn alongside your package.json and lockfiles before running yarn install, you give the container both the installation rules and the Yarn 4 runtime binary needed to resolve and link dependencies properly.

# In Yarn Classic or npm, 23 only copy package.json and yarn.lock. 
# In Yarn Berry, if 23 forget to copy .yarnrc.yml or the .yarn/ directory before running install, the build fails or completely misses internal configurations and plugins.  
# .yarnrc.yaml - Yarn Berry package manager configuration file. 
# Classic: Uses .yarnrc (a flat key-value text format).
# Berry repos typically include a .yarn/ directory containing:
# .yarn/releases/: Bundled Yarn binary files (e.g., yarn-4.x.x.cjs).
# .yarn/plugins/: Custom or official plugins (like @yarnpkg/plugin-workspace-tools).
# .yarn/cache/: Zip archives of packages if using offline/zero-install mode.

COPY package.json yarn.lock* package-lock.json* .yarnrc.yml* ./
COPY .yarn* ./.yarn

# Install all dependencies (including devDependencies needed for compiling TS)
# The --frozen-lockfile flag instructs Yarn to install dependencies using only what is recorded
# in yarn.lock without updating or generating the file.  If package.json and yarn.lock are out of sync, the command fails immediately instead of modifying the lockfile.
# This construct guarantees that whichever package manager was committed to the repository, dependencies are installed deterministically without modifying the lockfile during the Docker build.
# When you run the first yarn install --frozen-lockfile, it downloads everything—including
# devDependencies like the TypeScript compiler, testing frameworks (Jest), linters (ESLint), and @types packages. You need these to compile the code (yarn build), but you do not need them to run the compiled JavaScript
# devDependencies often take up hundreds of megabytes of disk space. By running yarn install --production (or npm prune --production) after the build finishes, the package manager deletes all those development tools from the node_modules folder. When the final stage copies node_modules over, it only copies the bare minimum required to run the app.
# Leaving build tools, compilers, and test frameworks inside a production container increases the attack surface. If a vulnerability is found in a development package, your production scanner will flag your image. Pruning guarantees those tools never make it into the final container.

# The primary job of yarn install is to download third-party code that your application depends on.
# It looks at your package.json and yarn.lock files.
# It connects to the npm package registry, downloads all the libraries (like Express, React, TypeScript, or utility tools), and unpacks them into the node_modules folder.
# It prepares the environment so your code has all the external tools it needs to function.npm install do the same thing, but it uses package-lock.json instead of yarn.lock.

RUN if [ -f yarn.lock ]; then yarn install --frozen-lockfile; \
    else npm ci; fi

# Copy source code and build production bundle
# The primary job of yarn build is to translate or package your actual application source code into a format that a server or browser can run.
# It looks at the "scripts" section inside package.json file for a command named "build".
# It executes whatever command the developers defined there. In modern Node.js apps, the code is usually written in TypeScript.
# Node.js cannot run TypeScript natively. yarn build triggers the TypeScript compiler to translate all .ts files into standard .js files.
# It processes your source code and outputs the final production-ready files, usually into a folder named dist (which is why the Dockerfile later copies /usr/src/app/dist
# npm build do the same thing, but it uses package.json instead of yarn.lock.
COPY . .
RUN if [ -f yarn.lock ]; then yarn build; \
    else npm run build; fi


# Publish to Azure Artifacts via mounted secret (in-memory, discarded after RUN)
# RUN --mount=type=secret,id=AZURE_PAT \
#     export AZURE_PAT=$(cat /run/secrets/AZURE_PAT) && \
#     if [ -z "$AZURE_PAT" ]; then echo "ERROR: AZURE_PAT secret is empty!" && exit 1; fi && \
#     export BASE64_PAT=$(node -e "console.log(Buffer.from(process.env.AZURE_PAT.trim()).toString('base64'))") && \
#     echo "//pkgs.dev.azure.com/my-learning2026/DevOpsLearning/_packaging/store-app/npm/registry/:username=my-learning2026" > /root/.npmrc && \
#     echo "//pkgs.dev.azure.com/my-learning2026/DevOpsLearning/_packaging/store-app/npm/registry/:_password=${BASE64_PAT}" >> /root/.npmrc && \
#     echo "//pkgs.dev.azure.com/my-learning2026/DevOpsLearning/_packaging/store-app/npm/registry/:email=npmUser@example.com" >> /root/.npmrc && \
#     echo "//pkgs.dev.azure.com/my-learning2026/DevOpsLearning/_packaging/store-app/npm/:always-auth=true" >> /root/.npmrc && \
#     npm publish && \
#     rm -f /root/.npmrc

# Tells Yarn Berry to focus on the current workspace (or service) and install or keep only production dependencies (dependencies), stripping out all development dependencies (devDependencies) like TypeScript, 
# testing suites, linters, and build compilers.
RUN if [ -f yarn.lock ]; then \
      yarn workspaces focus --production; \
    else \
      npm prune --production; \
    fi

# Stage 2: Minimal Production Runtime
FROM node:20-alpine AS runner

WORKDIR /app

# Set production environment
ENV NODE_ENV=production

# node:alpine already includes a non-root 'node' user (UID 1000)
USER node

# Copy only production artifacts and node_modules
# Copying node_modules is necessary for Yarn Berry to install dependencies.
COPY --chown=node:node --from=builder /usr/src/app/node_modules ./node_modules
# The builder stage compiles TypeScript into vanilla JavaScript inside dist/. Without this, application code does not exist.
COPY --chown=node:node --from=builder /usr/src/app/dist ./dist
# Node.js requires package.json at runtime to determine module resolution (such as "type": "module" vs CommonJS), app entry points, and service version metadata. It is copied directly from the build context because the local file is identical to the one inside the builder.
COPY --chown=node:node package.json ./

ENTRYPOINT ["node", "dist/main.js"]


# When to add mvn deploy to CI/CD pipeline
# mvn deploy belongs in a pipeline only when the build's output is something another build needs to consume as a dependency — not when the output is a thing that gets run (an app, a service, a container).
# You're building a shared/internal library
# Example: company-logger, company-security-utils, a shared DTO/model module, a common validation library. Its whole purpose is to be declared in other projects' pom.xml files:
# xml
#<dependency>
#    <groupId>com.mycompany</groupId>
#    <artifactId>company-logger</artifactId>
#    <version>1.0.5</version>
#</dependency>
# Without mvn deploy publishing it to a feed (Azure Artifacts, Nexus, Artifactory), no other project can ever resolve that dependency. This is the case you already have — company-logger-ci.yml.
# If you write a custom Maven plugin your org uses across projects (<plugin><groupId>com.mycompany</groupId>...), it has to be deployed to a repo for mvn to fetch it at build time, same mechanism as a library dependency.
# You maintain a multi-module Maven "platform" or BOM (Bill of Materials)
# A parent POM or BOM that centralizes dependency versions for many services (<dependencyManagement> imports) needs to be deployed so those services' pom.xml files can inherit/import it by version.
# You publish SDKs/client libraries for external or cross-team consumption
# E.g., a Java client SDK for your public API, published so partner teams or external customers can add it as a dependency.
# napshot/local-repo installs in a monorepo build chain (edge case)
# Sometimes mvn install (not deploy) is used to put a module in the local .m2 cache so a sibling module in the same monorepo build can immediately consume it without round-tripping through a remote feed. That's install, not deploy, and it's local-only — worth knowing they're different goals.
# What all of these have in common: the artifact is consumed by other Maven builds via <dependency> resolution. cart fails that test — nothing declares <artifactId>cart</artifactId> in a pom.xml; it's consumed by Docker (COPY --from=builder) and ultimately run by a JVM in a container, not resolved as a Maven coordinate by anyone. That's the dividing line: if nothing will ever write <dependency>...cart...</dependency>, cart-ci.yml never needs mvn deploy.