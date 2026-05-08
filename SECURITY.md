# Security Policy

## Supported Versions

This repository provides Docker Compose configurations, not versioned software releases. Security fixes are applied to the `master` branch and are immediately available to all users.

## Reporting a Vulnerability

**Do not open a public GitHub issue for security vulnerabilities.**

If you discover a security issue — such as a configuration that exposes credentials, a misconfigured network that allows unintended access, or a default value that is insecure — please report it privately:

**Email**: aslamabdika@gmail.com  
**Subject**: `[SECURITY] docker-database — <short description>`

Please include:
- Which database folder is affected (`PostgreSQL`, `MySQL`, `MongoDB`, `Redis`)
- A description of the vulnerability and its potential impact
- Steps to reproduce or a proof-of-concept (if applicable)
- Your suggested fix (if you have one)

You will receive a response within **72 hours**. If the issue is confirmed, a fix will be committed and the reporter credited in the commit message (unless you prefer to remain anonymous).

## Security Design Notes

This repository is designed with the following security defaults:

- **No credentials in version control** — `.env` files are excluded via `.gitignore`. Only `.env.example` files with placeholder values are committed.
- **Dangerous commands disabled in Redis** — `FLUSHALL`, `FLUSHDB`, and `DEBUG` are renamed to empty strings by default.
- **GUI tools disabled in production** — `pgAdmin`, `phpMyAdmin`, `Mongo Express`, and `RedisInsight` are assigned a `dev-only` profile and do not start in prod mode.
- **No host port exposure in production** — the prod override uses `ports: !reset []` to remove all host port bindings.
- **Resource limits enforced in production** — all prod containers have explicit CPU and memory limits to reduce the blast radius of a compromised container.

## Scope

The following are **in scope** for security reports:

- Configurations that expose credentials or sensitive data
- Network configurations that allow unintended access between services
- Default values in `.env.example` that could mislead users into insecure deployments
- Missing security controls that should be present by default

The following are **out of scope**:

- Vulnerabilities in the upstream Docker images themselves (`postgres`, `mysql`, `mongo`, `redis`) — report those to the respective projects
- Issues that require the user to have already made insecure changes to their `.env` file
