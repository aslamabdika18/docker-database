# MongoDB — Docker Setup

MongoDB 7 containerized for local development and production deployment, with Mongo Express as the web-based GUI administration tool for development environments.

MongoDB is a document-oriented NoSQL database that stores data in flexible, JSON-like BSON documents. It is well-suited for applications with evolving data schemas, high write throughput, hierarchical or nested data structures, and workloads that benefit from horizontal scaling — such as real-time analytics, content management, and event-driven microservices.

---

## Services Included

| Service         | Image                   | Port                   | Description                                         |
|-----------------|-------------------------|------------------------|-----------------------------------------------------|
| `mongodb`       | `mongo:7`               | `${MONGO_PORT}`        | MongoDB 7 database server (dev & prod)              |
| `mongo-express` | `mongo-express:latest`  | `${MONGO_EXPRESS_PORT}` | Web-based database administration UI (dev only)    |

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

**Step 1.** Navigate to the `MongoDB` directory from the repo root:

```bash
cd MongoDB
```

**Step 2.** Copy the example environment file:

```bash
# Linux / macOS
cp .env.example .env

# Windows (PowerShell)
Copy-Item .env.example .env
```

**Step 3.** Open `.env` in a text editor and replace every placeholder value:

| Variable                    | Required to Change  | What to set                                               |
|-----------------------------|---------------------|-----------------------------------------------------------|
| `MONGO_INITDB_ROOT_USERNAME`| Yes                 | Admin username for MongoDB (e.g., `mongoadmin`)           |
| `MONGO_INITDB_ROOT_PASSWORD`| **Yes — critical**  | A strong, unique password — minimum 16 characters         |
| `MONGO_PORT`                | Only if conflicting | Change `27017` only if another service uses that port     |
| `ME_CONFIG_BASICAUTH_USERNAME` | Yes              | Username to log in to the Mongo Express web UI            |
| `ME_CONFIG_BASICAUTH_PASSWORD` | **Yes — critical** | A strong password for the Mongo Express web UI          |
| `MONGO_EXPRESS_PORT`        | Only if conflicting | Change `8081` only if another service uses that port      |

> **Note if upgrading from a previous version of this file**: the old compose used `MONGO_USER` and `MONGO_PASSWORD` variables. These have been replaced with `MONGO_INITDB_ROOT_USERNAME`, `MONGO_INITDB_ROOT_PASSWORD`, `ME_CONFIG_BASICAUTH_USERNAME`, and `ME_CONFIG_BASICAUTH_PASSWORD`. Update your existing `.env` accordingly.

**Step 4.** Start the containers in detached mode:

```bash
docker compose up -d
```

**Step 5.** Verify that both containers are running and healthy:

```bash
docker compose ps
```

Expected output: `dev-mongodb` should show `running (healthy)` and `dev-mongo-express` should show `running`.

---

## Running — Dev Mode

All commands below must be run from inside the `MongoDB` directory.

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
docker compose logs -f mongodb
docker compose logs -f mongo-express
```

**Open an interactive mongosh shell inside the container:**
```bash
docker exec -it dev-mongodb mongosh \
  --username <MONGO_INITDB_ROOT_USERNAME> \
  --password <MONGO_INITDB_ROOT_PASSWORD> \
  --authenticationDatabase admin
```
> Replace the angle-bracket placeholders with values from your `.env` file.

**Check container status and health:**
```bash
docker compose ps
```

---

## Running — Prod Mode

Production mode applies the `docker-compose.prod.yaml` override on top of the base compose file. It disables Mongo Express, removes host port exposure, adds WiredTiger cache tuning, enables quiet logging, and enforces resource limits.

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
docker compose -f docker-compose.yaml -f docker-compose.prod.yaml logs -f mongodb
```

**What changes in production:**

| Feature                | Dev                               | Prod                              |
|------------------------|-----------------------------------|-----------------------------------|
| Image                  | `mongo:7`                         | `mongo:7`                         |
| Container name         | `dev-mongodb`                     | `prod-mongodb`                    |
| Mongo Express          | Running on `MONGO_EXPRESS_PORT`   | Disabled (profile: `dev-only`)    |
| Host port              | Exposed via `MONGO_PORT`          | Not exposed to host               |
| Network                | Default bridge                    | Isolated `mongodb_net` bridge     |
| RAM limit              | Unlimited                         | 512 MB                            |
| CPU limit              | Unlimited                         | 0.5 vCPU                          |
| WiredTiger cache       | Default (~50% of RAM)             | Capped at 0.5 GB                  |
| Log verbosity          | Default (verbose)                 | Quiet mode                        |

**Verify the production container is healthy:**
```bash
docker inspect --format='{{.State.Health.Status}}' prod-mongodb
```
Expected output: `healthy`

---

## Accessing Mongo Express (Dev Only)

Open your browser and navigate to:

```
http://localhost:8081
```
> Replace `8081` with the value of `MONGO_EXPRESS_PORT` in your `.env` file if you changed it.

**Login credentials (HTTP Basic Auth prompt):**
- **Username**: the value of `ME_CONFIG_BASICAUTH_USERNAME` in your `.env`
- **Password**: the value of `ME_CONFIG_BASICAUTH_PASSWORD` in your `.env`

After logging in, you will see a list of all databases in the left sidebar. You can browse collections, run queries, create indexes, and import/export documents directly from the web UI.

> The Basic Auth credentials (`ME_CONFIG_BASICAUTH_*`) are separate from the MongoDB admin credentials (`MONGO_INITDB_ROOT_*`). The Basic Auth layer protects the Mongo Express web UI itself; Mongo Express uses the admin credentials internally to connect to MongoDB.

---

## Environment Variables

| Variable                     | Description                                                    | Required | Example Value          |
|------------------------------|----------------------------------------------------------------|----------|------------------------|
| `MONGO_INITDB_ROOT_USERNAME` | MongoDB root admin username created on first start             | Yes      | `mongoadmin`           |
| `MONGO_INITDB_ROOT_PASSWORD` | Password for the MongoDB root admin                            | Yes      | `S3cur3P@ssw0rd!`      |
| `MONGO_PORT`                 | Host port mapped to the container's internal port 27017        | Yes      | `27017`                |
| `ME_CONFIG_BASICAUTH_USERNAME` | Username for the Mongo Express web UI login prompt           | Yes      | `webadmin`             |
| `ME_CONFIG_BASICAUTH_PASSWORD` | Password for the Mongo Express web UI login prompt           | Yes      | `WebP@ss!`             |
| `MONGO_EXPRESS_PORT`         | Host port mapped to the Mongo Express container's port 8081    | Yes      | `8081`                 |

---

## Healthcheck

The MongoDB container is configured with an automated healthcheck that runs every 10 seconds:

```yaml
test: ["CMD-SHELL", "mongosh --username $$MONGO_INITDB_ROOT_USERNAME --password $$MONGO_INITDB_ROOT_PASSWORD --authenticationDatabase admin --eval 'db.adminCommand({ ping: 1 })' --quiet"]
interval: 10s
timeout: 5s
retries: 5
start_period: 30s
```

- `mongosh` connects to MongoDB using the root credentials and runs `db.adminCommand({ ping: 1 })`, which returns `{ ok: 1 }` when the server is healthy.
- Auth credentials are passed via `$$VAR` syntax — Docker Compose resolves `$$` to a single `$`, which the shell then expands to the environment variable value inside the container.
- `start_period: 30s` grants MongoDB 30 seconds to initialize its storage engine before failures begin counting.
- `mongo-express` uses `depends_on: condition: service_healthy` and starts only after MongoDB passes its healthcheck.

**Check health status manually:**
```bash
docker inspect --format='{{.State.Health.Status}}' dev-mongodb
```

**View full healthcheck history (requires `jq`):**
```bash
docker inspect --format='{{json .State.Health}}' dev-mongodb | jq
```

---

## Persistence & Volumes

Data is persisted in a named Docker volume so it survives container restarts and removals.

| Volume Name  | Mount Point (inside container) |
|--------------|--------------------------------|
| `mongo_data` | `/data/db`                     |

**Backup the volume to a compressed archive:**
```bash
docker run --rm \
  -v mongo_data:/data \
  -v "$(pwd)":/backup \
  alpine \
  tar czf /backup/mongo_backup_$(date +%Y%m%d_%H%M%S).tar.gz -C /data .
```

**Restore from a backup archive:**
```bash
# WARNING: This overwrites ALL existing data in the volume.
docker run --rm \
  -v mongo_data:/data \
  -v "$(pwd)":/backup \
  alpine \
  sh -c "rm -rf /data/* && tar xzf /backup/<your_backup_file>.tar.gz -C /data"
```

**Export a collection using mongodump (logical backup):**
```bash
docker exec dev-mongodb mongodump \
  --username <MONGO_INITDB_ROOT_USERNAME> \
  --password <MONGO_INITDB_ROOT_PASSWORD> \
  --authenticationDatabase admin \
  --out /tmp/dump

docker cp dev-mongodb:/tmp/dump ./mongodump_$(date +%Y%m%d_%H%M%S)
```

**Restore from a mongodump directory:**
```bash
docker cp ./mongodump_<timestamp> dev-mongodb:/tmp/dump

docker exec dev-mongodb mongorestore \
  --username <MONGO_INITDB_ROOT_USERNAME> \
  --password <MONGO_INITDB_ROOT_PASSWORD> \
  --authenticationDatabase admin \
  /tmp/dump
```

**Remove the volume permanently (deletes all data):**
```bash
docker compose down -v
```

---

## Performance Tuning (Prod)

The production override passes two flags to the `mongod` process:

| Flag                     | Value | Reason                                                                                                  |
|--------------------------|-------|---------------------------------------------------------------------------------------------------------|
| `--wiredTigerCacheSizeGB`| `0.5` | Caps the WiredTiger storage engine's in-memory cache at 512 MB. By default, MongoDB uses 50% of RAM minus 1 GB, which would exceed the container's 512 MB limit and cause the container to be killed by the OOM killer. |
| `--quiet`                | —     | Suppresses informational log messages, reducing log volume and I/O overhead in production.              |

**When to adjust:**

- **`--wiredTigerCacheSizeGB`**: This value must always be set to less than the container's memory limit (minus ~100 MB for other MongoDB overhead). If you increase the container memory limit to `1g`, you can raise this to `0.75` or `0.8`. As a rule: `wiredTigerCacheSizeGB = (container_memory_MB - 100) / 1024 * 0.9`.
- **`--quiet`**: Remove this flag in a staging environment if you need verbose logs for debugging. Never remove it in production unless actively troubleshooting.

---

## Troubleshooting

**Problem**: Container fails to start with `Error starting userland proxy: bind: address already in use`
**Cause**: Another process (or another MongoDB instance) is already using port 27017 on the host.
**Solution**: Change `MONGO_PORT` in your `.env` to an unused port (e.g., `27018`), then run `docker compose up -d` again.

---

**Problem**: Healthcheck fails — container stays `unhealthy` after 60 seconds.
**Cause**: MongoDB authentication is failing in the healthcheck command, or MongoDB has not finished initializing.
**Solution**: Verify the credentials in `.env` match the values used when the volume was first created. Check logs:
```bash
docker compose logs mongodb
```

---

**Problem**: Mongo Express shows `MongoServerError: Authentication failed`.
**Cause**: `ME_CONFIG_BASICAUTH_USERNAME` / `ME_CONFIG_BASICAUTH_PASSWORD` or the MongoDB admin credentials in `.env` are incorrect, or the container started before the database was fully ready.
**Solution**: Verify `.env` values are correct. If the issue persists, restart the Mongo Express container:
```bash
docker compose restart mongo-express
```

---

**Problem**: `Authentication failed` when running `mongosh` commands inside the container.
**Cause**: The `--authenticationDatabase admin` flag was omitted, or wrong credentials were used.
**Solution**: Always include `--authenticationDatabase admin` when connecting as the root user:
```bash
docker exec -it dev-mongodb mongosh --username <user> --password <pass> --authenticationDatabase admin
```

---

**Problem**: Production container is killed by the OS with `Out of Memory` errors.
**Cause**: `--wiredTigerCacheSizeGB` was not set (or set too high), so MongoDB's cache exceeded the container memory limit and triggered the OOM killer.
**Solution**: Ensure `--wiredTigerCacheSizeGB` in `docker-compose.prod.yaml` is set to a value lower than the container's memory limit. The value is in GB — `0.5` means 512 MB.

---

**Problem**: Data appears missing after restarting the container.
**Cause**: The container was removed with `docker compose down -v`, which deletes the named volume.
**Solution**: Always use `docker compose down` (without `-v`) to stop containers. The `-v` flag is destructive and removes all persisted data. Restore from a backup if needed.

---

## Useful Commands

| Command                                                                          | Description                                              |
|----------------------------------------------------------------------------------|----------------------------------------------------------|
| `docker compose up -d`                                                           | Start all services in the background                     |
| `docker compose down`                                                            | Stop and remove containers (volume preserved)            |
| `docker compose down -v`                                                         | Stop, remove containers **and** volume (data deleted)    |
| `docker compose ps`                                                              | Show running containers and health status                |
| `docker compose logs -f mongodb`                                                 | Follow live logs from the mongodb container              |
| `docker exec -it dev-mongodb mongosh -u <user> -p <pass> --authenticationDatabase admin` | Open interactive mongosh shell             |
| `docker exec dev-mongodb mongodump -u <user> -p <pass> --authenticationDatabase admin --out /tmp/dump` | Dump all databases        |
| `docker cp dev-mongodb:/tmp/dump ./local_dump`                                   | Copy dump from container to host                         |
| `docker inspect --format='{{.State.Health.Status}}' dev-mongodb`                 | Check container health status                            |
| `docker stats dev-mongodb`                                                       | View live CPU, memory, and network usage                 |
