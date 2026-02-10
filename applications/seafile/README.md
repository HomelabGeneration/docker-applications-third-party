# Seafile 13 Community Edition

## Pre-Requisites

### Volumes

```bash
docker volume create seafile_data
docker volume create seafile_db
```

### Secrets

```bash
printf '<mysql-root-password>' | docker secret create seafile_mysql_root_password -
printf '<seafile-db-password>' | docker secret create seafile_db_password -
printf '<admin-password>' | docker secret create seafile_admin_password -
openssl rand -hex 20 | docker secret create seafile_jwt_private_key -
```

## Portainer App Template

```json
{
  "type": 2,
  "title": "Seafile",
  "description": "Self-hosted file sync and share solution (Community Edition 13)",
  "categories": ["Homelab", "Tooling", "Storage"],
  "platform": "linux",
  "repository": {
    "url": "https://github.com/HomelabGeneration/docker-applications-third-party",
    "stackfile": "applications/seafile/docker-compose.yml"
  },
  "env": [
    {
      "name": "HTTP_PORT",
      "label": "HTTP port",
      "default": "8080"
    },
    {
      "name": "VOLUME_NAME_DATA",
      "label": "Volume name data",
      "description": "Volume name for the seafile shared data",
      "default": "seafile_data"
    },
    {
      "name": "VOLUME_NAME_DB",
      "label": "Volume name db",
      "description": "Volume name for the seafile database",
      "default": "seafile_db"
    },
    {
      "name": "SEAFILE_DB_USER",
      "label": "DB username",
      "description": "MariaDB user for seafile",
      "default": "seafile"
    },
    {
      "name": "SEAFILE_SERVER_HOSTNAME",
      "label": "Server hostname",
      "description": "Public hostname for the seafile server (e.g. seafile.example.com)"
    },
    {
      "name": "SEAFILE_SERVER_PROTOCOL",
      "label": "Server protocol",
      "description": "Protocol used to access seafile (http or https)",
      "default": "https"
    },
    {
      "name": "SEAFILE_ADMIN_EMAIL",
      "label": "Admin email",
      "description": "Email address for the initial seafile admin account"
    }
  ]
}
```

## Notes

- Secrets werden per `command`-Override in den Seafile-Container injiziert, da das Image `_FILE`-Env-Vars nicht nativ unterstuetzt
- MariaDB nutzt den nativen `MYSQL_ROOT_PASSWORD_FILE`-Support
- Redis laeuft ohne Passwort (nur internes Docker-Netzwerk)
- SeaDoc ist deaktiviert, kann spaeter als zusaetzlicher Service ergaenzt werden
