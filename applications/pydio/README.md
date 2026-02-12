# Pydio Cells

## Pre-Requisites

### Volumes

```bash
docker volume create pydio_data
docker volume create pydio_db
```

### Secrets

```bash
printf '<db-password>' | docker secret create pydio_db_password -
```

## Portainer App Template

```json
{
  "type": 2,
  "title": "Pydio Cells",
  "description": "Self-hosted file sync and share platform",
  "categories": ["Homelab", "Tooling", "Storage"],
  "platform": "linux",
  "repository": {
    "url": "https://github.com/HomelabGeneration/docker-applications-third-party",
    "stackfile": "applications/pydio/docker-compose.yml"
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
      "description": "Volume name for the Pydio Cells data",
      "default": "pydio_data"
    },
    {
      "name": "VOLUME_NAME_DB",
      "label": "Volume name db",
      "description": "Volume name for the Pydio Cells database",
      "default": "pydio_db"
    },
    {
      "name": "CELLS_EXTERNAL_URL",
      "label": "External URL",
      "description": "Public URL for accessing Pydio Cells (e.g. https://pydio.example.com)"
    }
  ]
}
```

## Notes

- MariaDB root password is randomly generated (`MYSQL_RANDOM_ROOT_PASSWORD`) since Pydio connects as the regular `pydio` user
- Initial configuration (admin account, database connection) is done via the Pydio web installer at first start
- Web installer DB settings: Host `db`, Database `cells`, User `pydio`, Password from the `pydio_db_password` secret
- TLS is disabled (`CELLS_NO_TLS=1`) — use a reverse proxy (Traefik/Nginx) for SSL termination
