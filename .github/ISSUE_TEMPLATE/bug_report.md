---
name: Bug Report
about: Report a problem with a Docker Compose configuration or container behavior
title: "[BUG] <short description>"
labels: bug
assignees: aslamabdika18
---

## Database
Which database is affected?
- [ ] PostgreSQL
- [ ] MySQL
- [ ] MongoDB
- [ ] Redis

## Mode
- [ ] Dev (`docker compose up -d`)
- [ ] Prod (`docker compose -f docker-compose.yaml -f docker-compose.prod.yaml up -d`)

## Description
A clear and concise description of the bug.

## Steps to Reproduce
1. Go to `...`
2. Run `...`
3. See error

## Expected Behavior
What you expected to happen.

## Actual Behavior
What actually happened. Include the full error message.

## Logs
```
paste output of: docker compose logs <service-name>
```

## Environment
- OS: (e.g., Ubuntu 22.04, macOS 14, Windows 11)
- Docker version: (output of `docker --version`)
- Docker Compose version: (output of `docker compose version`)
