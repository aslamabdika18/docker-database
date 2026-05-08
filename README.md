# docker-databases

> One-command Docker setup for PostgreSQL, MySQL, MongoDB, and Redis — with separate dev and production configurations.

![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white)
![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)
![Platform](https://img.shields.io/badge/platform-Linux%20%7C%20macOS%20%7C%20Windows-lightgrey)

---

## Overview

This repository provides ready-to-use Docker Compose configurations for the four most common databases used in modern application development. Each database lives in its own folder and can be started independently.

**Who this is for:**

- **Developers** who want a database running locally in under two minutes without installing anything natively.
- **DevOps engineers** who need a reproducible, production-grade baseline for containerized database deployments.
- **Teams** who want a shared, version-controlled database setup that works identically across all developer machines.

**Dev vs Prod philosophy:**

Each database folder contains two compose files. The base `docker-compose.yaml` is optimized for developer experience: ports are exposed to the host, a GUI administration tool is included, and there are no resource constraints. The `docker-compose.prod.yaml` is an override file applied on top of the base: it disables GUI tools, removes host port exposure, isolates the service in a dedicated network, and enforces CPU and memory limits. This separation means both modes share the same environment variable file and the same named volume, so switching between them never requires data migration.

---

## Available Services

| Database   | Version              | GUI Tool        | Dev Port | Prod Port    |
|------------|----------------------|-----------------|----------|--------------|
| PostgreSQL | 17 (dev), 17-alpine (prod) | pgAdmin 4  | 5432     | not exposed  |
| MySQL      | 8.4                  | phpMyAdmin      | 3306     | not exposed  |
| MongoDB    | 7                    | Mongo Express   | 27017    | not exposed  |
| Redis      | 7-alpine             | RedisInsight    | 6379     | not exposed  |

All GUI tools run in dev mode only and are accessible via a browser on the host machine. See each folder's README for the exact URL and login credentials.

---

## Prerequisites

Before you begin, verify that the following are installed on your machine:

- **Docker Engine 24.0 or higher**
  Verify: `docker --version`
  Install: https://docs.docker.com/engine/install/

- **Docker Compose v2.22.0 or higher**
  Verify: `docker compose version`
  Bundled with Docker Desktop. For Linux servers, install the Compose plugin separately: https://docs.docker.com/compose/install/

- **Supported operating systems**: Linux, macOS, Windows 10/11 (with WSL2 for Windows)

- **Minimum recommended RAM**: 4 GB total system memory (each database uses up to 512 MB in prod mode)

---

## Quick Start

The following steps start a single database. Repeat from step 2 for any additional database you need.

**1.** Clone this repository:

```bash
git clone https://github.com/aslamabdika18/docker-database.git
cd docker-database
```

**2.** Navigate to the folder for the database you want to run:

```bash
# Choose one:
cd PostgreSQL
cd MySQL
cd MongoDB
cd Redis
```

**3.** Copy the example environment file:

```bash
# Linux / macOS
cp .env.example .env

# Windows (PowerShell)
Copy-Item .env.example .env
```

**4.** Open `.env` in a text editor and replace all placeholder values with real ones. Every variable marked `YOUR_..._HERE` must be changed. See the [Environment Variables](#environment-variables) section or the folder's own README for details.

**5.** Start the containers:

```bash
docker compose up -d
```

**6.** Verify the containers are running and healthy:

```bash
docker compose ps
```

The database is ready when the status column shows `running (healthy)`. The GUI tool may take an additional 10–20 seconds to finish initializing.

---

## Dev vs Prod Mode

| Feature              | Dev Mode                         | Prod Mode                              |
|----------------------|----------------------------------|----------------------------------------|
| GUI Tool             | Included and running             | Disabled                               |
| Host port exposure   | Yes — mapped via env variable    | No — internal only                     |
| Container prefix     | `dev-`                           | `prod-`                                |
| Network              | Default project bridge           | Isolated per-database bridge           |
| RAM limit            | None                             | 512 MB (256 MB for Redis)              |
| CPU limit            | None                             | 0.5 vCPU (0.25 for Redis)              |
| Image variant        | Standard (e.g., `postgres:17`)   | Alpine where available (smaller, more secure) |
| Log driver           | json-file, 10 MB × 3 files       | json-file, 10 MB × 3 files             |
| Intended use         | Local development                | Server / cloud deployment              |

---

## Running in Production

> **Read the [Security Checklist](#security-checklist) before deploying to any internet-facing server.**

**Step 1.** Complete the [Quick Start](#quick-start) steps 1–4 in the target database folder.

**Step 2.** Start in production mode using both compose files:

```bash
docker compose -f docker-compose.yaml -f docker-compose.prod.yaml up -d
```

**Step 3.** Verify the container started with the correct name and is healthy:

```bash
docker compose -f docker-compose.yaml -f docker-compose.prod.yaml ps
```

The production container name is `prod-<database>` (e.g., `prod-postgres`). Status should show `running (healthy)`.

**Step 4.** Confirm the database port is not exposed to the host:

```bash
docker inspect prod-postgres | grep -A5 '"Ports"'
```

Expected output: the host binding should be empty (`{}`), confirming the port is accessible only from within Docker networks.

**Step 5.** To connect your application to the production database, add your application's container to the database's isolated network. For example, for PostgreSQL:

```yaml
# In your application's docker-compose.yaml
services:
  app:
    networks:
      - postgresql_postgres_net   # project_name + network_name

networks:
  postgresql_postgres_net:
    external: true
```

---

## Environment Variables

Each database folder has its own `.env` file. The table below lists the key variables across all databases. Refer to each folder's README for the full variable reference and required values.

| Variable                     | Database   | Description                                    | Required |
|------------------------------|------------|------------------------------------------------|----------|
| `POSTGRES_USER`              | PostgreSQL | Database superuser username                    | Yes      |
| `POSTGRES_PASSWORD`          | PostgreSQL | Database superuser password                    | Yes      |
| `POSTGRES_DB`                | PostgreSQL | Default database name                          | Yes      |
| `POSTGRES_PORT`              | PostgreSQL | Host port (default: `5432`)                    | Yes      |
| `PGADMIN_EMAIL`              | PostgreSQL | pgAdmin login email                            | Yes      |
| `PGADMIN_PASSWORD`           | PostgreSQL | pgAdmin login password                         | Yes      |
| `PGADMIN_PORT`               | PostgreSQL | pgAdmin host port (default: `5050`)            | Yes      |
| `MYSQL_ROOT_PASSWORD`        | MySQL      | MySQL root account password                    | Yes      |
| `MYSQL_USER`                 | MySQL      | Application database username                  | Yes      |
| `MYSQL_PASSWORD`             | MySQL      | Application database password                  | Yes      |
| `MYSQL_DATABASE`             | MySQL      | Default database name                          | Yes      |
| `MYSQL_PORT`                 | MySQL      | Host port (default: `3306`)                    | Yes      |
| `PHPMYADMIN_PORT`            | MySQL      | phpMyAdmin host port (default: `8080`)         | Yes      |
| `MONGO_INITDB_ROOT_USERNAME` | MongoDB    | MongoDB root admin username                    | Yes      |
| `MONGO_INITDB_ROOT_PASSWORD` | MongoDB    | MongoDB root admin password                    | Yes      |
| `MONGO_PORT`                 | MongoDB    | Host port (default: `27017`)                   | Yes      |
| `ME_CONFIG_BASICAUTH_USERNAME` | MongoDB  | Mongo Express web UI username                  | Yes      |
| `ME_CONFIG_BASICAUTH_PASSWORD` | MongoDB  | Mongo Express web UI password                  | Yes      |
| `MONGO_EXPRESS_PORT`         | MongoDB    | Mongo Express host port (default: `8081`)      | Yes      |
| `REDIS_PASSWORD`             | Redis      | Redis authentication password                  | Yes      |
| `REDIS_PORT`                 | Redis      | Host port (default: `6379`)                    | Yes      |
| `REDIS_MAX_MEMORY`           | Redis      | Max memory before eviction (e.g., `128mb`)     | Yes      |
| `REDISINSIGHT_PORT`          | Redis      | RedisInsight host port (default: `5540`)       | Yes      |

---

## Project Structure

```
docker-databases/
├── PostgreSQL/
│   ├── docker-compose.yaml       # Dev: postgres:17 + pgAdmin
│   ├── docker-compose.prod.yaml  # Prod override: alpine, limits, no GUI
│   ├── .env.example              # Template — copy to .env and fill in values
│   └── README.md                 # Full setup and reference for PostgreSQL
│
├── MySQL/
│   ├── docker-compose.yaml       # Dev: mysql:8.4 + phpMyAdmin
│   ├── docker-compose.prod.yaml  # Prod override: InnoDB tuning, limits, no GUI
│   ├── .env.example
│   └── README.md
│
├── MongoDB/
│   ├── docker-compose.yaml       # Dev: mongo:7 + Mongo Express
│   ├── docker-compose.prod.yaml  # Prod override: WiredTiger cap, limits, no GUI
│   ├── .env.example
│   └── README.md
│
├── Redis/
│   ├── docker-compose.yaml       # Dev: redis:7-alpine + RedisInsight
│   ├── docker-compose.prod.yaml  # Prod override: limits, no GUI
│   ├── .env.example
│   └── README.md
│
├── .gitignore                    # Ignores all .env files, keeps .env.example
└── README.md                     # This file
```

Each `.env` file is excluded from version control by `.gitignore`. Only `.env.example` files are committed. Never commit a `.env` file containing real credentials.

---

## Security Checklist

Complete this checklist before deploying any database to a production or internet-accessible server:

- [ ] **`.env` is not committed to git.** Run `git status` and confirm no `.env` files appear as tracked or staged.
- [ ] **All passwords have been changed** from the placeholder values in `.env.example`. Passwords should be at least 16 characters and randomly generated.
- [ ] **Database ports are not exposed to the public internet.** In prod mode, `ports: !reset []` removes host port bindings. Verify with `docker inspect <container> | grep -A5 Ports`.
- [ ] **GUI tools are disabled in prod mode.** Confirm `docker compose ps` shows no `pgadmin`, `phpmyadmin`, `mongo-express`, or `redisinsight` containers running.
- [ ] **Resource limits match your server's capacity.** The defaults (512 MB RAM / 0.5 CPU) are conservative starting points. Adjust them in `docker-compose.prod.yaml` based on your actual server specs and expected load.
- [ ] **`REDIS_MAX_MEMORY` is lower than the Redis container memory limit.** The default `128mb` is well below the 256 MB container limit. If you raise the limit, also raise `REDIS_MAX_MEMORY` proportionally.
- [ ] **MongoDB WiredTiger cache is sized below the container memory limit.** The `--wiredTigerCacheSizeGB 0.5` flag caps the cache at 512 MB, which matches the 512 MB container limit with very little headroom. For a production server, increase the container limit and cache together.
- [ ] **Docker Engine and images are up to date.** Run `docker pull <image>` for each image in use to ensure you have the latest security patches.
- [ ] **Firewall rules block direct access to Docker ports.** On Linux, Docker bypasses `ufw` and `firewalld` by writing directly to `iptables`. Use Docker's own network isolation (no port bindings in prod) rather than relying on host firewall rules.
- [ ] **Volumes are backed up on a schedule.** Named volumes are not backed up automatically. Set up a cron job or backup service before storing production data.

---

## Contributing

Contributions are welcome. Please follow the steps below to ensure a clean and reviewable pull request.

**Fork and clone:**

```bash
git clone https://github.com/<your-username>/docker-databases.git
cd docker-databases
```

**Create a branch using the naming convention below:**

| Type          | Branch name pattern             | Example                          |
|---------------|---------------------------------|----------------------------------|
| New feature   | `feat/<short-description>`      | `feat/add-cassandra`             |
| Bug fix       | `fix/<short-description>`       | `fix/redis-healthcheck-auth`     |
| Documentation | `docs/<short-description>`      | `docs/improve-mysql-readme`      |
| Refactor      | `refactor/<short-description>`  | `refactor/prod-network-config`   |

**Commit message format** — this project follows [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <short summary>

[optional body]
```

Examples:
```
feat(mongodb): add replica set support in prod mode
fix(redis): correct healthcheck auth flag order
docs(postgresql): add pgbouncer connection pooling note
```

Types: `feat`, `fix`, `docs`, `refactor`, `chore`
Scope: folder name in lowercase (`postgresql`, `mysql`, `mongodb`, `redis`, `root`)

**Open a pull request** against the `master` branch with a clear description of what changed and why.

---

## License

MIT License

Copyright (c) 2026 aslamabdika18

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
