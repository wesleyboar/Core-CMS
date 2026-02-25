# AGENTS.md

## Cursor Cloud specific instructions

This is a **Docker-based Django CMS** project. All application code runs inside Docker containers.

### Services

| Service | Container | Port |
| --- | --- | --- |
| Django CMS (app) | `core_cms` | `localhost:8000` |
| PostgreSQL 14.9 | `core_cms_postgres` | `5432` (internal) |
| Elasticsearch 7.17 | `core_cms_elasticsearch` | `localhost:9201` |

### Makefile commands

Use the `Makefile` instead of raw `docker compose` commands:

| Command | Purpose |
| --- | --- |
| `make setup` | One-command full setup (see caveat below) |
| `make build` | Build Docker images |
| `make start` | Start containers (`ARGS="--detach"` for background) |
| `make stop` | Stop containers |
| `make clean` | Stop containers, remove volumes and images |

### First-time setup

```sh
# 1. Create postgres secret files (needed for volume mounts, not created by make setup)
mkdir -p conf/postgres
echo "taccsite" > conf/postgres/pg_db.secret
echo "postgresadmin" > conf/postgres/pg_user.secret
echo "taccforever" > conf/postgres/pg_password.secret

# 2. Run the setup script (handles everything else)
make setup
```

`make setup` (i.e. `bin/setup-cms.sh`) handles: settings file creation, Docker build, container startup, readiness polling, migrations, superuser creation, CSS build, and `collectstatic`. In a non-interactive shell it auto-creates a default superuser (`admin`/`admin`); in a TTY it prompts interactively.

### Key gotchas

- **Settings files** are gitignored. Created from `*.example.py` by `bin/setup-cms.sh` or manually.
- The `secrets.py` Elasticsearch host should be `core_cms_elasticsearch` (the Docker hostname), not `elasticsearch`.
- Docker commands may need `sudo` depending on the environment.

### Lint, test, build

- **Lint:** `docker exec core_cms flake8 taccsite_cms/ --max-line-length=120` (pre-existing warnings expected)
- **Tests:** `docker exec core_cms python manage.py test taccsite_cms.contrib.taccsite_sample --no-input`
- **CSS build:** `npm run build` (on host)
- **Collect static:** `docker exec core_cms python manage.py collectstatic --no-input`

See `README.md` for full setup instructions.
