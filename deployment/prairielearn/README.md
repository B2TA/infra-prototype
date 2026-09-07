# PrairieLearn test deployment

This Compose project runs a small, single-container PrairieLearn instance for
integration testing. PrairieLearn's PostgreSQL data is stored in a named Docker
volume and course repositories are stored under `courses/` on the host.

This deliberately does not mount the Docker socket or run the grader/workspace
hosts. Standard PrairieLearn questions work, but external graders and coding
workspaces do not. This keeps the service unprivileged and avoids the extra
compute infrastructure that PrairieLearn recommends for those optional
features.

## Configure Google OAuth

PrairieLearn supports Google OAuth 2 as the login method for self-hosted
instances. Create a Google OAuth client before starting the service:

1. In Google Cloud Console, create or select a project.
2. Under **Google Auth Platform**, configure the OAuth consent screen.
3. Create an OAuth client with application type **Web application**.
4. Add `https://pl.example.com` as an authorized JavaScript origin, replacing
   the example hostname with the public PrairieLearn hostname.
5. Add `https://pl.example.com/pl/oauth2callback` as an authorized redirect
   URI, using the same hostname.
6. Copy the configuration template and fill in the hostname, cookie domain,
   client ID, client secret, and two application keys:

   ```bash
   cp config.example.json config.json
   chmod 600 config.json
   openssl rand -hex 32
   openssl rand -hex 32
   ```

`config.json` is ignored by Git because it contains the OAuth client secret.
The values of `serverCanonicalHost` and `googleRedirectUrl` must use the public
HTTPS hostname, and the callback path must remain `/pl/oauth2callback`.
`cookieDomain` must begin with a dot; for `pl.example.com`, use
`.pl.example.com`. Put the two different 64-character hexadecimal values from
the `openssl` commands in `secretKey` and `databaseEncryptionKey`.

The template is tuned for this small instance:

- `workersCount` is `1`, instead of PrairieLearn's default of one Python
  question-code worker per CPU. This lowers idle memory usage but queues
  simultaneous question rendering and grading behind that one worker.
- `postgresqlPoolSize` is capped at `10`, instead of the default maximum of
  `100`. Connections are opened on demand, so this mainly limits memory growth
  under concurrency rather than reducing idle memory.
- `workspaceEnable` is `false`. External graders are also unavailable because
  this deployment intentionally does not mount the Docker socket.

Do not set `workersExecutionMode` to `disabled`: the native Python worker is
needed for ordinary questions and elements that execute `server.py`.

## Start PrairieLearn

From this directory, start the service:

```bash
docker compose pull
docker compose up -d
docker compose ps
```

The image is large, so its first download can take a few minutes. The default
binding is `127.0.0.1:3001`; it is not exposed directly to the internet. Point
the hostname's DNS record at the EC2 Elastic IP, then add this site to the
host's Caddyfile:

```caddyfile
pl.example.com {
    reverse_proxy 127.0.0.1:3001
}
```

After reloading Caddy, verify PrairieLearn's documented health endpoint:

```bash
curl --fail https://pl.example.com/pl/webhooks/ping
```

## Create the first administrator

First sign in through Google once so PrairieLearn creates your user. Then open
PrairieLearn's PostgreSQL shell:

```bash
docker compose exec app psql -U postgres -d postgres
```

Find your PrairieLearn user ID:

```sql
SELECT id, uid, uin, name FROM users;
```

Promote that user, replacing `1` with the correct ID:

```sql
INSERT INTO administrators (user_id) VALUES (1);
```

Exit with `\q`, then reload PrairieLearn in the browser.

## Add a course

Place each course repository in `courses/`, for example:

```bash
git clone https://github.com/example/course.git courses/course
```

In PrairieLearn, go to **Admin**, add a course, and use its container path:

```text
/courses/course
```

Keeping repositories in this bind-mounted directory preserves them when the
application container is replaced.

## Operate and remove the instance

View logs or update to the current `us-prod-live` image:

```bash
docker compose logs --follow app
docker compose pull
docker compose up -d
```

Stop the service while preserving data:

```bash
docker compose down
```

To discard the database permanently, including users and course-instance data:

```bash
docker compose down -v
```

Course repositories under `courses/` are not deleted by `down -v`.

## Scope and references

This is a low-footprint test deployment, not a high-availability production
design. Back up both the PostgreSQL volume and `courses/` before storing data
that matters.

- [PrairieLearn: Using Docker Compose](https://docs.prairielearn.com/running-in-production/docker-compose/)
- [PrairieLearn: Running in Production](https://docs.prairielearn.com/running-in-production/setup/)
- [PrairieLearn: User Authentication](https://docs.prairielearn.com/running-in-production/authentication/)
- [PrairieLearn: Admin User Setup](https://docs.prairielearn.com/running-in-production/admin-user/)
