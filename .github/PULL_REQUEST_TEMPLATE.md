## Description
A clear and concise description of what this PR changes and why.

Closes # (issue number, if applicable)

## Type of Change
- [ ] Bug fix
- [ ] New feature (new database or configuration option)
- [ ] Documentation update
- [ ] Refactor (no functional change)

## Database(s) Affected
- [ ] PostgreSQL
- [ ] MySQL
- [ ] MongoDB
- [ ] Redis
- [ ] Root / All

## Checklist
- [ ] I have tested the changes with `docker compose up -d` (dev mode)
- [ ] I have tested the changes with the prod override (if applicable)
- [ ] I have updated the relevant `README.md` to reflect the changes
- [ ] I have updated `.env.example` if new environment variables were added
- [ ] My commit messages follow the [Conventional Commits](https://www.conventionalcommits.org/) format
- [ ] No `.env` files or real credentials are included in this PR

## How to Test
Step-by-step instructions to verify this change works correctly:
1. `cd <database-folder>`
2. `cp .env.example .env`
3. `docker compose up -d`
4. Expected result: ...
