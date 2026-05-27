# Dokploy Deployment Guide

This project can be deployed on Dokploy as a Docker Compose application, but you should **not** use the VPS-oriented `Caddy` service there.

Dokploy already manages public routing and domains through its own Traefik integration, so binding `80:80` and `443:443` inside your app stack will conflict with Dokploy's proxy layer.

## What to deploy in Dokploy

Use a Docker Compose app with these services only:

- `frontend`
- `backend`
- `postgres`
- `redis`

Do **not** deploy:

- `caddy`

## Why

- Dokploy recommends configuring domains in the Dokploy UI for Docker Compose apps.
- Dokploy writes environment variables to a `.env` file for the compose app, but containers only receive them if you use `env_file` or explicit `environment` mappings.
- Dokploy works best with Docker named volumes for persistence and backups.

## Important difference from plain VPS deployment

Dokploy deploys from your git repository.

That means your local `backend/.env` file is **not** available in Dokploy, because this repo ignores `**/.env`.

So for Dokploy you must do one of these:

1. Add the same values manually in the Dokploy `Environment Variables` UI.
2. Or upload/create an env file inside Dokploy and reference it explicitly.

The safer Dokploy-native approach is option 1, using Dokploy environment variables and explicit Compose mappings.

## Recommended Dokploy compose file

Create a Dokploy Compose application and use this as the compose content:

```yaml
services:
  frontend:
    build:
      context: ./frontend
      args:
        BACKEND_HOST: forgecrm-backend
        BACKEND_PORT: 3011
    container_name: forgecrm-frontend
    restart: unless-stopped
    depends_on:
      backend:
        condition: service_healthy
    networks:
      - forgecrm

  backend:
    build:
      context: ./backend
    container_name: forgecrm-backend
    restart: unless-stopped
    environment:
      NODE_ENV: ${NODE_ENV}
      PORT: ${PORT}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      DATABASE_URL: ${DATABASE_URL}
      POSTGRES_SSL: ${POSTGRES_SSL}
      REDIS_URL: ${REDIS_URL}
      JWT_SECRET: ${JWT_SECRET}
      FORGECRM_ENCRYPTION_KEY: ${FORGECRM_ENCRYPTION_KEY}
      CORS_ORIGIN: ${CORS_ORIGIN}
      META_API_VERSION: ${META_API_VERSION}
      META_WEBHOOK_VERIFY_TOKEN: ${META_WEBHOOK_VERIFY_TOKEN}
      META_ACCESS_TOKEN: ${META_ACCESS_TOKEN}
      MEDIA_DIR: ${MEDIA_DIR}
      ADMIN_EMAIL: ${ADMIN_EMAIL}
      ADMIN_PASSWORD: ${ADMIN_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://127.0.0.1:3011/health >/dev/null 2>&1 || exit 1"]
      interval: 15s
      timeout: 5s
      retries: 10
      start_period: 30s
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    volumes:
      - backend_uploads:/app/uploads
      - backend_media:/app/media
    networks:
      - forgecrm

  postgres:
    image: postgres:16-alpine
    container_name: forgecrm-db
    restart: unless-stopped
    environment:
      POSTGRES_DB: postgres
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d postgres"]
      interval: 10s
      timeout: 5s
      retries: 10
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./db/initdb:/docker-entrypoint-initdb.d:ro
      - ./db/migrations:/migrations:ro
    networks:
      - forgecrm

  redis:
    image: redis:7-alpine
    container_name: forgecrm-redis
    restart: unless-stopped
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    networks:
      - forgecrm

volumes:
  postgres_data:
  redis_data:
  backend_uploads:
  backend_media:

networks:
  forgecrm:
    driver: bridge
```

## Dokploy setup steps

1. Create a new `Docker Compose` application in Dokploy.
2. Connect the repository/branch that contains this project.
3. Paste the compose file above into Dokploy, or point Dokploy to a dedicated compose file with the same content.
4. In Dokploy `Environment Variables`, add the same values currently present in your local `backend/.env`.
5. In Dokploy, add your domain in the `Domains` tab and select the `frontend` service.
6. Set the public port for the domain target to `80`.
7. Do not expose the `backend`, `postgres`, or `redis` services publicly.

## Environment variables to add in Dokploy

Because `backend/.env` is not committed to git, Dokploy must receive these values through its own environment-variable UI.

Add at least:

```env
NODE_ENV=production
PORT=3011
POSTGRES_PASSWORD=...
DATABASE_URL=postgresql://postgres:...@forgecrm-db:5432/postgres
POSTGRES_SSL=false
REDIS_URL=redis://redis:6379
JWT_SECRET=...
FORGECRM_ENCRYPTION_KEY=...
CORS_ORIGIN=https://whatsapp.dhaanishchennai.in,http://localhost
META_API_VERSION=v21.0
META_WEBHOOK_VERIFY_TOKEN=...
MEDIA_DIR=/app/media
ADMIN_EMAIL=...
ADMIN_PASSWORD=...
```

Important:

- Copy the values from your current local `backend/.env` into Dokploy. Do not leave them only on your laptop.
- `DATABASE_URL` must keep `forgecrm-db` as the hostname, because that is the Postgres container name in the compose stack.
- `REDIS_URL` should stay `redis://redis:6379`.
- `PORT` should stay `3011`.
- `CORS_ORIGIN` should include your real Dokploy domain.

## First deployment behavior

On a fresh Dokploy volume:

- Postgres will initialize its data directory.
- `db/initdb/0000_schema.sql` will create the `coexistence` schema.
- `db/initdb/0010_run_migrations.sh` will apply all SQL files from `db/migrations`.
- Then the backend should start against the fully initialized schema.

If you already deployed once with a broken DB state, remove the old Postgres volume before redeploying so the init scripts run cleanly.

## Health and routing

- Frontend serves on internal port `80`.
- Backend serves on internal port `3011`.
- Dokploy should route the public domain to the `frontend` service on port `80`.
- The frontend nginx config already proxies `/api`, `/api/events`, `/uploads`, and `/l/` to `forgecrm-backend:3011`.

## After deploying

Check these in Dokploy:

1. `postgres` logs: migrations completed without SQL errors.
2. `backend` logs: app starts and listens on port `3011`.
3. `frontend` logs: nginx starts cleanly.
4. Domain status: points to `frontend` and serves the app.

Then verify in the browser:

1. `https://whatsapp.dhaanishchennai.in`
2. `https://whatsapp.dhaanishchennai.in/health` will not work, because health is on the backend, not the frontend.
3. A better backend check is opening the Dokploy terminal/logs for the backend service and confirming `/health` returns OK internally.

## Notes

- The repo's root `docker-compose.yml` is suitable for a plain VPS Docker host, not ideal for Dokploy, because it includes `Caddy`.
- For Dokploy, let Dokploy manage HTTPS and domain routing.
- Keep using named volumes so Dokploy backups can work correctly.
- Your local `backend/.env` is not uploaded automatically to Dokploy unless you explicitly recreate those values there.

## Official references

- Dokploy Docker Compose docs: https://docs.dokploy.com/docs/core/docker-compose
- Dokploy Domains for Docker Compose: https://docs.dokploy.com/docs/core/docker-compose/domains
- Dokploy Environment Variables: https://docs.dokploy.com/docs/core/variables
