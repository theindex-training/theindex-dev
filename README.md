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

----------

## Production (VPS)

`docker-compose.prod.yml` is the production compose file. The VPS runs a copy of it, so when you change one, change the other:

| In this repo | On the VPS |
|--------------|------------|
| `docker-compose.prod.yml` | `/opt/theindex/docker-compose.yml` |
| `media/default.conf` | `/opt/theindex/media/default.conf` |
| `.env.example` (names only) | `/opt/theindex/.env` (real values, never committed) |

Services: `traefik` (HTTPS, Let's Encrypt), `db`, `api`, `front` (the portal), `web` (the public site, `theindex-web`) and `media` (nginx serving the article images from the `theindex_media` volume, read-only).

Volumes:

| Volume | Mounted in | Holds |
|--------|------------|-------|
| `theindex_pgdata` | `db` | the database |
| `theindex_media` | `api` (`/data/media`, read-write), `media` (`/srv/media`, read-only) | **public** media: article images, on the web at `media.theindex.training` |
| `theindex_media_private` | `api` only (`/data/media-private`) | **private** media: the lesson-plan PDFs workouts are imported from (CrossFit's copyrighted material). Only the API reads them, for ADMIN and TRAINER |

The API's media settings are in the `api` service: `MEDIA_STORAGE_DIR` and `MEDIA_PUBLIC_URL` (public), `MEDIA_PRIVATE_STORAGE_DIR` (private; it must be outside `MEDIA_STORAGE_DIR`, or the API refuses it). Without a setting the API still starts; only that area's requests fail (503), and a PDF import still fills the form without keeping the PDF. **Never mount `theindex_media_private` into `media` or any other web server**: everything in a served volume is public. Back up both media volumes along with the database: the rows in `media` point at their files.

Hosts (compose phase B of the public site's cutover, see `docs/decision-record.md` section 11 in `theindex-web`):
-   `theindex.training` → `web`
-   `app.theindex.training` → `front`, with `/api` → `api` (`CORS_ORIGIN=https://app.theindex.training`)
-   `media.theindex.training` → `media`
-   `www.theindex.training` → redirect to `theindex.training`

Things to know before changing it:
-   Every host needs a DNS `A` record that resolves **before** it appears in a router rule, or Let's Encrypt fails and rate-limits.
-   Each router has an `-http` twin that redirects to HTTPS; keep its rule's hosts the same as the HTTPS router's.
-   `media/default.conf` must exist on the VPS before `docker compose up`, or Docker creates an empty folder in its place and `media` doesn't start. Test it with `docker run --rm -v "$PWD/media/default.conf:/etc/nginx/conf.d/default.conf:ro" nginx:alpine nginx -t`.
-   `media` mounts the volume at `/srv/media`, not nginx's default `/usr/share/nginx/html`: Docker would copy nginx's default pages into the empty volume.
-   Never print or share `traefik/acme.json`: it holds the certificates' private keys.
-   A new named volume is created by the next `docker compose up -d`; nothing needs to be done by hand. The `api` deploy (on every push to `main` in `theindex-api`) runs `docker compose pull && docker compose up -d` in `/opt/theindex`, so change the VPS compose file **before** merging an API change that needs a new volume or setting.

Apply a change on the VPS:

```bash
cd /opt/theindex && docker compose config --quiet && docker compose pull && docker compose up -d
```
