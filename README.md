# Ember Server --- Deployment (Public)

This repository contains a public deployment bundle for the Ember chat
server, including a Docker Compose stack for:

-   Ember API + WebSocket server (`emberd`)
-   PostgreSQL (`postgres`)
-   Coturn TURN/STUN (`coturn`)
-   ION SFU (`ion-sfu`)

This README covers **three deployment methods**:

1.  Full Docker Compose stack (recommended)
2.  Standalone Docker image
3.  Standalone executable (binary)

------------------------------------------------------------------------

# 🚀 Option 1 --- Full Docker Compose (Recommended)

## Prerequisites

-   Docker + Docker Compose
-   A public hostname + TLS (recommended for production)
-   Firewall access for required ports

## 1) Create your .env

``` bash
cp env.example .env
nano .env
```

## 2) Start the stack

``` bash
docker compose up -d
```

## 3) Verify

``` bash
curl http://localhost:8085/health
docker compose ps
docker compose logs -f emberd
```

------------------------------------------------------------------------

# 🐳 Option 2 --- Docker Image Only

If you already have PostgreSQL and optionally your own TURN/SFU setup,
you can deploy only the Ember server container.

## Pull Image

``` bash
docker pull nocturnalcrispy/ember-chat-server:latest
```

## Run Container

``` bash
docker run -d   --name emberd   -p 8085:8085   -p 8086:8086   -e ENVIRONMENT=production   -e DB_HOST=your-db-host   -e DB_PORT=5432   -e DB_USER=your-user   -e DB_PASSWORD=your-password   -e DB_NAME=ember   -e JWT_SECRET=super-secret   -e TURN_SECRET=turn-secret   -e PUBLIC_HOST=your-domain.com   nocturnalcrispy/ember-chat-server:latest
```

Ensure your database is reachable and properly initialized.

------------------------------------------------------------------------

# 🧱 Option 3 --- Standalone Executable (Binary)

If you prefer not to use Docker, you can run the compiled executable
directly.

## 1) Download Binary

Download the latest release for your platform from GitHub Releases.

Example for Linux:

``` bash
chmod +x ember-linux
```

## 2) Create .env File

Create a `.env` file in the same directory as the binary with:

``` bash
ENVIRONMENT=production
DB_HOST=localhost
DB_PORT=5432
DB_USER=ember
DB_PASSWORD=change-me
DB_NAME=ember
JWT_SECRET=change-me
TURN_SECRET=change-me
PUBLIC_HOST=your-domain.com
ALLOWED_ORIGINS=https://your-domain.com
```

## 3) Run Server

``` bash
./ember-linux
```

The API will start on port 8085 and WebSocket on 8086 by default.

------------------------------------------------------------------------

# 🔌 Ports & Firewall

## Ember

-   HTTP API: 8085
-   WebSocket: 8086

## PostgreSQL

-   DB: 5432

## Coturn (Host Network Mode if using compose)

-   TURN/STUN: 3478 (UDP/TCP)
-   TURN TLS: 5349 (TCP)
-   Relay Ports: 49152--65535 (UDP)

## ION SFU

-   WebSocket Signaling: 7000 (TCP)
-   WebRTC Media: 5000--5200 (UDP)

Ensure your cloud firewall and host firewall allow these ports.

------------------------------------------------------------------------

# 🔒 Security Checklist

-   Rotate JWT_SECRET and TURN_SECRET
-   Restrict ALLOWED_ORIGINS
-   Use TLS for public deployments
-   Limit TURN relay port ranges if possible
-   Never expose PostgreSQL publicly

------------------------------------------------------------------------

# 🌐 Reverse Proxy Example (Nginx)

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

# 🛠 Troubleshooting

View logs (Docker):

``` bash
docker logs emberd
```

View logs (Compose):

``` bash
docker compose logs -f emberd
docker compose logs -f coturn
docker compose logs -f ion-sfu
docker compose logs -f postgres
```

------------------------------------------------------------------------

# 📬 Support

When opening an issue include: - Redacted .env - docker compose ps
output - Relevant service logs
