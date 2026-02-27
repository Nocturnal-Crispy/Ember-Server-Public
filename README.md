# Ember Server Deployment Guide

This guide explains how to deploy the Ember chat server using either the binary executable or Docker image.

## Quick Start

### Prerequisites

- Linux system (x86_64 architecture) **OR** Docker/Docker Compose
- PostgreSQL database (can be installed locally or hosted)
- Docker and Docker Compose (for WebRTC services like TURN/STUN and SFU)

## Deployment Options

You have two main deployment options:

1. **Binary Deployment** - Run the executable directly
2. **Docker Deployment** - Use the pre-built Docker image

Choose the option that best fits your environment.

---

## Option 1: Binary Deployment

### Step 1: Download and Prepare

1. Download the `ember-linux` binary from the [GitHub releases](https://github.com/Nocturnal-Crispy/Ember-Server-Public/releases/latest)
2. Make it executable:
   ```bash
   chmod +x ember-linux
   ```

### Step 2: Database Setup

Ember requires a PostgreSQL database. You have two options:

#### Option A: Use Docker Compose (Recommended for testing)
```bash
# Copy the example environment file
cp deploy/env.example .env

# Edit .env with your database credentials
nano .env

# Start PostgreSQL only
docker-compose -f deploy/docker-compose.yml up -d postgres
```

#### Option B: Use existing PostgreSQL
Ensure you have a PostgreSQL database and note these values:
- Host and port
- Database name
- Username and password

### Step 3: Configure Environment

Create a `.env` file with the following configuration:

```bash
# Environment
ENVIRONMENT=production

# Database
DB_HOST=localhost          # or your PostgreSQL host
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=your-db-password
DB_NAME=ember
DB_SSL_MODE=require        # use "disable" for local development

# Server ports
SERVER_PORT=8085          # HTTP API server
WS_PORT=8086              # WebSocket server

# Secrets (generate new ones for production!)
JWT_SECRET=your-super-secret-jwt-key-here
TURN_SECRET=your-turn-secret-here

# Public hostname (required in production)
PUBLIC_HOST=https://your-domain.com

# CORS (optional)
ALLOWED_ORIGINS=https://your-domain.com
```

### Step 4: Run the Server

#### Basic Deployment (API only)
```bash
./ember-linux
```

The server will start on:
- HTTP API: `http://localhost:8085`
- WebSocket: `ws://localhost:8086`

#### Full Deployment with WebRTC Support
For video/audio calling, you'll need additional services:

```bash
# Start all services (PostgreSQL, TURN server, SFU)
docker-compose -f deploy/docker-compose.yml up -d

# Then run the ember server
./ember-linux
```

---

## Option 2: Docker Deployment

### Step 1: Pull the Docker Image

```bash
docker pull nocturnalcrispy/ember-chat-server:latest
```

### Step 2: Configure Environment

Copy and edit the environment file:

```bash
cp deploy/env.example deploy/.env
nano deploy/.env
```

Configure the same environment variables as shown in the binary deployment section.

### Step 3: Run with Docker Compose

#### Basic Deployment (API + Database only)
```bash
docker-compose -f deploy/docker-compose.yml up -d
```

#### Production Deployment
For production, use the production compose file:

```bash
docker-compose -f deploy/docker-compose-prod.yml --env-file deploy/.env up -d
```

This will start:
- PostgreSQL database
- Ember server (from Docker Hub image)
- Coturn (TURN/STUN server for WebRTC)
- Ion-SFU (video streaming for WebRTC)

### Step 4: Verify Deployment

Check that the server is running:
```bash
curl http://localhost:8085/health
```

Or check the container status:
```bash
docker-compose -f deploy/docker-compose.yml ps
```

---

## Step 5: Verify Deployment (Both Options)

Check that the server is running:
```bash
curl http://localhost:8085/health
```

You should see a health check response.

## Production Deployment

### Choosing Your Deployment Method

**Choose Binary Deployment if:**
- You want direct control over the executable
- You need to customize the runtime environment
- You prefer traditional server management

**Choose Docker Deployment if:**
- You want consistent environments across machines
- You need easy scaling and orchestration
- You prefer containerized deployments

### Security Considerations

1. **Generate new secrets** before going to production:
   ```bash
   # Generate JWT secret (32+ characters)
   openssl rand -base64 32
   
   # Generate TURN secret
   openssl rand -base64 16
   ```

2. **Enable SSL** for your database connection in production

3. **Set PUBLIC_HOST** to your actual domain

4. **Configure firewall** to only allow necessary ports:
   - 8085 (HTTP API)
   - 8086 (WebSocket)
   - 443/80 (if using reverse proxy)
   - TURN/STUN ports (if using WebRTC)

### Production Setup - Binary

#### Systemd Service
Create a systemd service for automatic startup:

```ini
# /etc/systemd/system/ember.service
[Unit]
Description=Ember Chat Server
After=network.target

[Service]
Type=simple
User=ember
WorkingDirectory=/opt/ember
ExecStart=/opt/ember/ember-linux
Restart=always
RestartSec=5
Environment=ENVIRONMENT=production

[Install]
WantedBy=multi-user.target
```

Enable and start:
```bash
sudo systemctl enable ember
sudo systemctl start ember
```

### Production Setup - Docker

#### Using Docker Compose
For production, use the production compose file:

```bash
# Copy and configure environment
cp deploy/env.example deploy/.env
nano deploy/.env

# Start production services
docker-compose -f deploy/docker-compose-prod.yml --env-file deploy/.env up -d
```

#### Docker Service Management
```bash
# Check status
docker-compose -f deploy/docker-compose-prod.yml ps

# View logs
docker-compose -f deploy/docker-compose-prod.yml logs -f emberd

# Restart services
docker-compose -f deploy/docker-compose-prod.yml restart emberd

# Stop services
docker-compose -f deploy/docker-compose-prod.yml down
```

### Reverse Proxy Setup (Both Options)

Using nginx as a reverse proxy is recommended:

```nginx
server {
    listen 80;
    server_name your-domain.com;
    
    location / {
        proxy_pass http://localhost:8085;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
    
    location /ws {
        proxy_pass http://localhost:8086;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }
}
```

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `ENVIRONMENT` | No | development | Set to "production" for production |
| `DB_HOST` | Yes | localhost | PostgreSQL host |
| `DB_PORT` | Yes | 5432 | PostgreSQL port |
| `DB_USER` | Yes | postgres | PostgreSQL username |
| `DB_PASSWORD` | Yes | - | PostgreSQL password |
| `DB_NAME` | Yes | ember | Database name |
| `DB_SSL_MODE` | No | disable | PostgreSQL SSL mode |
| `SERVER_PORT` | No | 8085 | HTTP API port |
| `WS_PORT` | No | 8086 | WebSocket port |
| `JWT_SECRET` | Yes | - | JWT signing secret |
| `TURN_SECRET` | Yes | - | TURN server secret |
| `PUBLIC_HOST` | Yes (production) | - | Public base URL |
| `ALLOWED_ORIGINS` | No | file:// | CORS allowed origins |

## WebRTC Services (Optional)

For video/audio calling, the following services are included:

- **Coturn**: TURN/STUN server for NAT traversal
- **Ion-SFU**: Selective Forwarding Unit for video streams

These are configured in `docker-compose.yml` and will start automatically when using Docker Compose.

## Troubleshooting

### Database Connection Issues
- Verify PostgreSQL is running
- Check connection parameters in `.env`
- Ensure database exists and user has permissions

### Port Conflicts
- Change `SERVER_PORT` and `WS_PORT` in `.env` if ports are in use
- Update firewall rules accordingly

### WebSocket Issues
- Ensure reverse proxy supports WebSocket upgrades
- Check `ALLOWED_ORIGINS` CORS configuration

## Support

For issues and questions:
- Check the server logs for error messages
- Verify all environment variables are set correctly
- Ensure database connectivity and permissions
