# GitHub Actions — Build & Deploy an Elixir / Phoenix LiveView Container

A reference guide for automating the build and deployment of a **Phoenix LiveView** application using **GitHub Actions**, a **multi-stage Dockerfile**, and a **Docker Compose** file.

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
| `SECRET_KEY_BASE` | Phoenix secret key base (`mix phx.gen.secret`) |

---

## Project Structure

```text
project-root/
├── .github/
│   └── workflows/
│       └── deploy.yml              # GitHub Actions workflow
├── docker/
│   └── elixir/
│       └── Dockerfile              # Multi-stage Elixir container definition
├── lib/
│   ├── my_app/                     # Business logic
│   └── my_app_web/
│       └── live/                   # LiveView modules
├── assets/                         # JavaScript, CSS, Tailwind
├── config/
│   ├── config.exs
│   ├── dev.exs
│   ├── prod.exs
│   └── runtime.exs                 # Runtime configuration (env vars)
├── mix.exs                         # Project definition & dependencies
├── mix.lock
├── docker-compose.yml              # Service orchestration
├── rel/
│   └── env.sh.eex                  # Release environment template
└── .env.example                    # Environment variable template
```

---

## Dockerfile (Multi-Stage Build)

Three stages: dependency install, asset build, and slim release image.

```dockerfile
# --- Stage 1: Build dependencies and compile ---
ARG ELIXIR_VERSION=1.18
ARG OTP_VERSION=27
ARG DEBIAN_VERSION=bookworm-20240612-slim

ARG BUILDER_IMAGE="hexpm/elixir:${ELIXIR_VERSION}-erlang-${OTP_VERSION}-debian-${DEBIAN_VERSION}"
ARG RUNNER_IMAGE="debian:${DEBIAN_VERSION}"

FROM ${BUILDER_IMAGE} AS build

RUN apt-get update -y && apt-get install -y \
    build-essential \
    git \
    curl \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash - \
    && apt-get install -y nodejs \
    && npm install -g esbuild

WORKDIR /app

RUN mix local.hex --force && mix local.rebar --force

ENV MIX_ENV=prod

COPY mix.exs mix.lock ./
RUN mix deps.get --only prod
RUN mkdir config
COPY config/config.exs config/prod.exs config/
RUN mix deps.compile

# --- Stage 2: Build assets ---
COPY assets/ assets/
COPY priv/ priv/
COPY lib/ lib/

RUN mix assets.deploy

# --- Stage 3: Compile release ---
COPY config/runtime.exs config/
COPY rel/ rel/
RUN mix compile
RUN mix release

# --- Stage 4: Runtime image ---
FROM ${RUNNER_IMAGE}

RUN apt-get update -y && apt-get install -y \
    libstdc++6 \
    openssl \
    libncurses5 \
    locales \
    ca-certificates \
    curl \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

RUN sed -i '/en_US.UTF-8/s/^# //g' /etc/locale.gen && locale-gen
ENV LANG=en_US.UTF-8
ENV LANGUAGE=en_US:en
ENV LC_ALL=en_US.UTF-8

WORKDIR /app

RUN groupadd -r appuser && useradd -r -g appuser appuser

COPY --from=build --chown=appuser:appuser /app/_build/prod/rel/my_app ./

USER appuser

ENV PHX_HOST=localhost
ENV PORT=4000

EXPOSE 4000

ENTRYPOINT ["bin/my_app"]
CMD ["start"]
```

> [!TIP]
> Replace `my_app` with your actual application name (the one defined in `mix.exs` under `:app`). The release name must match.

> [!NOTE]
> Dependencies are fetched and compiled before copying application code. This maximises Docker layer caching — unchanged dependencies won't be re-downloaded on every build.

---

## Docker Compose File

Orchestrates the Phoenix app together with a PostgreSQL database.

```yaml
services:
  app:
    image: ${DOCKER_USERNAME}/my-app:${IMAGE_TAG:-latest}
    build:
      context: .
      dockerfile: docker/elixir/Dockerfile
    ports:
      - "4000:4000"
    environment:
      - DATABASE_URL=ecto://postgres:${DB_PASSWORD}@db:5432/${DB_NAME}
      - SECRET_KEY_BASE=${SECRET_KEY_BASE}
      - PHX_HOST=${PHX_HOST}
      - PORT=4000
      - POOL_SIZE=10
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
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d ${DB_NAME}"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

volumes:
  db_data:
```

---

## Runtime Configuration

Phoenix releases use `config/runtime.exs` to read environment variables at container start. This file is essential for Docker deployments.

```elixir
# config/runtime.exs
import Config

if config_env() == :prod do
  database_url =
    System.get_env("DATABASE_URL") ||
      raise "DATABASE_URL environment variable is missing"

  config :my_app, MyApp.Repo,
    url: database_url,
    pool_size: String.to_integer(System.get_env("POOL_SIZE") || "10")

  secret_key_base =
    System.get_env("SECRET_KEY_BASE") ||
      raise "SECRET_KEY_BASE environment variable is missing"

  host = System.get_env("PHX_HOST") || "localhost"
  port = String.to_integer(System.get_env("PORT") || "4000")

  config :my_app, MyAppWeb.Endpoint,
    url: [host: host, port: 443, scheme: "https"],
    http: [ip: {0, 0, 0, 0}, port: port],
    secret_key_base: secret_key_base,
    server: true
end
```

> [!IMPORTANT]
> The `server: true` option is required for releases — without it, the HTTP server does not start.

---

## GitHub Actions Workflow

File: `.github/workflows/deploy.yml`

```yaml
name: Build and Deploy Phoenix App

on:
  push:
    branches: [main]

env:
  IMAGE_NAME: ${{ secrets.DOCKER_USERNAME }}/my-app
  MIX_ENV: test
  ELIXIR_VERSION: "1.18"
  OTP_VERSION: "27"

jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest

    services:
      db:
        image: postgres:16
        ports:
          - 5432:5432
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: my_app_test
        options: >-
          --health-cmd "pg_isready -U postgres"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Elixir
        uses: erlef/setup-beam@v1
        with:
          elixir-version: ${{ env.ELIXIR_VERSION }}
          otp-version: ${{ env.OTP_VERSION }}

      - name: Cache deps
        uses: actions/cache@v4
        with:
          path: |
            deps
            _build
          key: ${{ runner.os }}-mix-${{ hashFiles('mix.lock') }}
          restore-keys: ${{ runner.os }}-mix-

      - name: Install dependencies
        run: mix deps.get

      - name: Compile project
        run: mix compile --warnings-as-errors

      - name: Run tests
        run: mix test
        env:
          DATABASE_URL: ecto://postgres:postgres@localhost:5432/my_app_test

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
          file: docker/elixir/Dockerfile
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
          target: /opt/my-app

      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1.2.0
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /opt/my-app
            export DOCKER_USERNAME=${{ secrets.DOCKER_USERNAME }}
            export IMAGE_TAG=${{ github.sha }}
            docker compose pull app
            docker compose up -d --no-build
            docker compose exec app bin/my_app eval "MyApp.Release.migrate()"
            docker image prune -f
```

---

## Workflow Breakdown

### Pipeline Flow

```text
test ──► build ──► deploy (+ migrate)
```

Each job depends on the previous one via `needs`.

### Trigger

```yaml
on:
  push:
    branches: [main]
```

| Trigger | Use Case |
| :--- | :--- |
| `push: branches: [main]` | Deploy on merge to main |
| `release: types: [published]` | Deploy on tagged release |
| `workflow_dispatch:` | Manual trigger via GitHub UI |

### Test Job

| Step | Action | Purpose |
| :--- | :--- | :--- |
| Checkout | `actions/checkout@v4` | Clones the repository |
| Set up Beam | `erlef/setup-beam@v1` | Installs Elixir + OTP |
| Cache deps | `actions/cache@v4` | Caches `deps/` and `_build/` |
| Install deps | `mix deps.get` | Fetches Hex packages |
| Compile | `mix compile --warnings-as-errors` | Compiles with strict warnings |
| Tests | `mix test` | Runs ExUnit test suite against PostgreSQL service |

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
| SSH deploy | `appleboy/ssh-action` | Pulls image, starts services, runs Ecto migrations |

> [!IMPORTANT]
> Migrations run **after** the new container starts via `docker compose exec`. This requires the app to be healthy before the migration command executes.

---

## Database Migrations in Releases

Since `mix` is not available inside a release, define a migration module.

```elixir
# lib/my_app/release.ex
defmodule MyApp.Release do
  @app :my_app

  def migrate do
    load_app()

    for repo <- repos() do
      {:ok, _, _} =
        Ecto.Migrator.with_repo(repo, &Ecto.Migrator.run(&1, :up, all: true))
    end
  end

  def rollback(repo, version) do
    load_app()

    {:ok, _, _} =
      Ecto.Migrator.with_repo(repo, &Ecto.Migrator.run(&1, :down, to: version))
  end

  defp repos do
    Application.fetch_env!(@app, :ecto_repos)
  end

  defp load_app do
    Application.ensure_all_started(:ssl)
    Application.load(@app)
  end
end
```

Invoke from the running container:

```bash
docker compose exec app bin/my_app eval "MyApp.Release.migrate()"
```

---

## Common Enhancements

### Adding Dialyzer to the Test Job

```yaml
- name: Cache PLTs
  uses: actions/cache@v4
  with:
    path: priv/plts
    key: ${{ runner.os }}-plt-${{ hashFiles('mix.lock') }}

- name: Run Dialyzer
  run: mix dialyzer
```

### Adding Credo for Code Quality

```yaml
- name: Run Credo
  run: mix credo --strict
```

### LiveView Asset Verification

Ensures Tailwind and esbuild produce output correctly.

```yaml
- name: Verify assets compile
  run: mix assets.deploy
  env:
    MIX_ENV: prod
```

### Limiting Deploys to Release Tags

```yaml
on:
  push:
    tags:
      - "v*"
```

### Health Check After Deploy

Uses the Phoenix health endpoint (available since Phoenix 1.7).

```yaml
- name: Verify deployment
  uses: appleboy/ssh-action@v1.2.0
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USER }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    script: |
      for i in 1 2 3 4 5; do
        STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:4000/healthz)
        if [ "$STATUS" = "200" ]; then
          echo "Health check passed"
          exit 0
        fi
        sleep 5
      done
      echo "Health check failed"
      exit 1
```

> [!TIP]
> Add a health check route in your router if your Phoenix version doesn't include one by default:
>
> ```elixir
> get "/healthz", PageController, :health
> ```

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
      cd /opt/my-app
      docker compose down
      export IMAGE_TAG=latest
      docker compose up -d --no-build
```

---

## Using a Private Registry (GHCR)

To use GitHub Container Registry instead of Docker Hub, replace the login step and image name.

```yaml
env:
  IMAGE_NAME: ghcr.io/${{ github.repository_owner }}/my-app

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
| `docker/elixir/Dockerfile` | Multi-stage Elixir/Phoenix image build |
| `docker-compose.yml` | Multi-container orchestration |
| `config/runtime.exs` | Runtime env var configuration for releases |
| `lib/my_app/release.ex` | Migration module for releases |
| `mix phx.gen.secret` | Generate a `SECRET_KEY_BASE` value |
| `mix test` | Run ExUnit test suite |
| `mix compile --warnings-as-errors` | Strict compilation |
| `mix assets.deploy` | Build CSS/JS for production |
| `docker compose exec app bin/my_app eval "MyApp.Release.migrate()"` | Run Ecto migrations in the container |
| `docker compose pull` | Fetch latest image from registry |
| `docker compose up -d` | Start services in background |
| `docker compose down` | Stop and remove containers |
| `docker compose logs -f app` | Stream application logs |
| `docker image prune -f` | Remove unused images |
