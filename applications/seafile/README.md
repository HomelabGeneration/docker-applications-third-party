# Seafile 13 Community Edition

## Pre-Requisites

### Volumes

```bash
docker volume create seafile_data
docker volume create seafile_db
docker volume create seafile_onlyoffice_data
docker volume create seafile_onlyoffice_lib
docker volume create seafile_onlyoffice_log
```

### Secrets

```bash
printf '<mysql-root-password>' | docker secret create seafile_mysql_root_password -
printf '<seafile-db-password>' | docker secret create seafile_db_password -
printf '<admin-password>' | docker secret create seafile_admin_password -
openssl rand -hex 20 | docker secret create seafile_jwt_private_key -
pwgen -s 40 1 | docker secret create onlyoffice_jwt_secret -
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
    },
    {
      "name": "ONLYOFFICE_PORT",
      "label": "ONLYOFFICE port",
      "description": "Port for ONLYOFFICE Document Server",
      "default": "6233"
    },
    {
      "name": "VOLUME_NAME_ONLYOFFICE_DATA",
      "label": "Volume ONLYOFFICE data",
      "description": "ONLYOFFICE-Daten",
      "default": "seafile_onlyoffice_data"
    },
    {
      "name": "VOLUME_NAME_ONLYOFFICE_LIB",
      "label": "Volume ONLYOFFICE lib",
      "description": "ONLYOFFICE-Lib (PostgreSQL, Converter)",
      "default": "seafile_onlyoffice_lib"
    },
    {
      "name": "VOLUME_NAME_ONLYOFFICE_LOG",
      "label": "Volume ONLYOFFICE log",
      "description": "ONLYOFFICE-Logs",
      "default": "seafile_onlyoffice_log"
    }
  ]
}
```

## Collabora Online (Office Editing -- Default)

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

## ONLYOFFICE Document Server (Office Editing -- Alternative)

**Wichtig:** Collabora und ONLYOFFICE koennen nicht gleichzeitig aktiv sein. In `seahub_settings.py` muss entweder `ENABLE_OFFICE_WEB_APP` (Collabora) oder `ENABLE_ONLYOFFICE` aktiviert sein, nicht beides.

### Voraussetzungen

- ONLYOFFICE muss ueber einen Reverse-Proxy extern erreichbar sein (z.B. via Pangolin auf Port `${ONLYOFFICE_PORT}`)
- Die URL muss per HTTPS erreichbar sein (TLS-Terminierung im Reverse-Proxy)

### Pangolin: Keine Authentifizierung auf der ONLYOFFICE-Route

Die ONLYOFFICE-Route in Pangolin darf **keine Authentifizierung** haben. ONLYOFFICE schuetzt sich selbst ueber JWT (`JWT_ENABLED: true` + Shared Secret).

**Hintergrund:** Nach dem Bearbeiten sendet ONLYOFFICE einen Callback an Seafile mit einer Download-URL (z.B. `https://onlyoffice.example.com:6233/cache/files/...`). Seafile holt das fertige Dokument von dieser URL. Wenn Pangolin die Route mit Auth schuetzt, erhaelt Seafile statt des Dokuments die Pangolin-Login-Seite als HTML — das fuehrt zu korrupten Dateien.

**Symptom:** Bearbeitete Dateien enthalten nach dem Speichern HTML-Code (Pangolin-Login-Seite) statt des eigentlichen Dokumentinhalts.

### Seahub Konfiguration

Nach dem ersten Start von Seafile muss `seahub_settings.py` ergaenzt werden:

```python
# Pfad im Container: /shared/seafile/conf/seahub_settings.py
# Bzw. im Volume: <seafile-data-volume>/seafile/conf/seahub_settings.py

# WICHTIG: Collabora deaktivieren wenn ONLYOFFICE genutzt wird
# ENABLE_OFFICE_WEB_APP = False

ENABLE_ONLYOFFICE = True
ONLYOFFICE_APIJS_URL = 'https://seafile.example.com:6233/web-apps/apps/api/documents/api.js'
ONLYOFFICE_FILE_EXTENSION = ('doc', 'docx', 'ppt', 'pptx', 'xls', 'xlsx', 'odt', 'fodt', 'odp', 'fodp', 'ods', 'fods', 'csv', 'ppsx', 'pps')
ONLYOFFICE_EDIT_FILE_EXTENSION = ('docx', 'pptx', 'xlsx')
ONLYOFFICE_JWT_SECRET = '<your jwt secret>'  # Muss mit dem Docker Secret onlyoffice_jwt_secret uebereinstimmen
```

Danach Seafile-Container neustarten.

**Hinweise:**
- `ONLYOFFICE_APIJS_URL` muss die extern erreichbare URL sein (nicht den Docker-internen Hostnamen), da das JavaScript im Browser des Nutzers geladen wird
- `ONLYOFFICE_JWT_SECRET` in `seahub_settings.py` muss den gleichen Wert haben wie das Docker Secret `onlyoffice_jwt_secret`

## Notes

- Secrets werden per `command`-Override in den Seafile-Container injiziert, da das Image `_FILE`-Env-Vars nicht nativ unterstuetzt
- MariaDB nutzt den nativen `MYSQL_ROOT_PASSWORD_FILE`-Support
- Redis laeuft ohne Passwort (nur internes Docker-Netzwerk)
- SeaDoc ist deaktiviert, kann spaeter als zusaetzlicher Service ergaenzt werden
