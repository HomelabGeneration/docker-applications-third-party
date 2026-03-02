# Traefik Reverse Proxy

Traefik v3 reverse proxy with automatic Let's Encrypt certificates via deSEC DNS-01 challenge. Provides `*.lab.denismir.de` wildcard TLS for all services.

## Pre-Requisites

### Volume

```bash
docker volume create traefik_certificates
```

### Secrets

```bash
printf '<desec-api-token>' | docker secret create traefik_desec_token -
```

Generate a token at [deSEC Token Management](https://desec.io/tokens).

### Network

```bash
docker network create --driver overlay traefik-public
```

### Dynamic Config Directory

Create a directory on the Docker host for file-based routing rules (non-Docker services):

```bash
mkdir -p /opt/traefik/dynamic
chmod 755 /opt/traefik/dynamic
```

Config files inside the directory need read permissions (`644`). Traefik runs as `root` in the container and only needs read access.

## Environment Variables

| Variable | Description | Default |
|---|---|---|
| `ACME_EMAIL` | Email for Let's Encrypt registration | |
| `DASHBOARD_HOST` | Hostname for the Traefik dashboard (e.g. `traefik.lab.denismir.de`) | |
| `TLS_DOMAIN` | Base domain for wildcard certificate (e.g. `lab.denismir.de`) | |
| `DYNAMIC_CONFIG_PATH` | Host path to dynamic config directory | `/opt/traefik/dynamic` |
| `VOLUME_NAME_CERTIFICATES` | Name of the external certificates volume | `traefik_certificates` |

## Portainer App Template

```json
{
  "type": 2,
  "title": "Traefik",
  "description": "Reverse proxy with automatic Let's Encrypt TLS via deSEC DNS-01",
  "categories": ["Homelab", "Networking", "Reverse Proxy"],
  "platform": "linux",
  "repository": {
    "url": "https://github.com/HomelabGeneration/docker-applications-third-party",
    "stackfile": "applications/traefik/docker-compose.yml"
  },
  "env": [
    {
      "name": "ACME_EMAIL",
      "label": "ACME email",
      "description": "Email address for Let's Encrypt certificate registration"
    },
    {
      "name": "DASHBOARD_HOST",
      "label": "Dashboard hostname",
      "description": "Hostname for the Traefik dashboard (e.g. traefik.lab.denismir.de)"
    },
    {
      "name": "TLS_DOMAIN",
      "label": "TLS domain",
      "description": "Base domain for wildcard certificate (e.g. lab.denismir.de)"
    },
    {
      "name": "DYNAMIC_CONFIG_PATH",
      "label": "Dynamic config path",
      "description": "Host path to the dynamic configuration directory",
      "default": "/opt/traefik/dynamic"
    },
    {
      "name": "VOLUME_NAME_CERTIFICATES",
      "label": "Volume name certificates",
      "description": "Volume name for Let's Encrypt certificate storage",
      "default": "traefik_certificates"
    }
  ]
}
```

## Usage with Other Services

Add these deploy labels to any Docker Swarm service to route it through Traefik:

```yaml
services:
  my-app:
    image: my-app:latest
    networks:
      - traefik-public
    deploy:
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.my-app.rule=Host(`my-app.lab.denismir.de`)"
        - "traefik.http.routers.my-app.entrypoints=websecure"
        - "traefik.http.routers.my-app.tls.certresolver=desecresolver"
        - "traefik.http.services.my-app.loadbalancer.server.port=8080"

networks:
  traefik-public:
    external: true
```

The service must be attached to the `traefik-public` overlay network. Traefik discovers services via Docker labels — no additional configuration needed.

## File Provider (Non-Docker Services)

To route traffic to services running outside Docker (e.g. Proxmox VE, PBS, TrueNAS), place YAML files in the dynamic config directory (`${DYNAMIC_CONFIG_PATH}`).

Example `/opt/traefik/dynamic/proxmox.yml`:

```yaml
http:
  routers:
    proxmox:
      rule: "Host(`pve.lab.denismir.de`)"
      entrypoints:
        - websecure
      tls:
        certResolver: desecresolver
      service: proxmox

  services:
    proxmox:
      loadBalancer:
        servers:
          - url: "https://192.168.1.10:8006"
```

Traefik watches the directory and picks up changes automatically. The `insecureSkipVerify` transport option is enabled so Traefik can connect to backends with self-signed certificates.
