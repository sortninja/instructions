# Supabase Installation Guide for Debian 13

## Complete Walkthrough: Installing Supabase with Docker Compose

This guide will walk you through setting up Supabase on a fresh Debian 13 installation using Docker Compose.

---

## Prerequisites

- Fresh Debian 13 installation
- Root or sudo access
- At least 4GB RAM (8GB recommended)
- 20GB+ free disk space
- Internet connection

---

## Step 1: Update System Packages

First, ensure your system is up to date:

```bash
sudo apt update && sudo apt upgrade -y
```

---

## Step 2: Install Required Dependencies

Install necessary packages for Docker installation:

```bash
sudo apt install -y ca-certificates curl gnupg lsb-release git
```

---

## Step 3: Install Docker

### Add Docker's Official GPG Key

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

### Add Docker Repository

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Install Docker Engine

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Verify Docker Installation

```bash
sudo docker --version
sudo docker compose version
```

### Add Your User to Docker Group (Optional)

This allows running Docker without sudo:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

**Note:** Log out and back in for group changes to take full effect.

---

## Step 4: Install Supabase

### Clone the Supabase Repository

```bash
cd ~
git clone --depth 1 https://github.com/supabase/supabase
cd supabase/docker
```

### Copy Environment File

```bash
cp .env.example .env
```

---

## Step 5: Configure Supabase

### Generate Secure Secrets

You need to generate secure JWT secrets and passwords. Use the following commands:

```bash
# Generate JWT Secret (should be at least 32 characters)
openssl rand -base64 32

# Generate Postgres password
openssl rand -base64 32

# Generate service role key (JWT)
openssl rand -base64 32
```

### Edit the .env File

Open the `.env` file in your preferred text editor:

```bash
nano .env
```

### Update Critical Environment Variables

Locate and update the following variables with your generated secrets:

```bash
############
# Secrets
############
POSTGRES_PASSWORD=your_generated_postgres_password_here
JWT_SECRET=your_generated_jwt_secret_here
ANON_KEY=your_generated_anon_key_here
SERVICE_ROLE_KEY=your_generated_service_role_key_here

############
# Database
############
POSTGRES_HOST=db
POSTGRES_DB=postgres
POSTGRES_PORT=5432

############
# API
############
KONG_HTTP_PORT=8000
KONG_HTTPS_PORT=8443

############
# Studio
############
STUDIO_PORT=3000

############
# Site URL
############
SITE_URL=http://localhost:3000
API_EXTERNAL_URL=http://localhost:8000
```

### Important Configuration Notes

1. **POSTGRES_PASSWORD**: Strong password for PostgreSQL database
2. **JWT_SECRET**: Used to sign JWT tokens (minimum 32 characters)
3. **ANON_KEY**: Public API key for client-side requests
4. **SERVICE_ROLE_KEY**: Admin API key with full access (keep secure!)
5. **SITE_URL**: Your application's URL (update for production)

**Security Warning:** Never commit your `.env` file to version control or share these secrets publicly.

---

## Step 6: Start Supabase

### Pull Docker Images

```bash
docker compose pull
```

### Start All Services

```bash
docker compose up -d
```

This will start all Supabase services in detached mode.

### Verify Services Are Running

```bash
docker compose ps
```

You should see all services in a "running" state.

---

## Step 7: Access Supabase Studio

Once all services are running, you can access:

- **Supabase Studio (Dashboard)**: http://localhost:3000
- **API Gateway (Kong)**: http://localhost:8000
- **Database**: localhost:5432

### Default Studio Credentials

On first access, you may need to use:
- **Email**: Set in your `.env` (or default)
- **Password**: Set in your `.env` (or default)

---

## Step 8: Verify Installation

### Check Service Health

```bash
# View logs
docker compose logs

# Check specific service logs
docker compose logs kong
docker compose logs postgres
docker compose logs studio
```

### Test API Connection

```bash
curl http://localhost:8000
```

You should receive a response from the Kong API gateway.

---

## Common Commands

### Stop Supabase

```bash
docker compose stop
```

### Start Supabase

```bash
docker compose start
```

### Restart Supabase

```bash
docker compose restart
```

### Stop and Remove Containers

```bash
docker compose down
```

### Stop and Remove Containers + Volumes (Full Reset)

**Warning:** This deletes all data!

```bash
docker compose down -v
```

### View Real-Time Logs

```bash
docker compose logs -f
```

### Update Supabase

```bash
cd ~/supabase/docker
git pull origin master
docker compose pull
docker compose up -d
```

---

## Troubleshooting

### Services Won't Start

1. Check if ports are already in use:
   ```bash
   sudo netstat -tulpn | grep -E '(3000|8000|5432)'
   ```

2. View service logs:
   ```bash
   docker compose logs
   ```

### Permission Denied Errors

Ensure your user is in the docker group:
```bash
sudo usermod -aG docker $USER
newgrp docker
```

### Database Connection Issues

1. Verify PostgreSQL is running:
   ```bash
   docker compose ps postgres
   ```

2. Check PostgreSQL logs:
   ```bash
   docker compose logs postgres
   ```

### Out of Memory Errors

Increase available RAM or adjust Docker memory limits in `/etc/docker/daemon.json`.

---

## Production Considerations

### Security Hardening

1. **Change all default passwords and secrets**
2. **Enable HTTPS** using a reverse proxy (nginx/Caddy)
3. **Configure firewall rules** to restrict access
4. **Use strong JWT secrets** (64+ characters)
5. **Regularly update** Supabase and Docker images
6. **Backup your database** regularly
7. **Enable monitoring** and logging

### Reverse Proxy Setup (Nginx Example)

For production, you should use a reverse proxy with SSL:

```nginx
server {
    listen 80;
    server_name yourdomain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name yourdomain.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /api/ {
        proxy_pass http://localhost:8000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Backup Database

```bash
docker compose exec postgres pg_dump -U postgres postgres > backup.sql
```

### Restore Database

```bash
docker compose exec -T postgres psql -U postgres postgres < backup.sql
```

---

## Resources

- **Official Documentation**: https://supabase.com/docs
- **GitHub Repository**: https://github.com/supabase/supabase
- **Community Discord**: https://discord.supabase.com
- **Docker Documentation**: https://docs.docker.com

---

## Summary

You now have a fully functional Supabase instance running on Debian 13 with Docker Compose! The setup includes:

- PostgreSQL database
- RESTful API (PostgREST)
- Authentication service (GoTrue)
- Storage service
- Real-time subscriptions
- Supabase Studio (Web UI)

Access your Supabase Studio at http://localhost:3000 and start building your applications!

---

**Last Updated**: February 2026
**Supabase Version**: Latest (self-hosted)
**Debian Version**: 13
