# Canvas test deployment

This Compose project runs a disposable Canvas integration-test instance with
Canvas, delayed jobs, PostgreSQL with pgvector, and Redis.

## Configure and start

Copy the environment template and replace its placeholder values:

```bash
cp .env.canvas.example .env.canvas
chmod 600 .env.canvas
```

Update `CANVAS_DOMAIN` in `.env.canvas` with the hostname that users will use
to reach Canvas. Set `CANVAS_SSL=false` only when Canvas is served over plain
HTTP. The other files under `config/` are safe starting points for this test
deployment.

Initialize the database and first administrator account once:

```bash
docker compose --env-file .env.canvas up -d postgres redis
docker compose --env-file .env.canvas --profile init run --rm init
docker compose --env-file .env.canvas up -d web jobs
```

The administrator credentials come from `CANVAS_LMS_ADMIN_EMAIL` and
`CANVAS_LMS_ADMIN_PASSWORD` in `.env.canvas`.

To discard the test instance and all its data:

```bash
docker compose --env-file .env.canvas down -v
```
