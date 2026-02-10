# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Monorepo of third-party Docker applications, each configured as Docker Compose services under `applications/`. All compose files use format version `3.8`.

## Conventions

### Commit Messages

Prefix with `[TASK]` or `[BUGFIX]`:
```
[TASK] Adds immich changes for v2.5.0
[BUGFIX] Fixes the cloudflared application part2
```

Always sign commits with `-S` flag.

### Docker Compose Patterns

- **External volumes**: Volumes are declared as external (must be pre-created). Volume names use `${VARIABLE}` substitution.
- **Secrets**: Sensitive data uses Docker secrets mounted at `/run/secrets/`, not environment variables. Example: `POSTGRES_PASSWORD_FILE: /run/secrets/<service>_pg_password`.
- **Backup labels**: Services with volumes use `docker-volume-backup.stop-during-backup=<service-name>` labels for backup integration.
- **Timezone**: Hardcoded to `Europe/Berlin` (via `TZ` env var or `/etc/localtime` mount).
- **Restart policy**: `restart: unless-stopped` or `deploy.restart_policy.condition: on-failure`.
- **Swarm compatibility**: Many apps include `deploy` keys (replicas, placement constraints, configs) for Docker Swarm support.

### Deployment

No build scripts or CI/CD. Applications are deployed directly via Docker Compose or Docker Stack:
```bash
# Standalone
docker compose -f applications/<app>/docker-compose.yml up -d

# Swarm
docker stack deploy -c applications/<app>/docker-compose.yml <stack-name>
```

Pre-requisites before deploying: create external volumes and Docker secrets referenced in the compose file.