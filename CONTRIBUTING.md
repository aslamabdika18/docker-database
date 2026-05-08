# Contributing to docker-database

Thank you for taking the time to contribute. This document explains how to set up your environment, what standards to follow, and how to submit changes.

---

## Table of Contents

1. [Getting Started](#getting-started)
2. [What Can I Contribute?](#what-can-i-contribute)
3. [Branch Naming](#branch-naming)
4. [Commit Message Format](#commit-message-format)
5. [Pull Request Process](#pull-request-process)
6. [Code Standards](#code-standards)
7. [Reporting Bugs](#reporting-bugs)

---

## Getting Started

**1.** Fork the repository on GitHub, then clone your fork:

```bash
git clone https://github.com/<your-username>/docker-database.git
cd docker-database
```

**2.** Add the upstream remote so you can pull future changes:

```bash
git remote add upstream https://github.com/aslamabdika18/docker-database.git
```

**3.** Create a branch for your change (see [Branch Naming](#branch-naming)):

```bash
git checkout -b feat/add-mariadb
```

**4.** Make your changes, then test locally:

```bash
cd <database-folder>
cp .env.example .env
# edit .env with test values
docker compose up -d
docker compose ps
```

**5.** Push and open a Pull Request against the `master` branch.

---

## What Can I Contribute?

| Type | Examples |
|------|---------|
| New database | MariaDB, Cassandra, Elasticsearch, MinIO |
| Config improvement | Better healthcheck, new prod tuning flag |
| Documentation | Fix typos, improve setup steps, add examples |
| Bug fix | Container fails to start, healthcheck always fails |

Before starting on a large change, open an issue first to discuss it.

---

## Branch Naming

| Type | Pattern | Example |
|------|---------|---------|
| New feature | `feat/<description>` | `feat/add-mariadb` |
| Bug fix | `fix/<description>` | `fix/redis-healthcheck` |
| Documentation | `docs/<description>` | `docs/improve-postgres-readme` |
| Refactor | `refactor/<description>` | `refactor/prod-network` |

---

## Commit Message Format

This project follows [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <short summary>

[optional body explaining why, not what]
```

**Types:** `feat`, `fix`, `docs`, `refactor`, `chore`

**Scopes:** `postgresql`, `mysql`, `mongodb`, `redis`, `root`

**Examples:**
```
feat(mongodb): add replica set support in prod mode
fix(redis): correct password flag order in healthcheck
docs(mysql): add ProxySQL connection pooling note
chore(root): update .gitignore to exclude OS files
```

Rules:
- Use the imperative mood: "add" not "added", "fix" not "fixed"
- Keep the summary under 72 characters
- Do not end the summary with a period

---

## Pull Request Process

1. Ensure your branch is up to date with `upstream/master` before opening a PR:
   ```bash
   git fetch upstream
   git rebase upstream/master
   ```

2. Fill in the PR template completely — incomplete PRs will be asked to revise.

3. Every PR that touches a compose file must be tested locally with both dev and prod modes.

4. Every PR that adds a new environment variable must update the corresponding `.env.example`.

5. Every PR that changes behavior must update the corresponding `README.md`.

6. PRs are merged using **squash merge** to keep the history linear.

---

## Code Standards

**Docker Compose files:**
- Use 2-space indentation
- All services must have `restart: unless-stopped`
- All services must have a `healthcheck` with `start_period`
- All services must have `logging` with `json-file` driver
- Environment variables must use `${VAR}` syntax, not hardcoded values
- Dev compose uses `dev-` container prefix; prod uses `prod-`

**Environment files:**
- `.env.example` must contain only placeholder values (no real credentials)
- Every variable must have a comment explaining its purpose if not self-evident

**README files:**
- Written in English
- All commands must be inside fenced code blocks and tested
- Follow the section structure of existing README files

---

## Reporting Bugs

Use the [Bug Report](.github/ISSUE_TEMPLATE/bug_report.md) issue template. Always include:
- Docker and Docker Compose versions
- Full output of `docker compose logs <service>`
- Your OS and architecture

For security vulnerabilities, do **not** open a public issue. See [SECURITY.md](SECURITY.md) instead.
