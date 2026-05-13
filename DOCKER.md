# Docker Deployment Guide for Mike

This guide explains how to run Mike using Docker and Docker Compose.

## Prerequisites

- Docker (version 20.10 or higher)
- Docker Compose (version 1.29 or higher)
- All environment variables configured (see below)

## Quick Start

### 1. Create Environment File

Copy the example environment file:

```bash
cp .env.docker.example .env.docker
```

Edit `.env.docker` and fill in your configuration values:

```bash
# Required: Supabase setup
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_SECRET_KEY=your-service-role-key
NEXT_PUBLIC_SUPABASE_PUBLISHABLE_DEFAULT_KEY=your-anon-key

# Required: At least one AI provider
ANTHROPIC_API_KEY=your-key
# OR
GEMINI_API_KEY=your-key
# OR
OPENAI_API_KEY=your-key

# Required: Storage (R2 or S3-compatible)
R2_ENDPOINT_URL=https://your-account.r2.cloudflarestorage.com
R2_ACCESS_KEY_ID=your-key
R2_SECRET_ACCESS_KEY=your-secret

# Optional but recommended
RESEND_API_KEY=your-email-service-key
```

### 2. Generate Secrets

Before running, generate some required secrets:

```bash
# Generate a random 32-byte hex string for DOWNLOAD_SIGNING_SECRET
openssl rand -hex 32

# Generate a random secret for USER_API_KEYS_ENCRYPTION_SECRET
openssl rand -hex 32
```

Add these to your `.env.docker` file.

### 3. Build and Run

```bash
# Build images and start services
docker-compose --env-file .env.docker up -d

# Check service status
docker-compose ps

# View logs
docker-compose logs -f

# Stop services
docker-compose down
```

## Usage

Once running:

- **Frontend**: Open `http://localhost:3000`
- **Backend API**: `http://localhost:3001`

The services will:
- Automatically restart if they crash
- Have health checks enabled
- Wait for backend to be ready before starting frontend
- Connect to your external Supabase instance

## Common Issues

### Database Connection Fails

Ensure `SUPABASE_URL` and `SUPABASE_SECRET_KEY` are correct. Run migrations if using a new database:

```bash
# Connect to your Supabase and run the schema
cat backend/schema.sql | psql your-database-url
```

### File Upload Fails

Check that R2/S3 credentials are correct:
- `R2_ENDPOINT_URL` should include the account ID
- `R2_ACCESS_KEY_ID` and `R2_SECRET_ACCESS_KEY` must be valid
- Ensure the bucket exists and has proper permissions

### AI Features Not Working

At least one provider API key is required:
- `ANTHROPIC_API_KEY` for Claude
- `GEMINI_API_KEY` for Google Gemini
- `OPENAI_API_KEY` for OpenAI

### Frontend Can't Connect to Backend

Ensure `NEXT_PUBLIC_API_BASE_URL` matches your backend URL. In docker-compose, use the service name:
```bash
NEXT_PUBLIC_API_BASE_URL=http://backend:3001
```

But when accessing from your browser, use:
```bash
NEXT_PUBLIC_API_BASE_URL=http://localhost:3001
```

## Production Deployment

For production, consider:

1. **Reverse Proxy**: Use Nginx or Traefik in front of both services
2. **SSL/TLS**: Configure HTTPS certificates
3. **Environment**: Use separate `.env.prod` file
4. **Resource Limits**: Add memory/CPU limits in docker-compose
5. **Backups**: Set up regular Supabase backups
6. **Monitoring**: Add health checks and logging

### Example Production Configuration

```bash
docker-compose \
  --env-file .env.prod \
  -f docker-compose.yml \
  -f docker-compose.prod.yml \
  up -d
```

## Customization

### Custom Ports

Edit ports in `docker-compose.yml` or override with:

```bash
BACKEND_PORT=8001 FRONTEND_PORT=8000 docker-compose up
```

### Use Different Dockerfile

If you need LibreOffice for DOC/DOCX conversion:

```bash
docker build -f Dockerfile.backend.libreoffice -t mike-backend:with-libreoffice .
```

### View Logs

```bash
# All services
docker-compose logs

# Specific service
docker-compose logs -f backend

# Last 100 lines
docker-compose logs --tail 100
```

## Cleanup

```bash
# Stop and remove containers
docker-compose down

# Remove volumes (be careful - deletes data)
docker-compose down -v

# Remove images
docker rmi mike-backend mike-frontend
```

## Network

Services communicate via the internal `mike-network`:
- Frontend can reach backend at `http://backend:3001`
- External access through exposed ports (3000, 3001)

## Health Checks

Both services have health checks:

```bash
# Check health status
docker ps

# Manual health check
curl http://localhost:3001/health  # backend
curl http://localhost:3000         # frontend
```

The frontend will only start after the backend is healthy.

## Environment Variables Reference

See `.env.docker.example` for the complete list of configurable variables.

## Troubleshooting

### Container won't start

Check logs:
```bash
docker-compose logs backend
docker-compose logs frontend
```

### Healthcheck failing

Wait a bit longer:
```bash
docker-compose ps  # Check if healthy
```

### Port already in use

Change ports in docker-compose.yml or `.env.docker`:
```bash
BACKEND_PORT=3002 FRONTEND_PORT=3001 docker-compose up
```

### Permission denied

On Linux, you might need to run docker commands with `sudo`:
```bash
sudo docker-compose up
```

Or add your user to the docker group:
```bash
sudo usermod -aG docker $USER
newgrp docker
```
