# AGENTS.md

## Cursor Cloud specific instructions

This is a **Docker-based Django CMS** project. All application code runs inside Docker containers.

### Services

| Service | Container | Port |
| --- | --- | --- |
| Django CMS (app) | `core_cms` | `localhost:8000` |
| PostgreSQL 14.9 | `core_cms_postgres` | `5432` (internal) |
| Elasticsearch 7.17 | `core_cms_elasticsearch` | `localhost:9201` |

### Starting services

```sh
# Start all containers (detached)
make start ARGS="--detach"
# or: sudo docker compose -f docker-compose.dev.yml up --detach
```

Wait for PostgreSQL (`docker exec core_cms_postgres pg_isready -U postgresadmin`) and Elasticsearch (`curl -s http://localhost:9201/_cluster/health`) before running Django management commands.

### Key gotchas

- **CSS build must happen on the host** (not inside the CMS container). The dev compose mounts `.:/code` which overwrites the Docker image's pre-built CSS. Run `npm ci && npm run build` on the host, then `docker exec core_cms python manage.py collectstatic --no-input` to serve styles.
- **Settings files** (`taccsite_cms/settings_custom.py`, `taccsite_cms/secrets.py`, `taccsite_cms/settings_local.py`) are gitignored. They are created from `*.example.py` files by `bin/setup-cms.sh` or manually.
- **Postgres secrets** (`conf/postgres/pg_db.secret`, `pg_user.secret`, `pg_password.secret`) must exist for the postgres container volume mount. Values match those in `docker-compose.dev.yml`.
- The `secrets.py` Elasticsearch host should be `core_cms_elasticsearch` (the Docker hostname), not `elasticsearch`.
- Docker commands may need `sudo` depending on the environment.

### Lint, test, build

- **Lint:** `docker exec core_cms flake8 taccsite_cms/ --max-line-length=120` (pre-existing warnings expected)
- **Tests:** `docker exec core_cms python manage.py test taccsite_cms.contrib.taccsite_sample --no-input`
- **CSS build:** `npm run build` (on host)
- **Collect static:** `docker exec core_cms python manage.py collectstatic --no-input`

See `README.md` for full setup instructions and `Makefile` for common commands.
