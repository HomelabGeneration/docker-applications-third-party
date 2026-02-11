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
    },
    {
      "name": "COLLABORA_PORT",
      "label": "Collabora port",
      "description": "Port for Collabora Online (office editing)",
      "default": "9980"
    },
    {
      "name": "COLLABORA_SERVER_HOSTNAME",
      "label": "Collabora hostname",
      "description": "Public hostname for Collabora (e.g. collabora.example.com, must be reachable from browser via reverse proxy)"
    }
  ]
}
```

## Collabora Online (Office Editing)

Collabora Online ermoeglicht das Bearbeiten von Office-Dokumenten direkt im Browser.

### Voraussetzungen

- Collabora muss ueber eine eigene Subdomain extern erreichbar sein (z.B. `collabora.example.com`)
- Reverse-Proxy-Route (Pangolin) von `collabora.example.com` auf Port `${COLLABORA_PORT}` konfigurieren

### Pangolin: Auth-Bypass fuer WOPI-Pfade

Wenn die Seafile-Route in Pangolin mit Authentifizierung geschuetzt ist, muss ein Pfad-Bypass fuer die WOPI-API eingerichtet werden. Andernfalls kann Collabora den WOPI-Callback (`CheckFileInfo`, `GetFile`, `PutFile`) nicht ausfuehren, da Pangolin den Request mit einer Login-Seite blockt.

**Symptom:** Beim Oeffnen von Office-Dateien erscheint "Access denied" und in den Collabora-Logs:
```
WOPI::CheckFileInfo failed or no valid JSON payload returned. Access denied.
```

**Loesung:** In der Pangolin-Admin-UI fuer die Seafile-Route einen Pfad-Bypass konfigurieren:

- **Pfad-Muster:** `/api2/wopi/*`
- **Methoden:** GET, POST

Die WOPI-Endpoints sind ueber Access Tokens geschuetzt (`WOPI_ACCESS_TOKEN_EXPIRATION` in `seahub_settings.py`), daher ist kein zusaetzlicher Auth-Layer noetig.

### Seahub Konfiguration

Nach dem ersten Start von Seafile muss `seahub_settings.py` ergaenzt werden:

```python
# Pfad im Container: /shared/seafile/conf/seahub_settings.py
# Bzw. im Volume: <seafile-data-volume>/seafile/conf/seahub_settings.py

ENABLE_OFFICE_WEB_APP = True
OFFICE_SERVER_TYPE = 'CollaboraOffice'
OFFICE_WEB_APP_BASE_URL = 'http://collabora:9980/hosting/discovery'
OFFICE_WEB_APP_FILE_EXTENSION = ('odp', 'ods', 'odt', 'xls', 'xlsb', 'xlsm', 'xlsx', 'ppsx', 'ppt', 'pptm', 'pptx', 'doc', 'docm', 'docx')
OFFICE_WEB_APP_EDIT_FILE_EXTENSION = ('odp', 'ods', 'odt', 'xls', 'xlsb', 'xlsm', 'xlsx', 'ppsx', 'ppt', 'pptm', 'pptx', 'doc', 'docm', 'docx')
ENABLE_OFFICE_WEB_APP_EDIT = True
WOPI_ACCESS_TOKEN_EXPIRATION = 1800
```

Danach Seafile-Container neustarten.

**Hinweis:** `OFFICE_WEB_APP_BASE_URL` nutzt den Docker-internen Hostnamen `collabora` (Server-zu-Server-Kommunikation). Der Browser erreicht Collabora ueber die externe URL dank `server_name`-Konfiguration im Collabora-Container.

## Notes

- Secrets werden per `command`-Override in den Seafile-Container injiziert, da das Image `_FILE`-Env-Vars nicht nativ unterstuetzt
- MariaDB nutzt den nativen `MYSQL_ROOT_PASSWORD_FILE`-Support
- Redis laeuft ohne Passwort (nur internes Docker-Netzwerk)
- SeaDoc ist deaktiviert, kann spaeter als zusaetzlicher Service ergaenzt werden
