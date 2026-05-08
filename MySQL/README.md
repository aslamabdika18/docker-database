# MySQL — Docker Setup

MySQL 8.4 containerized for local development and production deployment, with phpMyAdmin as the web-based GUI administration tool for development environments.

MySQL is the world's most popular open-source relational database management system. It is well-suited for web applications, e-commerce platforms, and any workload that benefits from a mature, widely-supported SQL engine with a large ecosystem of tools, drivers, and hosting providers.

---

## Services Included

| Service      | Image                          | Port               | Description                                      |
|--------------|--------------------------------|--------------------|--------------------------------------------------|
| `mysql`      | `mysql:8.4`                    | `${MYSQL_PORT}`    | MySQL 8.4 database server (dev & prod)           |
| `phpmyadmin` | `phpmyadmin/phpmyadmin:latest` | `${PHPMYADMIN_PORT}` | Web-based database administration UI (dev only) |

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

**Step 1.** Navigate to the `MySQL` directory from the repo root:

```bash
cd MySQL
```

**Step 2.** Copy the example environment file:

```bash
# Linux / macOS
cp .env.example .env

# Windows (PowerShell)
Copy-Item .env.example .env
```

**Step 3.** Open `.env` in a text editor and replace every placeholder value:

| Variable             | Required to Change  | What to set                                           |
|----------------------|---------------------|-------------------------------------------------------|
| `MYSQL_ROOT_PASSWORD`| **Yes — critical**  | A strong, unique password for the MySQL root account  |
| `MYSQL_USER`         | Yes                 | Any alphanumeric username for the app database user   |
| `MYSQL_PASSWORD`     | **Yes — critical**  | A strong, unique password for the app database user   |
| `MYSQL_DATABASE`     | Yes                 | The name of the database to create (e.g., `appdb`)    |
| `MYSQL_PORT`         | Only if conflicting | Change `3306` only if another service uses that port  |
| `PHPMYADMIN_PORT`    | Only if conflicting | Change `8080` only if another service uses that port  |

**Step 4.** Start the containers in detached mode:

```bash
docker compose up -d
```

**Step 5.** Verify that both containers are running and healthy:

```bash
docker compose ps
```

Expected output: both `dev-mysql` and `dev-phpmyadmin` should show `running (healthy)` or `running`.

---

## Running — Dev Mode

All commands below must be run from inside the `MySQL` directory.

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
docker compose logs -f mysql
docker compose logs -f phpmyadmin
```

**Open an interactive MySQL shell inside the container:**
```bash
docker exec -it dev-mysql mysql -u <MYSQL_USER> -p <MYSQL_DATABASE>
```
> You will be prompted for the password. Replace `<MYSQL_USER>` and `<MYSQL_DATABASE>` with values from your `.env` file.

**Open a root MySQL shell:**
```bash
docker exec -it dev-mysql mysql -u root -p
```

**Check container status and health:**
```bash
docker compose ps
```

---

## Running — Prod Mode

Production mode applies the `docker-compose.prod.yaml` override on top of the base compose file. It disables phpMyAdmin, removes host port exposure, adds InnoDB performance tuning, and enforces resource limits.

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
docker compose -f docker-compose.yaml -f docker-compose.prod.yaml logs -f mysql
```

**What changes in production:**

| Feature                   | Dev                            | Prod                             |
|---------------------------|--------------------------------|----------------------------------|
| Image                     | `mysql:8.4`                    | `mysql:8.4`                      |
| Container name            | `dev-mysql`                    | `prod-mysql`                     |
| phpMyAdmin                | Running on `PHPMYADMIN_PORT`   | Disabled (profile: `dev-only`)   |
| Host port                 | Exposed via `MYSQL_PORT`       | Not exposed to host              |
| Network                   | Default bridge                 | Isolated `mysql_net` bridge      |
| RAM limit                 | Unlimited                      | 512 MB                           |
| CPU limit                 | Unlimited                      | 0.5 vCPU                         |
| `innodb_buffer_pool_size` | Default (128 MB)               | 256 MB                           |
| `max_connections`         | Default (151)                  | 100                              |
| `general_log`             | Default (OFF)                  | Explicitly OFF                   |

**Verify the production container is healthy:**
```bash
docker inspect --format='{{.State.Health.Status}}' prod-mysql
```
Expected output: `healthy`

---

## Accessing phpMyAdmin (Dev Only)

Open your browser and navigate to:

```
http://localhost:8080
```
> Replace `8080` with the value of `PHPMYADMIN_PORT` in your `.env` file if you changed it.

**Login credentials:**
- **Username**: `root`
- **Password**: the value of `MYSQL_ROOT_PASSWORD` in your `.env`

phpMyAdmin is pre-configured to connect to the `mysql` service automatically via the `PMA_HOST` environment variable. You do not need to enter a server address — just enter the username and password on the login screen.

To manage a specific database as the app user instead of root, use:
- **Username**: the value of `MYSQL_USER`
- **Password**: the value of `MYSQL_PASSWORD`

---

## Environment Variables

| Variable              | Description                                              | Required | Example Value          |
|-----------------------|----------------------------------------------------------|----------|------------------------|
| `MYSQL_ROOT_PASSWORD` | Password for the MySQL `root` superuser                  | Yes      | `R00tP@ssw0rd!`        |
| `MYSQL_USER`          | Application database username created on first start     | Yes      | `appuser`              |
| `MYSQL_PASSWORD`      | Password for the application database user               | Yes      | `S3cur3P@ssw0rd!`      |
| `MYSQL_DATABASE`      | Default database name created on first start             | Yes      | `appdb`                |
| `MYSQL_PORT`          | Host port mapped to the container's internal port 3306   | Yes      | `3306`                 |
| `PHPMYADMIN_PORT`     | Host port mapped to the phpMyAdmin container's port 80   | Yes      | `8080`                 |

---

## Healthcheck

The MySQL container is configured with an automated healthcheck that runs every 10 seconds:

```yaml
test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "--silent"]
interval: 10s
timeout: 5s
retries: 5
start_period: 30s
```

- `mysqladmin ping` checks whether the MySQL server process is alive and accepting connections.
- `--silent` suppresses output so only the exit code is used to determine health.
- `start_period: 30s` grants MySQL 30 seconds to finish its initialization sequence before failures begin counting.
- The container is marked `unhealthy` only after 5 consecutive failures.
- `phpmyadmin` uses `depends_on: condition: service_healthy` and starts only after MySQL passes its healthcheck.

**Check health status manually:**
```bash
docker inspect --format='{{.State.Health.Status}}' dev-mysql
```

**View full healthcheck history (requires `jq`):**
```bash
docker inspect --format='{{json .State.Health}}' dev-mysql | jq
```

---

## Persistence & Volumes

Data is persisted in a named Docker volume so it survives container restarts and removals.

| Volume Name  | Mount Point (inside container) |
|--------------|--------------------------------|
| `mysql_data` | `/var/lib/mysql`               |

**Backup the volume to a compressed archive:**
```bash
docker run --rm \
  -v mysql_data:/data \
  -v "$(pwd)":/backup \
  alpine \
  tar czf /backup/mysql_backup_$(date +%Y%m%d_%H%M%S).tar.gz -C /data .
```

**Restore from a backup archive:**
```bash
# WARNING: This overwrites ALL existing data in the volume.
docker run --rm \
  -v mysql_data:/data \
  -v "$(pwd)":/backup \
  alpine \
  sh -c "rm -rf /data/* && tar xzf /backup/<your_backup_file>.tar.gz -C /data"
```

**Export a database to a SQL file (logical backup):**
```bash
docker exec dev-mysql mysqldump -u root -p<MYSQL_ROOT_PASSWORD> <MYSQL_DATABASE> > backup.sql
```
> Replace `<MYSQL_ROOT_PASSWORD>` and `<MYSQL_DATABASE>` with values from your `.env` file. Note: no space between `-p` and the password.

**Import a SQL file into the database:**
```bash
docker exec -i dev-mysql mysql -u root -p<MYSQL_ROOT_PASSWORD> <MYSQL_DATABASE> < backup.sql
```

**Remove the volume permanently (deletes all data):**
```bash
docker compose down -v
```

---

## Performance Tuning (Prod)

The production override adds three MySQL server flags via the `command` directive:

| Flag                      | Value  | Reason                                                                                           |
|---------------------------|--------|--------------------------------------------------------------------------------------------------|
| `innodb-buffer-pool-size` | `256M` | Memory pool InnoDB uses for caching table and index data. The single most impactful MySQL setting. Set to ~50–70% of available RAM for a dedicated DB server. |
| `max-connections`         | `100`  | Limits simultaneous client connections. Prevents memory exhaustion; each connection uses ~1 MB.  |
| `general-log`             | `0`    | Disables the general query log, which writes every SQL statement to disk and degrades performance. |

**When to adjust:**

- **`innodb-buffer-pool-size`**: Increase this if your working dataset is larger than 256 MB and the server has more RAM. For a server with 2 GB RAM dedicated to MySQL, set this to `1G`. Also increase `deploy.resources.limits.memory` proportionally.
- **`max-connections`**: Each connection consumes ~1 MB of RAM. The formula is: `max_connections = available_RAM / memory_per_connection`. For 512 MB with other overhead, 100 is a safe default. Use a connection pool (e.g., ProxySQL) for higher concurrency.
- **`general-log`**: Set to `1` temporarily for debugging query issues in a staging environment. Never enable in production for extended periods.

---

## Troubleshooting

**Problem**: Container fails to start with `Error starting userland proxy: bind: address already in use`
**Cause**: Another process (or another MySQL instance) is already using port 3306 on the host.
**Solution**: Change `MYSQL_PORT` in your `.env` to an unused port (e.g., `3307`), then run `docker compose up -d` again.

---

**Problem**: Container is stuck in `starting` health state for more than 60 seconds.
**Cause**: MySQL is initializing its data directory on first run, which can take longer on slow disks. On subsequent runs, this indicates the server failed to start.
**Solution**: Check the logs for errors:
```bash
docker compose logs mysql
```
Common causes include incorrect permissions on the volume or conflicting `MYSQL_ROOT_PASSWORD`.

---

**Problem**: `Access denied for user 'root'@'localhost'` when connecting.
**Cause**: The `MYSQL_ROOT_PASSWORD` in `.env` does not match the password stored in the volume from a previous run.
**Solution**: Either use the original password, or reset the database completely:
```bash
docker compose down -v && docker compose up -d
```
> **Warning**: `down -v` permanently deletes all data. Back up first if needed.

---

**Problem**: phpMyAdmin shows `mysqli_real_connect(): (HY000/2002): Connection refused`.
**Cause**: phpMyAdmin started before MySQL was fully ready, or MySQL is unhealthy.
**Solution**: Wait for MySQL to pass its healthcheck, then restart phpMyAdmin:
```bash
docker compose restart phpmyadmin
```

---

**Problem**: `ERROR 1045 (28000): Access denied` when importing a SQL file via `mysqldump`.
**Cause**: Wrong credentials were passed to the `mysqldump` command, or the user does not have sufficient privileges.
**Solution**: Use the `root` user with `MYSQL_ROOT_PASSWORD` for full access, ensuring no space between `-p` and the password value.

---

**Problem**: MySQL exits immediately with `[ERROR] --initialize specified but the data directory has files in it`.
**Cause**: The `mysql_data` volume already contains data from a previous run, and MySQL is trying to re-initialize.
**Solution**: This typically indicates a version mismatch or corrupted data. Check logs, and if recovery is not possible:
```bash
docker compose down -v && docker compose up -d
```

---

## Useful Commands

| Command                                                                    | Description                                           |
|----------------------------------------------------------------------------|-------------------------------------------------------|
| `docker compose up -d`                                                     | Start all services in the background                  |
| `docker compose down`                                                      | Stop and remove containers (volume preserved)         |
| `docker compose down -v`                                                   | Stop, remove containers **and** volume (data deleted) |
| `docker compose ps`                                                        | Show running containers and health status             |
| `docker compose logs -f mysql`                                             | Follow live logs from the mysql container             |
| `docker exec -it dev-mysql mysql -u root -p`                               | Open interactive MySQL shell as root                  |
| `docker exec dev-mysql mysqldump -u root -p<pass> <db> > backup.sql`       | Export database to a SQL file on the host             |
| `docker exec -i dev-mysql mysql -u root -p<pass> <db> < backup.sql`        | Import a SQL file into the database                   |
| `docker inspect --format='{{.State.Health.Status}}' dev-mysql`             | Check container health status                         |
| `docker stats dev-mysql`                                                   | View live CPU, memory, and network usage              |
