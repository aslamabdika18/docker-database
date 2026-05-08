# Redis — Docker Setup

Redis 7 containerized for local development and production deployment, with RedisInsight as the web-based GUI administration tool for development environments.

Redis is an open-source, in-memory data structure store used as a database, cache, message broker, and streaming engine. It is well-suited for session storage, real-time leaderboards, rate limiting, pub/sub messaging, and any workload that requires sub-millisecond response times. Redis persists data to disk via append-only file (AOF), ensuring durability even when running in-memory.

---

## Services Included

| Service        | Image                     | Port                    | Description                                       |
|----------------|---------------------------|-------------------------|---------------------------------------------------|
| `redis`        | `redis:7-alpine`          | `${REDIS_PORT}`         | Redis 7 server (dev & prod)                       |
| `redisinsight` | `redis/redisinsight:latest` | `${REDISINSIGHT_PORT}` | Web-based Redis management UI (dev only)          |

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

**Step 1.** Navigate to the `Redis` directory from the repo root:

```bash
cd Redis
```

**Step 2.** Copy the example environment file:

```bash
# Linux / macOS
cp .env.example .env

# Windows (PowerShell)
Copy-Item .env.example .env
```

**Step 3.** Open `.env` in a text editor and replace every placeholder value:

| Variable            | Required to Change  | What to set                                                       |
|---------------------|---------------------|-------------------------------------------------------------------|
| `REDIS_PASSWORD`    | **Yes — critical**  | A strong, unique password — minimum 16 characters                 |
| `REDIS_PORT`        | Only if conflicting | Change `6379` only if another service uses that port              |
| `REDIS_MAX_MEMORY`  | Yes                 | Max memory Redis may use (e.g., `128mb`, `512mb`). Must be lower than the prod container limit of `256m`. |
| `REDISINSIGHT_PORT` | Only if conflicting | Change `5540` only if another service uses that port              |

**Step 4.** Start the containers in detached mode:

```bash
docker compose up -d
```

**Step 5.** Verify that both containers are running and healthy:

```bash
docker compose ps
```

Expected output: `dev-redis` should show `running (healthy)` and `dev-redisinsight` should show `running`.

---

## Running — Dev Mode

All commands below must be run from inside the `Redis` directory.

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
docker compose logs -f redis
docker compose logs -f redisinsight
```

**Open an interactive redis-cli shell inside the container:**
```bash
docker exec -it dev-redis redis-cli -a <REDIS_PASSWORD>
```
> Replace `<REDIS_PASSWORD>` with the value from your `.env` file. Once inside, type `PING` and expect `PONG` to confirm the connection.

**Check container status and health:**
```bash
docker compose ps
```

---

## Running — Prod Mode

Production mode applies the `docker-compose.prod.yaml` override on top of the base compose file. It disables RedisInsight, removes host port exposure, and enforces strict resource limits.

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
docker compose -f docker-compose.yaml -f docker-compose.prod.yaml logs -f redis
```

**What changes in production:**

| Feature              | Dev                           | Prod                             |
|----------------------|-------------------------------|----------------------------------|
| Image                | `redis:7-alpine`              | `redis:7-alpine`                 |
| Container name       | `dev-redis`                   | `prod-redis`                     |
| RedisInsight         | Running on `REDISINSIGHT_PORT` | Disabled (profile: `dev-only`)  |
| Host port            | Exposed via `REDIS_PORT`      | Not exposed to host              |
| Network              | Default bridge                | Isolated `redis_net` bridge      |
| RAM limit            | Unlimited                     | 256 MB                           |
| CPU limit            | Unlimited                     | 0.25 vCPU                        |
| Security config      | Same as dev                   | Same as dev (set in base file)   |

> All security-related configuration (`requirepass`, `rename-command`, `maxmemory`, etc.) is defined in the base `docker-compose.yaml` and applies to both dev and prod modes.

**Verify the production container is healthy:**
```bash
docker inspect --format='{{.State.Health.Status}}' prod-redis
```
Expected output: `healthy`

---

## Accessing RedisInsight (Dev Only)

Open your browser and navigate to:

```
http://localhost:5540
```
> Replace `5540` with the value of `REDISINSIGHT_PORT` in your `.env` file if you changed it.

RedisInsight does not require login credentials on first access. On the welcome screen:

1. Click **Add Redis Database**
2. Fill in the connection form:
   - **Host**: `redis` — use the Docker service name, **not** `localhost`
   - **Port**: `6379`
   - **Name**: any label (e.g., `Dev Redis`)
   - **Password**: the value of `REDIS_PASSWORD` in your `.env`
3. Click **Add Redis Database**

You can now browse keys, run CLI commands, inspect memory usage, and view slow log entries from the UI.

---

## Environment Variables

| Variable            | Description                                                           | Required | Example Value   |
|---------------------|-----------------------------------------------------------------------|----------|-----------------|
| `REDIS_PASSWORD`    | Password required by all clients connecting to Redis (`requirepass`)  | Yes      | `S3cur3P@ss!`   |
| `REDIS_PORT`        | Host port mapped to the container's internal port 6379                | Yes      | `6379`          |
| `REDIS_MAX_MEMORY`  | Maximum memory Redis may use before eviction policy activates         | Yes      | `128mb`         |
| `REDISINSIGHT_PORT` | Host port mapped to the RedisInsight container's port 5540            | Yes      | `5540`          |

---

## Healthcheck

The Redis container is configured with an automated healthcheck that runs every 10 seconds:

```yaml
test: ["CMD-SHELL", "redis-cli --no-auth-warning -a $$REDIS_PASSWORD ping | grep -q PONG"]
interval: 10s
timeout: 5s
retries: 5
start_period: 10s
```

- `redis-cli ping` sends a `PING` command to the server; a healthy server responds with `PONG`.
- `-a $$REDIS_PASSWORD` passes the password for authentication. The `$$` syntax causes Docker Compose to write a literal `$` into the command, which the container's shell then expands to the environment variable value.
- `--no-auth-warning` suppresses the CLI warning about passing passwords on the command line, keeping healthcheck logs clean.
- `start_period: 10s` is short because Redis starts almost instantly.
- `redisinsight` uses `depends_on: condition: service_healthy` and starts only after Redis passes its healthcheck.

**Check health status manually:**
```bash
docker inspect --format='{{.State.Health.Status}}' dev-redis
```

**View full healthcheck history (requires `jq`):**
```bash
docker inspect --format='{{json .State.Health}}' dev-redis | jq
```

---

## Persistence & Volumes

Data is persisted via the Append-Only File (AOF) mechanism (`--appendonly yes`) and stored in a named Docker volume.

| Volume Name  | Mount Point (inside container) |
|--------------|--------------------------------|
| `redis_data` | `/data`                        |

With AOF enabled, every write operation is appended to `/data/appendonly.aof`. On restart, Redis replays this file to restore the dataset. This provides much stronger durability guarantees than the default RDB snapshot-only mode.

**Backup the volume to a compressed archive:**
```bash
docker run --rm \
  -v redis_data:/data \
  -v "$(pwd)":/backup \
  alpine \
  tar czf /backup/redis_backup_$(date +%Y%m%d_%H%M%S).tar.gz -C /data .
```

**Restore from a backup archive:**
```bash
# WARNING: This overwrites ALL existing data in the volume.
docker run --rm \
  -v redis_data:/data \
  -v "$(pwd)":/backup \
  alpine \
  sh -c "rm -rf /data/* && tar xzf /backup/<your_backup_file>.tar.gz -C /data"
```

**Trigger a manual AOF rewrite to compact the file:**
```bash
docker exec dev-redis redis-cli --no-auth-warning -a <REDIS_PASSWORD> BGREWRITEAOF
```

**Remove the volume permanently (deletes all data):**
```bash
docker compose down -v
```

---

## Performance Tuning (Prod)

All Redis configuration is applied via the `command` directive in the base `docker-compose.yaml` and is active in both dev and prod modes. The prod override adds only resource limits on top.

| Option                        | Value           | Reason                                                                                                  |
|-------------------------------|-----------------|---------------------------------------------------------------------------------------------------------|
| `--requirepass`               | `$REDIS_PASSWORD` | Enforces password authentication. Without this, Redis accepts all connections with no credentials.    |
| `--appendonly yes`            | —               | Enables AOF persistence. Data survives container restarts. Without this, all data is lost on restart.  |
| `--maxmemory`                 | `$REDIS_MAX_MEMORY` | Caps memory usage. When the limit is reached, the eviction policy activates to free space.         |
| `--maxmemory-policy allkeys-lru` | —            | When full, evicts the least-recently-used key across all keys. Appropriate for general-purpose caches. |
| `--timeout 300`               | `300` seconds   | Closes idle client connections after 5 minutes, freeing connection slots and memory.                   |
| `--tcp-keepalive 60`          | `60` seconds    | Sends TCP keepalive probes every 60 seconds to detect dead clients and close stale connections.         |
| `--rename-command FLUSHALL ""` | —              | Disables the `FLUSHALL` command, preventing accidental or malicious deletion of all databases.          |
| `--rename-command FLUSHDB ""`  | —              | Disables the `FLUSHDB` command, preventing accidental deletion of the current database.                 |
| `--rename-command DEBUG ""`    | —              | Disables the `DEBUG` command, which can be used to crash the server or inspect memory.                  |

**When to adjust:**

- **`--maxmemory-policy`**: `allkeys-lru` is optimal for pure cache use cases. For mixed workloads where some keys must never be evicted, use `volatile-lru` (evicts only keys with a TTL set) instead.
- **`REDIS_MAX_MEMORY`**: Must always be lower than the container's memory limit (`256m` in prod). A safe formula: `REDIS_MAX_MEMORY = container_limit * 0.75`. For the 256 MB prod limit, `192mb` is a safer upper bound than `256mb`.

---

## Troubleshooting

**Problem**: Container fails to start with `Error starting userland proxy: bind: address already in use`
**Cause**: Another process (or another Redis instance) is already using port 6379 on the host.
**Solution**: Change `REDIS_PORT` in your `.env` to an unused port (e.g., `6380`), then run `docker compose up -d` again.

---

**Problem**: Healthcheck fails — container stays `unhealthy`.
**Cause**: `REDIS_PASSWORD` in `.env` is empty or incorrect, so `redis-cli -a` fails authentication.
**Solution**: Ensure `REDIS_PASSWORD` is set to a non-empty value in `.env`. Check logs for the exact error:
```bash
docker compose logs redis
```

---

**Problem**: `WRONGPASS invalid username-password pair` when connecting from an application.
**Cause**: The application is using a different or missing password.
**Solution**: Verify that the application's Redis connection string includes the correct password matching `REDIS_PASSWORD` in `.env`. For Redis URL format: `redis://:<REDIS_PASSWORD>@localhost:<REDIS_PORT>`.

---

**Problem**: Redis exits with `OOM command not allowed when used memory > 'maxmemory'`.
**Cause**: Redis has reached the `REDIS_MAX_MEMORY` limit and cannot accept new write commands because the eviction policy could not free enough space.
**Solution**: Either increase `REDIS_MAX_MEMORY` in `.env`, or review the application's key expiry strategy to ensure keys are not stored indefinitely.

---

**Problem**: `ERR unknown command 'FLUSHALL'` or `ERR unknown command 'FLUSHDB'` when running a command.
**Cause**: These commands have been intentionally disabled via `--rename-command` for security. This is expected behavior.
**Solution**: This is by design. If you need to clear the database in development, connect via `redis-cli` and use `DEL` on specific keys, or remove the `--rename-command` lines from `docker-compose.yaml` temporarily (dev only).

---

**Problem**: Data is lost after restarting the container even though AOF is enabled.
**Cause**: The container was started without the `redis_data` volume being mounted, or the volume was deleted with `docker compose down -v`.
**Solution**: Always use `docker compose down` (without `-v`) to stop containers. Verify the volume exists:
```bash
docker volume ls | grep redis_data
```

---

## Useful Commands

| Command                                                                        | Description                                           |
|--------------------------------------------------------------------------------|-------------------------------------------------------|
| `docker compose up -d`                                                         | Start all services in the background                  |
| `docker compose down`                                                          | Stop and remove containers (volume preserved)         |
| `docker compose down -v`                                                       | Stop, remove containers **and** volume (data deleted) |
| `docker compose ps`                                                            | Show running containers and health status             |
| `docker compose logs -f redis`                                                 | Follow live logs from the redis container             |
| `docker exec -it dev-redis redis-cli -a <password>`                            | Open interactive redis-cli shell                      |
| `docker exec dev-redis redis-cli --no-auth-warning -a <pass> INFO server`     | Display Redis server info and stats                   |
| `docker exec dev-redis redis-cli --no-auth-warning -a <pass> INFO memory`     | Display memory usage statistics                       |
| `docker exec dev-redis redis-cli --no-auth-warning -a <pass> DBSIZE`          | Show total number of keys in the database             |
| `docker inspect --format='{{.State.Health.Status}}' dev-redis`                 | Check container health status                         |
