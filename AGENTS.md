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

`make setup` (i.e. `bin/setup-cms.sh`) is the canonical setup script. It handles settings file creation, Docker build, container startup, readiness polling, migrations, superuser creation, and `collectstatic`. However, it uses `docker exec -it` and an interactive `createsuperuser` prompt, so **non-interactive agents** must replicate its steps without `-it`:

```sh
# 1. Create settings files (script does this automatically)
cp taccsite_cms/settings_custom.example.py taccsite_cms/settings_custom.py
cp taccsite_cms/secrets.example.py taccsite_cms/secrets.py
cp taccsite_cms/settings_local.example.py taccsite_cms/settings_local.py

# 2. Create postgres secret files (needed for volume mounts)
mkdir -p conf/postgres
echo "taccsite" > conf/postgres/pg_db.secret
echo "postgresadmin" > conf/postgres/pg_user.secret
echo "taccforever" > conf/postgres/pg_password.secret

# 3. Build and start
make build
make start ARGS="--detach"

# 4. Wait for services, then run Django setup (no -it flag)
docker exec core_cms python manage.py migrate
docker exec core_cms python manage.py shell -c \
  "from django.contrib.auth import get_user_model; User = get_user_model(); User.objects.create_superuser('admin','admin@example.com','admin') if not User.objects.filter(is_superuser=True).exists() else None"
docker exec core_cms python manage.py collectstatic --no-input

# 5. Build CSS on the host (not covered by make setup)
npm ci
npm run build
docker exec core_cms python manage.py collectstatic --no-input
```

### Key gotchas

- **CSS build must happen on the host.** The dev compose mounts `.:/code` which overwrites the Docker image's pre-built CSS. Run `npm ci && npm run build` on the host, then `collectstatic` again.
- **Settings files** are gitignored. Created from `*.example.py` by `bin/setup-cms.sh` or manually.
- The `secrets.py` Elasticsearch host should be `core_cms_elasticsearch` (the Docker hostname), not `elasticsearch`.
- Docker commands may need `sudo` depending on the environment.

### Lint, test, build

- **Lint:** `docker exec core_cms flake8 taccsite_cms/ --max-line-length=120` (pre-existing warnings expected)
- **Tests:** `docker exec core_cms python manage.py test taccsite_cms.contrib.taccsite_sample --no-input`
- **CSS build:** `npm run build` (on host)
- **Collect static:** `docker exec core_cms python manage.py collectstatic --no-input`

See `README.md` for full setup instructions.
