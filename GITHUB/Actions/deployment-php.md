# GitHub Actions — Build & Deploy a PHP Container

A reference guide for automating the build and deployment of a PHP application using **GitHub Actions**, a **Dockerfile**, and a **Docker Compose** file.

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
│       └── deploy.yml          # GitHub Actions workflow
├── docker/
│   └── php/
│       └── Dockerfile          # PHP container definition
├── src/
│   └── index.php               # Application entry point
├── docker-compose.yml          # Service orchestration
└── .env.example                # Environment variable template
```

---

## Dockerfile

Defines the PHP runtime image with Apache.

```dockerfile
FROM php:8.3-apache

RUN apt-get update && apt-get install -y \
    libpng-dev \
    libzip-dev \
    unzip \
    && docker-php-ext-install pdo pdo_mysql gd zip opcache \
    && a2enmod rewrite \
    && rm -rf /var/lib/apt/lists/*

COPY src/ /var/www/html/

RUN chown -R www-data:www-data /var/www/html

EXPOSE 80
```

> [!TIP]
> Pin the PHP version (e.g. `php:8.3-apache`) instead of using `latest` to guarantee reproducible builds.

---

## Docker Compose File

Orchestrates the PHP app together with a MySQL database.

```yaml
services:
  app:
    image: ${DOCKER_USERNAME}/php-app:${IMAGE_TAG:-latest}
    build:
      context: .
      dockerfile: docker/php/Dockerfile
    ports:
      - "80:80"
    environment:
      - DB_HOST=db
      - DB_NAME=${DB_NAME}
      - DB_USER=${DB_USER}
      - DB_PASSWORD=${DB_PASSWORD}
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

  db:
    image: mysql:8.0
    volumes:
      - db_data:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=${DB_ROOT_PASSWORD}
      - MYSQL_DATABASE=${DB_NAME}
      - MYSQL_USER=${DB_USER}
      - MYSQL_PASSWORD=${DB_PASSWORD}
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
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
name: Build and Deploy PHP App

on:
  push:
    branches: [main]

env:
  IMAGE_NAME: ${{ secrets.DOCKER_USERNAME }}/php-app

jobs:
  build:
    name: Build & Push Image
    runs-on: ubuntu-latest

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

      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: .
          file: docker/php/Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}

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
          target: /opt/php-app

      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1.2.0
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /opt/php-app
            export DOCKER_USERNAME=${{ secrets.DOCKER_USERNAME }}
            export IMAGE_TAG=${{ github.sha }}
            docker compose pull app
            docker compose up -d --no-build
            docker image prune -f
```

---

## Workflow Breakdown

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

### Build Job

| Step | Action | Purpose |
| :--- | :--- | :--- |
| Checkout | `actions/checkout@v4` | Clones the repository |
| Login | `docker/login-action@v3` | Authenticates with Docker Hub |
| Metadata | `docker/metadata-action@v5` | Generates image tags from the Git SHA |
| Build & Push | `docker/build-push-action@v6` | Builds the Dockerfile and pushes the image |

### Deploy Job

| Step | Action | Purpose |
| :--- | :--- | :--- |
| Checkout | `actions/checkout@v4` | Clones the repository |
| Copy files | `appleboy/scp-action` | Transfers Compose file to the server |
| SSH deploy | `appleboy/ssh-action` | Pulls the new image and restarts services |

> [!IMPORTANT]
> The `needs: build` key ensures the deploy job waits until the image has been successfully built and pushed.

---

## Common Enhancements

### Adding a Build Cache

Speeds up repeated builds by caching Docker layers.

```yaml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Build and push image
  uses: docker/build-push-action@v6
  with:
    context: .
    file: docker/php/Dockerfile
    push: true
    tags: ${{ steps.meta.outputs.tags }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

### Limiting Deploys to Release Tags

```yaml
on:
  push:
    tags:
      - "v*"
```

### Health Check After Deploy

Appends a verification step to the deploy job.

```yaml
- name: Verify deployment
  uses: appleboy/ssh-action@v1.2.0
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USER }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    script: |
      for i in 1 2 3 4 5; do
        STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost)
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
      cd /opt/php-app
      docker compose down
      export IMAGE_TAG=latest
      docker compose up -d --no-build
```

---

## Using a Private Registry (GHCR)

To use GitHub Container Registry instead of Docker Hub, replace the login step and image name.

```yaml
env:
  IMAGE_NAME: ghcr.io/${{ github.repository_owner }}/php-app

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
| `docker/php/Dockerfile` | PHP image build instructions |
| `docker-compose.yml` | Multi-container orchestration |
| `docker compose pull` | Fetch latest image from registry |
| `docker compose up -d` | Start services in background |
| `docker compose down` | Stop and remove containers |
| `docker compose logs -f app` | Stream application logs |
| `docker image prune -f` | Remove unused images |
