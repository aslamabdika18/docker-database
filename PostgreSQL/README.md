# PostgreSQL — Docker Setup

PostgreSQL 17 containerized for local development and production deployment, with pgAdmin 4 as the web-based GUI administration tool for development environments.

PostgreSQL is a powerful open-source object-relational database system with over 35 years of active development. It is well-suited for applications that require complex queries, ACID-compliant transactions, and strong data integrity — from small single-machine apps to large internet-facing services handling millions of rows.

---

## Services Included

| Service    | Image                   | Port               | Description                                         |
|------------|-------------------------|--------------------|-----------------------------------------------------|
| `postgres` | `postgres:17`           | `${POSTGRES_PORT}` | PostgreSQL 17 database server (dev)                 |
| `postgres` | `postgres:17-alpine`    | not exposed        | PostgreSQL 17 Alpine image, no host port (prod)     |
| `pgadmin`  | `dpage/pgadmin4:latest` | `${PGADMIN_PORT}`  | Web-based database administration UI (dev only)     |

---

## Prerequisites

- **Docker Engine** 24.0 or higher — [Install Docker](https://docs.docker.com/engine/install/)
- **Docker Compose** v2.22.0 or higher — bundled with Docker Desktop; verify with:
  ```bash
  docker compose version
  ```
- A terminal (Bash, Zsh, or PowerShell)
- `.env` file created from `.env.example` (see [Setup](#setup) below)

---

## Setup

**Step 1.** Navigate to the `PostgreSQL` directory from the repo root:

```bash
cd PostgreSQL
```

**Step 2.** Copy the example environment file:

```bash
# Linux / macOS
cp .env.example .env

# Windows (PowerShell)
Copy-Item .env.example .env
```

**Step 3.** Open `.env` in a text editor and replace every placeholder value:

| Variable            | Required to Change  | What to set                                           |
|---------------------|---------------------|-------------------------------------------------------|
| `POSTGRES_USER`     | Yes                 | Any alphanumeric username (e.g., `appuser`)           |
| `POSTGRES_PASSWORD` | **Yes — critical**  | A strong, unique password — minimum 16 characters     |
| `POSTGRES_DB`       | Yes                 | The name of the database to create (e.g., `appdb`)    |
| `POSTGRES_PORT`     | Only if conflicting | Change `5432` only if another service uses that port  |
| `PGADMIN_EMAIL`     | Yes                 | Your email address — used to log in to pgAdmin        |
| `PGADMIN_PASSWORD`  | **Yes — critical**  | A strong, unique password for pgAdmin                 |
| `PGADMIN_PORT`      | Only if conflicting | Change `5050` only if another service uses that port  |

**Step 4.** Start the containers in detached mode:

```bash
docker compose up -d
```

**Step 5.** Verify that both containers are running and healthy:

```bash
docker compose ps
```

Expected output: both `dev-postgres` and `dev-pgadmin` should show `running (healthy)` or `running`.

---

## Running — Dev Mode

All commands below must be run from inside the `PostgreSQL` directory.

**Start all services:**
```bash
docker compose up -d
```

**Stop all services (data is preserved in the volume):**
```bash
docker compose down
```

**Restart all services:**
```bash
docker compose restart
```

**Follow live logs for all services:**
```bash
docker compose logs -f
```

**Follow live logs for a specific service:**
```bash
docker compose logs -f postgres
docker compose logs -f pgadmin
```

**Open an interactive psql shell inside the container:**
```bash
docker exec -it dev-postgres psql -U <POSTGRES_USER> -d <POSTGRES_DB>
```
> Replace `<POSTGRES_USER>` and `<POSTGRES_DB>` with the actual values from your `.env` file.

**Check container status and health:**
```bash
docker compose ps
```

---

## Running — Prod Mode

Production mode applies the `docker-compose.prod.yaml` override on top of the base compose file. It switches to the Alpine image, disables pgAdmin, removes host port exposure, enables performance tuning, and enforces resource limits.

> **Before running in production**, ensure you have completed the [Security Checklist](../README.md#security-checklist) in the root README.

**Start in production mode:**
```bash
docker compose -f docker-compose.yaml -f docker-compose.prod.yaml up -d
```

**Stop production containers:**
```bash
docker compose -f docker-compose.yaml -f docker-compose.prod.yaml down
```

**Follow production logs:**
```bash
docker compose -f docker-compose.yaml -f docker-compose.prod.yaml logs -f postgres
```

**What changes in production:**

| Feature           | Dev                          | Prod                             |
|-------------------|------------------------------|----------------------------------|
| Image             | `postgres:17`                | `postgres:17-alpine`             |
| Container name    | `dev-postgres`               | `prod-postgres`                  |
| pgAdmin           | Running on `PGADMIN_PORT`    | Disabled (profile: `dev-only`)   |
| Host port         | Exposed via `POSTGRES_PORT`  | Not exposed to host              |
| Network           | Default bridge               | Isolated `postgres_net` bridge   |
| RAM limit         | Unlimited                    | 512 MB                           |
| CPU limit         | Unlimited                    | 0.5 vCPU                         |
| `max_connections` | Default (100)                | Explicitly set to 100            |
| `shared_buffers`  | Default (128 MB)             | 256 MB                           |

**Verify the production container is healthy:**
```bash
docker inspect --format='{{.State.Health.Status}}' prod-postgres
```
Expected output: `healthy`

---

## Accessing pgAdmin (Dev Only)

Open your browser and navigate to:

```
http://localhost:5050
```
> Replace `5050` with the value of `PGADMIN_PORT` in your `.env` file if you changed it.

**Login credentials:**
- **Email**: the value of `PGADMIN_EMAIL` in your `.env`
- **Password**: the value of `PGADMIN_PASSWORD` in your `.env`

**Connect pgAdmin to the PostgreSQL server (first-time setup):**

1. After logging in, right-click **Servers** in the left panel and select **Register → Server**
2. In the **General** tab, set a name (e.g., `Dev PostgreSQL`)
3. In the **Connection** tab, enter:
   - **Host**: `postgres` — use the Docker service name, **not** `localhost`
   - **Port**: `5432`
   - **Maintenance database**: the value of `POSTGRES_DB`
   - **Username**: the value of `POSTGRES_USER`
   - **Password**: the value of `POSTGRES_PASSWORD`
4. Click **Save**

You should now see your database listed under the server in the left panel.

---

## Environment Variables

| Variable            | Description                                              | Required | Example Value        |
|---------------------|----------------------------------------------------------|----------|----------------------|
| `POSTGRES_USER`     | PostgreSQL superuser username created on first start     | Yes      | `appuser`            |
| `POSTGRES_PASSWORD` | Password for the PostgreSQL superuser                    | Yes      | `S3cur3P@ssw0rd!`    |
| `POSTGRES_DB`       | Default database name created on first start             | Yes      | `appdb`              |
| `POSTGRES_PORT`     | Host port mapped to the container's internal port 5432   | Yes      | `5432`               |
| `PGADMIN_EMAIL`     | Email address used to log in to the pgAdmin web UI       | Yes      | `admin@example.com`  |
| `PGADMIN_PASSWORD`  | Password used to log in to the pgAdmin web UI            | Yes      | `AdminP@ss!`         |
| `PGADMIN_PORT`      | Host port mapped to the pgAdmin container's port 80      | Yes      | `5050`               |

---

## Healthcheck

The PostgreSQL container is configured with an automated healthcheck that runs every 10 seconds:

```yaml
test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
interval: 10s
timeout: 5s
retries: 5
start_period: 30s
```

- `pg_isready` verifies that PostgreSQL is accepting connections for the given user and database.
- `start_period: 30s` grants PostgreSQL 30 seconds to finish initializing before health failures begin counting.
- The container is marked `unhealthy` only after 5 consecutive failures.
- `pgadmin` uses `depends_on: condition: service_healthy` and starts only after PostgreSQL passes its healthcheck.

**Check health status manually:**
```bash
docker inspect --format='{{.State.Health.Status}}' dev-postgres
```

**View full healthcheck history (requires `jq`):**
```bash
docker inspect --format='{{json .State.Health}}' dev-postgres | jq
```

---

## Persistence & Volumes

Data is persisted in a named Docker volume so it survives container restarts and removals.

| Volume Name     | Mount Point (inside container) |
|-----------------|--------------------------------|
| `postgres_data` | `/var/lib/postgresql/data`     |

**Backup the volume to a compressed archive:**
```bash
docker run --rm \
  -v postgres_data:/data \
  -v "$(pwd)":/backup \
  alpine \
  tar czf /backup/postgres_backup_$(date +%Y%m%d_%H%M%S).tar.gz -C /data .
```

**Restore from a backup archive:**
```bash
# WARNING: This overwrites ALL existing data in the volume.
docker run --rm \
  -v postgres_data:/data \
  -v "$(pwd)":/backup \
  alpine \
  sh -c "rm -rf /data/* && tar xzf /backup/<your_backup_file>.tar.gz -C /data"
```

**Export a database to a SQL file (logical backup):**
```bash
docker exec dev-postgres pg_dump -U <POSTGRES_USER> <POSTGRES_DB> > backup.sql
```

**Import a SQL file into the database:**
```bash
docker exec -i dev-postgres psql -U <POSTGRES_USER> -d <POSTGRES_DB> < backup.sql
```

**Remove the volume permanently (deletes all data):**
```bash
docker compose down -v
```

---

## Performance Tuning (Prod)

The production override adds two PostgreSQL configuration parameters via command-line flags:

| Flag              | Value   | Reason                                                                                        |
|-------------------|---------|-----------------------------------------------------------------------------------------------|
| `max_connections` | `100`   | Limits simultaneous client connections. Prevents memory exhaustion under load.                |
| `shared_buffers`  | `256MB` | Memory pool PostgreSQL uses for caching data pages. Set to ~25% of available RAM.            |

**When to adjust:**

- **`max_connections` above 100**: Each connection consumes RAM (~5–10 MB). Before raising this value, consider deploying [PgBouncer](https://www.pgbouncer.org/) as a connection pooler in front of PostgreSQL instead.
- **`shared_buffers` for larger servers**: Use 25% of total RAM. For a 4 GB server, use `1GB`. After changing this, also update `deploy.resources.limits.memory` accordingly.

---

## Troubleshooting

**Problem**: Container fails to start with `Error starting userland proxy: bind: address already in use`
**Cause**: Another process is already using port 5432 on the host.
**Solution**: Change `POSTGRES_PORT` in your `.env` to an unused port (e.g., `5433`), then run `docker compose up -d` again.

---

**Problem**: Container is stuck in `starting` health state for more than 60 seconds.
**Cause**: PostgreSQL is still initializing its data directory on first run, or the database process failed.
**Solution**: Check the logs:
```bash
docker compose logs postgres
```

---

**Problem**: `FATAL: password authentication failed for user "..."` when connecting from an application.
**Cause**: The credentials in `.env` do not match what was stored in the volume on first run.
**Solution**: Either use the original credentials, or reset the database completely:
```bash
docker compose down -v && docker compose up -d
```
> **Warning**: `down -v` permanently deletes all data. Back up first if needed.

---

**Problem**: pgAdmin shows `Unable to connect to server` after clicking Save.
**Cause**: The Host field was set to `localhost` or `127.0.0.1` instead of the Docker service name.
**Solution**: In pgAdmin's connection settings, set **Host** to `postgres` (the service name in `docker-compose.yaml`).

---

**Problem**: Application container cannot connect to `dev-postgres`.
**Cause**: The application container is not on the same Docker network as the `postgres` service.
**Solution**: In your application's `docker-compose.yaml`, expose `POSTGRES_PORT` to the host and connect via `localhost:<port>`, or attach the app container to the `postgresql_default` network.

---

**Problem**: pgAdmin container starts but the web page refuses to connect or shows a blank screen.
**Cause**: pgAdmin takes 10–20 seconds to initialize after the container starts.
**Solution**: Wait 20–30 seconds and refresh the browser. If the issue persists:
```bash
docker compose logs pgadmin
```

---

## Useful Commands

| Command                                                                  | Description                                           |
|--------------------------------------------------------------------------|-------------------------------------------------------|
| `docker compose up -d`                                                   | Start all services in the background                  |
| `docker compose down`                                                    | Stop and remove containers (volume preserved)         |
| `docker compose down -v`                                                 | Stop, remove containers **and** volume (data deleted) |
| `docker compose ps`                                                      | Show running containers and health status             |
| `docker compose logs -f postgres`                                        | Follow live logs from the postgres container          |
| `docker exec -it dev-postgres psql -U <user> -d <db>`                   | Open interactive psql shell in the container          |
| `docker exec dev-postgres pg_dump -U <user> <db> > backup.sql`          | Export database to a SQL file on the host             |
| `docker exec -i dev-postgres psql -U <user> -d <db> < backup.sql`       | Import a SQL file into the database                   |
| `docker inspect --format='{{.State.Health.Status}}' dev-postgres`        | Check container health status                         |
| `docker stats dev-postgres`                                              | View live CPU, memory, and network usage              |
