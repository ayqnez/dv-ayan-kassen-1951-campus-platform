# Scholarship applications

> **CSS4003 DevOps 2 — Final Project (Narxoz University).**
> Assignment: [final-project.pdf](https://weeebdev-edu.github.io/css4003/pdfs/final-project.pdf) · Rubric: [final-project-rubric.pdf](https://weeebdev-edu.github.io/css4003/pdfs/final-project-rubric.pdf) · Report template: [final-project-report-template.pdf](https://weeebdev-edu.github.io/css4003/pdfs/final-project-report-template.pdf) · Free tier guide: [free-tier-guide.pdf](https://weeebdev-edu.github.io/css4003/pdfs/free-tier-guide.pdf)


A FastAPI service for recording scholarship applications. Each item is one
application with a title, description and an optional supporting document.
The local stack runs the API and PostgreSQL with Docker Compose.

## What the app does

- Stores **scholarship applications** in PostgreSQL.
- Stores **attachments** in object storage: a local folder, **Amazon S3** or
  **Google Cloud Storage**.
- Exposes **health checks**, **Prometheus metrics** and **JSON logs**.
- Has a **password policy checker** you can use in your security checks.

## Run locally

Use Docker with Compose. In PowerShell, run `Copy-Item .env.example .env`
for the first setup. Edit `.env` to add your full student ID to `APP_OWNER`
and replace the example credentials. Generate a fresh value for each secret
with `python -c "import secrets; print(secrets.token_urlsafe(48))"`.
`.env` is excluded from Git.

```powershell
docker compose up --build --detach
Invoke-RestMethod http://localhost:8000/readyz
Invoke-RestMethod http://localhost:8000/info
$application = @{ title = 'Merit scholarship application'; description = 'Fall semester' } | ConvertTo-Json
Invoke-RestMethod http://localhost:8000/api/items -Method Post -ContentType application/json -Body $application
Invoke-RestMethod http://localhost:8000/api/items
```

Readiness reports `ready: true` when both database and attachment storage
respond. The info endpoint reports the owner, database engine and version.
To run local monitoring, use `docker compose --profile monitoring up --build --detach`.
Prometheus listens on port 9090 and Grafana on 3000.
Use `docker compose --profile monitoring down` to stop the stack.

Run the tests and the linter without Docker:

```bash
make venv && make test && make lint
```

## Configuration (environment variables)

Every setting is an environment variable. **Any secret can instead be read from
a file**: set `NAME_FILE=/path/to/file` (this is how Kubernetes and Docker
secrets are mounted).

| Variable | Required | Example / default | Meaning |
|---|---|---|---|
| `SECRET_KEY` | **yes** | 48 random chars | App secret. The app refuses to start if it's missing or shorter than 32 characters. |
| `DB_URL` | **yes** | `postgresql+psycopg://app_rw:${DB_PASSWORD}@postgres:5432/campus` | Database connection string. |
| `DB_PASSWORD` | if `DB_URL` has `${DB_PASSWORD}` | from your secret manager | Filled into `DB_URL` and URL-encoded by the app. |
| `CLICKHOUSE_URL` | no | unset | Leave unset for this project. |
| `STORAGE_BACKEND` | no | `local` · `s3` · `gcs` | Where attachments go. |
| `BUCKET` | for s3/gcs | `/app/data/uploads` for local Compose | Bucket name or local folder for application documents. |
| `APP_NAME` | no | `Scholarship applications` | Shown on the home page. |
| `APP_OWNER` | for course submission | Your name and full student ID | Shown on `/` and `/info`. |
| `APP_VERSION`, `GIT_SHA` | no | set by your CI | Shown on `/info`; useful to prove which release is running. |
| `LOG_LEVEL`, `LOG_FORMAT` | no | `INFO`, `json` | Logging. Keep JSON in production. |
| `PASSWORD_MIN_LENGTH` | no | `16` | Minimum number of characters accepted by the checker. |
| `PASSWORD_REQUIRE_SYMBOL` | no | `false` | Set to `true` to require punctuation. Uppercase, lowercase, a digit and the common-password check always apply. |

Cloud credentials are **not** configured here. boto3 (AWS) and
google-cloud-storage (GCP) use their default credential chains: an instance
role or service account in the cloud, `AWS_ENDPOINT_URL` for local emulators.
Never put access keys in the image, the repo or `.env` files you commit.

## Endpoints

| Method & path | What it does |
|---|---|
| `GET /` | HTML page with app name, owner, version and the host that served it |
| `GET /info` | JSON: version, git sha, hostname, db engine, storage backend |
| `GET /healthz` | Liveness: the process is up (no dependencies checked) |
| `GET /readyz` | Readiness: database and storage reachable, else **503** |
| `GET /metrics` | Prometheus: `http_requests_total`, `http_request_duration_seconds`, `items_total`, `db_up` |
| `GET/POST /api/items` | List (with `limit`, `offset`) / create items |
| `GET/PUT/DELETE /api/items/{id}` | Read / update / delete one item |
| `POST/GET /api/items/{id}/attachment` | Upload / download a file (multipart field `file`, max 10 MB) |
| `POST /api/admin/check-password` | `{"password": "..."}` → which policy rules pass |

Interactive API docs: `http://localhost:8000/docs`.

## What's in the repo

```
app/                 the service (FastAPI, SQLAlchemy, prometheus-client)
app/sql/             EXAMPLE least-privilege roles for PostgreSQL / MySQL, ClickHouse DDL
tests/               pytest suite (uses SQLite, no services needed)
Dockerfile           multi-stage, non-root, with HEALTHCHECK
compose.yaml         LOCAL development only
deploy/local/        Prometheus + Grafana config for the local monitoring profile
.github/workflows/   ci.yml: lint + tests
```
