# TheIndex – Local Development (2-minute setup)
This repository provides a **local development environment** using Docker Compose profiles.

**Goal:**  
Each developer runs **only what they need**:
-   Frontend dev → API + DB in Docker, Angular locally
-   Backend dev → DB in Docker, API locally
-   Full stack → everything in Docker
-   Nobody installs PostgreSQL locally

----------

## Requirements
-   Docker installed on your machine
-   Access to GitHub Container Registry images:
    -   `ghcr.io/theindex-training/theindex-api:main`
    -   `ghcr.io/theindex-training/theindex-front:main`

### If pulling images fails with 403

Create a GitHub token with `read:packages`, then login:

`docker login ghcr.io -u <github-username>`

Paste the token when prompted.

----------

## Compose profiles overview


| Profile  | What runs                     |
|----------|-------------------------------|
| `db`     | PostgreSQL only               |
| `api`    | PostgreSQL + API container    |
| `front`  | Frontend container + API + DB |
| `full`   | Frontend + API + DB           |


----------

## Frontend developer workflow (recommended)

You run **API + DB in Docker**, Angular locally via `ng serve`.

### 1) Start DB + API

`docker compose -f docker-compose.dev.yml --profile api up -d`

-   API → http://localhost:3000
-   DB → localhost:5432

### 2) Run Angular locally

In the Angular repository:
1.  Run `npm start`

Your frontend calls the API via `/api/...`.

----------

## Backend developer workflow (recommended)

You run **DB in Docker**, API locally in watch mode.

### 1) Start DB only

`docker compose -f docker-compose.dev.yml --profile db up -d`

### 2) Run API locally

In the NestJS repository, set:

`DATABASE_URL=postgresql://theindex:theindex@localhost:5432/theindex`

Then run:

`npm run start:dev`

----------

## Full stack in Docker (optional)

If you want to run the **built frontend and API** entirely in Docker:

`docker compose -f docker-compose.dev.yml --profile full up -d`

Open:
-   Frontend → http://localhost:4201
-   API → http://localhost:3000

----------

## Stop / reset

Stop containers:

`docker compose -f docker-compose.dev.yml down`

Stop and **delete DB data** (⚠️ destructive):

`docker compose -f docker-compose.dev.yml down -v`
