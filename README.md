# Ember Server --- Deployment (Public)

This repository contains a public deployment bundle for the Ember chat
server, including a Docker Compose stack for:

-   Ember API + WebSocket server (`emberd`)
-   PostgreSQL (`postgres`)
-   Coturn TURN/STUN (`coturn`)
-   ION SFU (`ion-sfu`)

------------------------------------------------------------------------

## Quick Start (Docker Compose)

### Prerequisites

-   Docker + Docker Compose
-   A public hostname + TLS (recommended for production)
-   Firewall access for required ports

### 1) Create your .env

``` bash
cp env.example .env
nano .env
```

### 2) Start the stack

``` bash
docker compose up -d
```

### 3) Verify

``` bash
curl http://localhost:8085/health
docker compose ps
docker compose logs -f emberd
```

------------------------------------------------------------------------

## Ports & Firewall

### Ember

-   HTTP API: 8085
-   WebSocket: 8086

### PostgreSQL

-   DB: 5432

### Coturn (Host Network Mode)

-   TURN/STUN: 3478 (UDP/TCP)
-   TURN TLS: 5349 (TCP)
-   Relay Ports: 49152--65535 (UDP)

### ION SFU

-   WebSocket Signaling: 7000 (TCP)
-   WebRTC Media: 5000--5200 (UDP)

Ensure your cloud firewall and host firewall allow these ports.

------------------------------------------------------------------------

## Configuration

Key environment variables (see env.example):

-   ENVIRONMENT=production (for production)
-   DB_USER
-   DB_PASSWORD
-   DB_NAME
-   JWT_SECRET (rotate in production)
-   TURN_SECRET (rotate in production)
-   PUBLIC_HOST
-   ALLOWED_ORIGINS

------------------------------------------------------------------------

## Reverse Proxy Example (Nginx)

``` nginx
server {
  listen 80;
  server_name your-domain.com;

  location / {
    proxy_pass http://127.0.0.1:8085;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
  }

  location /ws {
    proxy_pass http://127.0.0.1:8086;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
  }
}
```

TURN and SFU ports must still be directly exposed.

------------------------------------------------------------------------

## Security Checklist

-   Rotate JWT_SECRET and TURN_SECRET
-   Restrict ALLOWED_ORIGINS
-   Use TLS for public deployments
-   Limit TURN relay port ranges if possible

------------------------------------------------------------------------

## Troubleshooting

View logs:

``` bash
docker compose logs -f emberd
docker compose logs -f coturn
docker compose logs -f ion-sfu
docker compose logs -f postgres
```

------------------------------------------------------------------------

## Support

Open an issue and include: - Redacted .env - docker compose ps output -
Relevant service logs
