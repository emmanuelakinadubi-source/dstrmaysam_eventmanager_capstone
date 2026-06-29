# Event Manager

An AI-powered corporate event planning platform that extracts event requirements, recommends ranked venues, and answers event-related questions via an intelligent chat interface.

## What this project does

- Accepts event briefs via file upload (PDF/Word) or free-text input
- Extracts structured requirements (city, attendees, budget, date) using an LLM agent
- Matches and ranks venues from a live Canvas API index using a multi-factor scoring model
- Provides a RAG-based chat interface for venue discovery and event Q&A
- Indexes venues automatically on a daily schedule via APScheduler
- Tracks family budget data and expense categories (Phase 2)

## Architecture

```mermaid
flowchart LR
    User --> UI["UI — Streamlit :8501"]
    UI --> API["API — FastAPI :8000"]
    API --> DB[("PostgreSQL")]
    API --> Chroma[("ChromaDB")]
    API --> LLM["Azure OpenAI"]
    API --> Canvas["Canvas Venues API"]
```

## Tech stack

| Layer | Technology |
|---|---|
| Frontend | Streamlit 1.37 |
| Backend | FastAPI + Python 3.11 |
| Database | PostgreSQL 16 |
| Vector store | ChromaDB |
| AI / LLM | Azure OpenAI (GPT-4.1-mini + embeddings) |
| Orchestration | LangChain agents |
| Containerisation | Docker + Docker Compose |
| CI/CD | GitHub Actions |

## Project structure

```text
family-finance-chat-app/
├── api/
│   ├── app/
│   │   ├── agent/            # LangChain event agent + tools
│   │   ├── core/             # Config, database, dependencies
│   │   ├── models/           # SQLAlchemy ORM models
│   │   ├── modules/          # Feature modules (events, family, vendors…)
│   │   ├── routes/           # FastAPI routers (chat, event, health, index)
│   │   ├── services/         # Business logic and scheduler
│   │   └── main.py
│   ├── alembic/              # Database migrations
│   ├── requirements.txt
│   └── Dockerfile
├── ui/
│   ├── pages/                # Multi-page Streamlit views
│   ├── services/             # API client helpers
│   ├── app.py
│   ├── requirements.txt
│   └── Dockerfile
├── .github/
│   └── workflows/
│       ├── ci.yml            # Pull request checks
│       └── cd.yml            # Build, push, and deploy on merge
├── .env.example
├── docker-compose.yml
└── README.md
```

---

## Local setup

### 1. Clone the repository

```bash
git clone https://github.com/your-org/family-finance-chat-app.git
cd family-finance-chat-app
```

### 2. Create the environment file

```bash
cp .env.example .env
```

Edit `.env` and fill in the required values (see [Environment variables](#environment-variables) below).

### 3. Start all services

```bash
docker compose up --build
```

### 4. Run database migrations

```bash
docker compose exec api alembic upgrade head
```

### 5. Open the app

| Service | URL |
|---|---|
| UI (Streamlit) | http://localhost:8501 |
| API docs (Swagger) | http://localhost:8000/docs |

---

## Environment variables

Copy `.env.example` to `.env`. The key variables are:

```dotenv
# App
APP_NAME=Event Manager
DEBUG=True
SECRET_KEY=change-me-in-production

# Database
DATABASE_URL=postgresql://postgres:postgres@db:5432/family_finance
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=family_finance

# Azure OpenAI
AZURE_OPENAI_API_KEY=
AZURE_OPENAI_ENDPOINT=
AZURE_OPENAI_API_VERSION=2024-12-01-preview
AZURE_OPENAI_DEPLOYMENT=gpt-4.1-mini
AZURE_OPENAI_EMBEDDING_DEPLOYMENT=text-embedding-3-small

# ChromaDB
CHROMA_PERSIST_DIR=/app/chroma_db

# Observability (optional)
LANGFUSE_PUBLIC_KEY=
LANGFUSE_SECRET_KEY=
MLFLOW_TRACKING_URI=
```

---

## CI/CD pipeline

The project uses **GitHub Actions** with two workflows:

| Workflow | File | Trigger | Purpose |
|---|---|---|---|
| CI | `.github/workflows/ci.yml` | Pull request → `main` | Lint, test, Docker build check |
| CD | `.github/workflows/cd.yml` | Push to `main` | Build + push images to GHCR, deploy |

### Pipeline overview

```
PR opened
  └── ci.yml
        ├── lint       (ruff — Python code quality)
        ├── test       (pytest — backend unit/integration tests)
        └── build      (docker build — verifies images compile)

Merge to main
  └── cd.yml
        ├── build-push (GHCR — publish api and ui images tagged :latest + :sha)
        └── deploy     (SSH — pull new images and restart services on server)
```

---

### Step 1 — Create the workflows directory

```bash
mkdir -p .github/workflows
```

---

### Step 2 — CI workflow

Create `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  pull_request:
    branches: [main]

jobs:
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install ruff
        run: pip install ruff

      - name: Run ruff
        run: ruff check api/app

  test:
    name: Test
    runs-on: ubuntu-latest
    needs: lint

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: family_finance_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd "pg_isready -U postgres"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    env:
      DATABASE_URL: postgresql://postgres:postgres@localhost:5432/family_finance_test
      SECRET_KEY: ci-secret-key
      DEBUG: "true"
      AZURE_OPENAI_API_KEY: placeholder
      AZURE_OPENAI_ENDPOINT: https://placeholder.openai.azure.com
      AZURE_OPENAI_API_VERSION: "2024-12-01-preview"
      AZURE_OPENAI_DEPLOYMENT: gpt-4.1-mini
      AZURE_OPENAI_EMBEDDING_DEPLOYMENT: text-embedding-3-small

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: pip install -r api/requirements.txt

      - name: Run migrations
        run: |
          cd api
          alembic upgrade head

      - name: Run tests
        run: |
          cd api
          pytest --tb=short -q

  build:
    name: Docker build check
    runs-on: ubuntu-latest
    needs: test

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build API image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: api/Dockerfile
          push: false
          tags: event-manager/api:ci

      - name: Build UI image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ui/Dockerfile
          push: false
          tags: event-manager/ui:ci
```

---

### Step 3 — CD workflow

Create `.github/workflows/cd.yml`:

```yaml
name: CD

on:
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_API: ghcr.io/${{ github.repository_owner }}/event-manager-api
  IMAGE_UI: ghcr.io/${{ github.repository_owner }}/event-manager-ui

jobs:
  build-push:
    name: Build & push images
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Extract metadata — API
        id: meta-api
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.IMAGE_API }}
          tags: |
            type=sha,prefix=sha-
            type=raw,value=latest

      - name: Build & push API image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: api/Dockerfile
          push: true
          tags: ${{ steps.meta-api.outputs.tags }}
          labels: ${{ steps.meta-api.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Extract metadata — UI
        id: meta-ui
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.IMAGE_UI }}
          tags: |
            type=sha,prefix=sha-
            type=raw,value=latest

      - name: Build & push UI image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ui/Dockerfile
          push: true
          tags: ${{ steps.meta-ui.outputs.tags }}
          labels: ${{ steps.meta-ui.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    name: Deploy to server
    runs-on: ubuntu-latest
    needs: build-push
    environment: production

    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.DEPLOY_HOST }}
          username: ${{ secrets.DEPLOY_USER }}
          key: ${{ secrets.DEPLOY_SSH_KEY }}
          script: |
            cd /opt/event-manager
            echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
            docker compose pull
            docker compose up -d --remove-orphans
            docker compose exec -T api alembic upgrade head
            docker image prune -f
```

---

### Step 4 — Configure GitHub Secrets

Go to your repository → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**.

| Secret | Description | Required for |
|---|---|---|
| `DEPLOY_HOST` | IP or hostname of your production server | CD — deploy job |
| `DEPLOY_USER` | SSH username on the server | CD — deploy job |
| `DEPLOY_SSH_KEY` | Private SSH key (server must have matching public key in `~/.ssh/authorized_keys`) | CD — deploy job |

> `GITHUB_TOKEN` is provided automatically by GitHub — you do not need to create it.

---

### Step 5 — Add ruff to dev dependencies

The CI lint step requires `ruff`. Add it to a dev requirements file:

```bash
# Create api/requirements-dev.txt
cat > api/requirements-dev.txt <<EOF
ruff>=0.4.0
pytest>=8.3.0
pytest-asyncio>=0.23.0
httpx>=0.27.0
EOF
```

Or append it to the existing `api/requirements.txt`:

```bash
echo "ruff>=0.4.0" >> api/requirements.txt
```

---

### Step 6 — Server setup (first deploy only)

On your production server, prepare the deployment directory:

```bash
ssh user@your-server
mkdir -p /opt/event-manager
cd /opt/event-manager
# Upload your docker-compose.yml and .env (with production values)
scp docker-compose.yml .env user@your-server:/opt/event-manager/
```

The CD workflow only pulls new images and restarts services — it does not overwrite config files already on the server.

---

### Step 7 — Protect the main branch

1. Repository → **Settings** → **Branches** → **Add branch protection rule**
2. Branch name pattern: `main`
3. Enable:
   - Require a pull request before merging
   - Require status checks to pass before merging
   - Select required checks: `Lint`, `Test`, `Docker build check`
   - Require branches to be up to date before merging

---

## Contribution guide

1. Fork the repository and create a feature branch:

```bash
git checkout -b feature/your-feature-name
```

2. Make changes and commit:

```bash
git add .
git commit -m "feat: describe your change"
```

3. Push and open a pull request against `main`. The CI workflow runs automatically on PR creation.

4. Once all checks pass and the PR is approved, merge to `main` to trigger the CD workflow and deploy.

### Contribution rules

- Keep changes small and focused
- Follow the existing folder structure
- Write or update tests for new behaviour
- Do not commit `.env` or any secrets
- Update this README when setup steps change
#   d s t r m a y s a m _ e v e n t m a n a g e r _ c a p s t o n e  
 