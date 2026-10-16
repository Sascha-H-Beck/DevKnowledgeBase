# GitHub Actions — Build & Deploy a Java Container

A reference guide for automating the build and deployment of a Java application using **GitHub Actions**, a **multi-stage Dockerfile**, and a **Docker Compose** file.

---

## Prerequisites

| Requirement | Purpose |
| :--- | :--- |
| GitHub repository | Hosts source code and workflow files |
| Docker Hub account (or other registry) | Stores built container images |
| Remote server with Docker & Docker Compose | Deployment target |
| SSH key pair | Passwordless access from Actions to the server |

### GitHub Secrets to Configure

Navigate to **Settings > Secrets and variables > Actions** in your repository and add:

| Secret | Value |
| :--- | :--- |
| `DOCKER_USERNAME` | Docker Hub username |
| `DOCKER_PASSWORD` | Docker Hub access token |
| `SSH_PRIVATE_KEY` | Private key for the deployment server |
| `SSH_HOST` | Server IP or hostname |
| `SSH_USER` | SSH login user on the server |

---

## Project Structure

```text
project-root/
├── .github/
│   └── workflows/
│       └── deploy.yml              # GitHub Actions workflow
├── docker/
│   └── java/
│       └── Dockerfile              # Multi-stage Java container definition
├── src/
│   └── main/
│       ├── java/                   # Application source code
│       └── resources/
│           └── application.yml     # Spring Boot configuration
├── pom.xml                         # Maven build file (or build.gradle)
├── docker-compose.yml              # Service orchestration
└── .env.example                    # Environment variable template
```

---

## Dockerfile (Multi-Stage Build)

Uses a build stage to compile the JAR and a slim runtime stage for the final image.

### Maven Project

```dockerfile
FROM eclipse-temurin:21-jdk AS build

WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline -B
COPY src ./src
RUN mvn package -DskipTests -B

FROM eclipse-temurin:21-jre

WORKDIR /app
COPY --from=build /app/target/*.jar app.jar

RUN groupadd -r appuser && useradd -r -g appuser appuser
USER appuser

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Gradle Project

```dockerfile
FROM eclipse-temurin:21-jdk AS build

WORKDIR /app
COPY gradle/ gradle/
COPY gradlew build.gradle settings.gradle ./
RUN ./gradlew dependencies --no-daemon
COPY src ./src
RUN ./gradlew bootJar --no-daemon -x test

FROM eclipse-temurin:21-jre

WORKDIR /app
COPY --from=build /app/build/libs/*.jar app.jar

RUN groupadd -r appuser && useradd -r -g appuser appuser
USER appuser

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

> [!TIP]
> The multi-stage build keeps the final image small — only the JRE and the compiled JAR are included, not the JDK or source code.

---

## Docker Compose File

Orchestrates the Java app together with a PostgreSQL database.

```yaml
services:
  app:
    image: ${DOCKER_USERNAME}/java-app:${IMAGE_TAG:-latest}
    build:
      context: .
      dockerfile: docker/java/Dockerfile
    ports:
      - "8080:8080"
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://db:5432/${DB_NAME}
      - SPRING_DATASOURCE_USERNAME=${DB_USER}
      - SPRING_DATASOURCE_PASSWORD=${DB_PASSWORD}
      - JAVA_OPTS=-Xms256m -Xmx512m
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

  db:
    image: postgres:16
    volumes:
      - db_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=${DB_NAME}
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d ${DB_NAME}"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

volumes:
  db_data:
```

---

## GitHub Actions Workflow

File: `.github/workflows/deploy.yml`

```yaml
name: Build and Deploy Java App

on:
  push:
    branches: [main]

env:
  IMAGE_NAME: ${{ secrets.DOCKER_USERNAME }}/java-app
  JAVA_VERSION: "21"

jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: ${{ env.JAVA_VERSION }}
          cache: maven

      - name: Run tests
        run: mvn verify -B

  build:
    name: Build & Push Image
    runs-on: ubuntu-latest
    needs: test

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=
            type=raw,value=latest

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: .
          file: docker/java/Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    name: Deploy to Server
    runs-on: ubuntu-latest
    needs: build

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Copy files to server
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: "docker-compose.yml,.env.example"
          target: /opt/java-app

      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1.2.0
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /opt/java-app
            export DOCKER_USERNAME=${{ secrets.DOCKER_USERNAME }}
            export IMAGE_TAG=${{ github.sha }}
            docker compose pull app
            docker compose up -d --no-build
            docker image prune -f
```

---

## Workflow Breakdown

### Pipeline Flow

```text
test ──► build ──► deploy
```

Each job depends on the previous one via `needs`.

### Trigger

```yaml
on:
  push:
    branches: [main]
```

The workflow runs on every push to `main`. Other common triggers:

| Trigger | Use Case |
| :--- | :--- |
| `push: branches: [main]` | Deploy on merge to main |
| `release: types: [published]` | Deploy on tagged release |
| `workflow_dispatch:` | Manual trigger via GitHub UI |

### Test Job

| Step | Action | Purpose |
| :--- | :--- | :--- |
| Checkout | `actions/checkout@v4` | Clones the repository |
| Set up JDK | `actions/setup-java@v4` | Installs Temurin JDK 21 with Maven cache |
| Run tests | `mvn verify -B` | Compiles and runs all tests |

### Build Job

| Step | Action | Purpose |
| :--- | :--- | :--- |
| Checkout | `actions/checkout@v4` | Clones the repository |
| Login | `docker/login-action@v3` | Authenticates with Docker Hub |
| Metadata | `docker/metadata-action@v5` | Generates image tags from the Git SHA |
| Buildx | `docker/setup-buildx-action@v3` | Enables build cache support |
| Build & Push | `docker/build-push-action@v6` | Multi-stage build, pushes the image |

### Deploy Job

| Step | Action | Purpose |
| :--- | :--- | :--- |
| Checkout | `actions/checkout@v4` | Clones the repository |
| Copy files | `appleboy/scp-action` | Transfers Compose file to the server |
| SSH deploy | `appleboy/ssh-action` | Pulls the new image and restarts services |

> [!IMPORTANT]
> The `needs` chain (`test → build → deploy`) ensures tests pass before any image is built, and the image exists before deployment begins.

---

## Common Enhancements

### Gradle Variant

Replace the test step for Gradle-based projects.

```yaml
- name: Set up JDK
  uses: actions/setup-java@v4
  with:
    distribution: temurin
    java-version: ${{ env.JAVA_VERSION }}
    cache: gradle

- name: Run tests
  run: ./gradlew check --no-daemon
```

### JVM Tuning via Environment Variables

Pass JVM flags through Docker Compose or the workflow.

```yaml
environment:
  - JAVA_OPTS=-Xms512m -Xmx1024m -XX:+UseG1GC
```

Override at container start:

```yaml
entrypoint: ["java", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
```

### Limiting Deploys to Release Tags

```yaml
on:
  push:
    tags:
      - "v*"
```

### Health Check After Deploy

Appends a verification step to the deploy job using the Spring Boot Actuator health endpoint.

```yaml
- name: Verify deployment
  uses: appleboy/ssh-action@v1.2.0
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USER }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    script: |
      for i in 1 2 3 4 5; do
        STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/actuator/health)
        if [ "$STATUS" = "200" ]; then
          echo "Health check passed"
          exit 0
        fi
        sleep 5
      done
      echo "Health check failed"
      exit 1
```

### Rollback on Failure

```yaml
- name: Rollback on failure
  if: failure()
  uses: appleboy/ssh-action@v1.2.0
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USER }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    script: |
      cd /opt/java-app
      docker compose down
      export IMAGE_TAG=latest
      docker compose up -d --no-build
```

---

## Using a Private Registry (GHCR)

To use GitHub Container Registry instead of Docker Hub, replace the login step and image name.

```yaml
env:
  IMAGE_NAME: ghcr.io/${{ github.repository_owner }}/java-app

# In the build job:
- name: Log in to GHCR
  uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}
```

> [!NOTE]
> `GITHUB_TOKEN` is provided automatically — no extra secret needed. Ensure the package visibility matches your repository settings under **Settings > Packages**.

---

## Quick Reference

| Command / File | Purpose |
| :--- | :--- |
| `.github/workflows/deploy.yml` | Workflow definition |
| `docker/java/Dockerfile` | Multi-stage Java image build |
| `docker-compose.yml` | Multi-container orchestration |
| `mvn verify -B` | Run Maven tests in batch mode |
| `./gradlew check` | Run Gradle tests |
| `docker compose pull` | Fetch latest image from registry |
| `docker compose up -d` | Start services in background |
| `docker compose down` | Stop and remove containers |
| `docker compose logs -f app` | Stream application logs |
| `docker image prune -f` | Remove unused images |
