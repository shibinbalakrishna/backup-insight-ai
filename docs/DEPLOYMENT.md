# Deployment Guide

Comprehensive guide for deploying Backup Insight AI Platform to production.

## Deployment Architectures

### 1. Single Server Deployment (MVP)

**Best For:** Small teams, PoC, low traffic

```
┌─────────────────────────────────┐
│      Single Linux Server         │
│  (8GB RAM, 2 CPU, 100GB SSD)   │
├─────────────────────────────────┤
│  • All services in Docker       │
│  • Local PostgreSQL             │
│  • Single Nginx reverse proxy   │
│  • Redis cache                  │
└─────────────────────────────────┘
     │
     ▼
Multiple remote SSH systems
```

**Resource Requirements:**
- 8GB+ RAM
- 2+ CPU cores
- 100GB+ SSD storage
- Stable internet (SSH access to remote systems)

### 2. High Availability Setup (Production)

```
┌────────────────────────────────────────────┐
│         Load Balancer (HTTPS)              │
│          (HAProxy/Nginx)                   │
└──────────────────────────────────────────┬─┘
        │
   ┌────┴──────┬──────────┐
   ▼           ▼          ▼
┌─────────┐ ┌──────────┐ ┌──────────┐
│ Backend │ │ Backend  │ │ Backend  │
│ Node 1  │ │ Node 2   │ │ Node 3   │
└────┬────┘ └────┬─────┘ └────┬─────┘
     │           │            │
     └───────────┼────────────┘
                 ▼
         ┌──────────────────┐
         │ PostgreSQL HA    │
         │ (Primary-Replica)│
         └────────┬─────────┘
                  │
        ┌─────────┴─────────┐
        ▼                   ▼
     Primary           Standby
    (Read/Write)       (Read)
```

**Resource Requirements per Node:**
- 16GB+ RAM
- 4+ CPU cores
- 200GB+ SSD storage

### 3. Kubernetes Deployment (Scale)

See `/docs/kubernetes-deployment.md` for detailed K8s setup.

---

## Pre-Deployment Checklist

- [ ] Server provisioning complete
- [ ] SSH access verified
- [ ] Docker & Docker Compose installed
- [ ] Firewall rules configured
- [ ] SSL certificates acquired
- [ ] Domain name configured
- [ ] Database backup strategy planned
- [ ] Monitoring solution selected
- [ ] Incident response plan created

---

## Single Server Deployment Steps

### Step 1: Server Setup

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Verify installation
docker --version
docker-compose --version

# Create app directory
sudo mkdir -p /opt/backup-ai
sudo chown $USER:$USER /opt/backup-ai
```

### Step 2: Application Setup

```bash
cd /opt/backup-ai

# Clone repository
git clone https://github.com/shibinbalakrishna/backup-insight-ai.git .

# Copy environment file
cp .env.example .env

# Edit configuration
nano .env
```

### Step 3: Configure Environment

```env
# Database
DATABASE_URL=postgresql://user:password@postgres:5432/backup_ai
POSTGRES_PASSWORD=your_secure_password

# JWT
SECRET_KEY=your-super-secret-key-min-32-chars

# Redis
REDIS_PASSWORD=your_redis_password

# Ollama
OLLAMA_URL=http://ollama:11434
OLLAMA_MODEL=llama2

# Teams
TEAMS_WEBHOOK_URL=https://outlook.webhook.office.com/...

# CORS
ALLOWED_ORIGINS=["https://yourdomain.com"]

# Features
ENABLE_PREDICTIVE_ANALYTICS=true
ENABLE_ANOMALY_DETECTION=true
```

### Step 4: Start Services

```bash
# Start all services
docker-compose up -d

# Verify services
docker-compose ps

# Check logs
docker-compose logs -f backend

# Wait for services to be healthy (2-3 minutes)
```

### Step 5: Initialize Database

```bash
# Run migrations
docker-compose exec backend python -m alembic upgrade head

# Create admin user
docker-compose exec backend python scripts/create_admin.py

# Seed sample data (optional)
docker-compose exec backend python scripts/seed_data.py
```

### Step 6: SSL/TLS Setup

```bash
# Generate self-signed certificate (dev)
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /opt/backup-ai/nginx/ssl/nginx.key \
  -out /opt/backup-ai/nginx/ssl/nginx.crt

# Or use Let's Encrypt (production)
sudo apt install certbot python3-certbot-nginx -y
sudo certbot certonly --standalone -d yourdomain.com

# Copy certificates
sudo cp /etc/letsencrypt/live/yourdomain.com/fullchain.pem /opt/backup-ai/nginx/ssl/
sudo cp /etc/letsencrypt/live/yourdomain.com/privkey.pem /opt/backup-ai/nginx/ssl/
```

### Step 7: Nginx Configuration

Update `nginx/nginx.conf`:

```nginx
server {
    listen 443 ssl http2;
    server_name yourdomain.com;

    ssl_certificate /etc/nginx/ssl/nginx.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx.key;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Proxy to backend
    location /api {
        proxy_pass http://backend:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Proxy to frontend
    location / {
        proxy_pass http://frontend:3000;
        proxy_set_header Host $host;
    }
}
```

### Step 8: Restart Services

```bash
# Reload Nginx
docker-compose exec nginx nginx -s reload

# Verify connectivity
curl -k https://yourdomain.com
```

---

## Backup Strategy

### Database Backups

```bash
# Create backup directory
mkdir -p /opt/backup-ai/backups

# Daily backup script
cat > /opt/backup-ai/scripts/backup-db.sh << 'EOF'
#!/bin/bash
BACKUP_DIR="/opt/backup-ai/backups"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
BACKUP_FILE="$BACKUP_DIR/backup_$TIMESTAMP.sql"

docker-compose exec -T postgres pg_dump -U user backup_ai > $BACKUP_FILE
gzip $BACKUP_FILE

# Keep last 30 days
find $BACKUP_DIR -name "backup_*.sql.gz" -mtime +30 -delete
EOF

chmod +x /opt/backup-ai/scripts/backup-db.sh

# Add to crontab
(crontab -l 2>/dev/null; echo "0 2 * * * /opt/backup-ai/scripts/backup-db.sh") | crontab -
```

### Volume Backups

```bash
# Backup Docker volumes
docker-compose exec -T postgres tar czf - /var/lib/postgresql/data | \
  gzip > /opt/backup-ai/backups/postgres-data-$(date +%Y%m%d).tar.gz
```

---

## Monitoring & Logging

### Docker Logs

```bash
# View service logs
docker-compose logs -f backend

# View specific container
docker-compose logs postgres

# Clear logs
docker-compose logs --follow --tail=100
```

### System Monitoring

```bash
# Install monitoring tools
sudo apt install htop iotop nethogs -y

# Monitor container resources
docker stats

# Monitor disk usage
df -h
du -sh /opt/backup-ai/*
```

### Prometheus Setup (Optional)

```yaml
# Add to docker-compose.yml
prometheus:
  image: prom/prometheus
  ports:
    - "9090:9090"
  volumes:
    - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
    - prometheus_data:/prometheus
```

---

## Maintenance & Updates

### Regular Maintenance

```bash
# Weekly system updates
sudo apt update && sudo apt upgrade -y

# Clean unused Docker images
docker image prune -a

# Clean unused volumes
docker volume prune

# Clean unused networks
docker network prune

# View disk usage
docker system df
```

### Application Updates

```bash
# Pull latest code
cd /opt/backup-ai
git pull origin main

# Rebuild images
docker-compose build

# Restart services
docker-compose down
docker-compose up -d

# Run migrations
docker-compose exec backend python -m alembic upgrade head
```

### Database Maintenance

```bash
# Optimize database
docker-compose exec postgres psql -U user -d backup_ai -c "VACUUM ANALYZE;"

# Check index health
docker-compose exec postgres psql -U user -d backup_ai -c "\d+ tablename"

# Monitor connections
docker-compose exec postgres psql -U user -d backup_ai -c "SELECT datname, count(*) FROM pg_stat_activity GROUP BY datname;"
```

---

## Troubleshooting

### Service Won't Start

```bash
# Check logs
docker-compose logs backend

# Check ports
sudo netstat -tulpn | grep LISTEN

# Free up port
sudo lsof -i :8000

# Restart Docker
sudo systemctl restart docker
docker-compose down
docker-compose up -d
```

### Database Connection Error

```bash
# Test connection
docker-compose exec postgres psql -U user -d backup_ai -c "SELECT 1;"

# Check password
cat .env | grep DATABASE_URL

# Reset database
docker-compose down -v  # ⚠️ WARNING: Deletes data
docker-compose up -d
```

### High Memory Usage

```bash
# Check container memory
docker stats

# Increase limits (docker-compose.yml)
services:
  backend:
    mem_limit: 2G
    memswap_limit: 3G

# Restart
docker-compose down
docker-compose up -d
```

---

## Security Hardening

### Firewall Setup

```bash
# UFW (Ubuntu Firewall)
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp      # SSH
sudo ufw allow 80/tcp      # HTTP
sudo ufw allow 443/tcp     # HTTPS
sudo ufw enable
```

### SSH Hardening

```bash
# Disable password auth (use keys only)
sudo nano /etc/ssh/sshd_config

# Set:
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes

# Restart SSH
sudo systemctl restart sshd
```

### Container Security

```bash
# Run containers as non-root
# Edit docker-compose.yml
user: "1000:1000"

# Enable read-only filesystem
read_only: true
tmpfs:
  - /tmp
  - /run
```

---

## Scaling Considerations

### Before Scaling to HA

1. ✅ Single server runs stable for 1+ week
2. ✅ Backup strategy tested
3. ✅ Monitoring in place
4. ✅ Incident response plan created
5. ✅ Load testing completed

### Horizontal Scaling

```
API Load: > 1000 req/sec → Add backend nodes
Data: > 500GB → Add storage
Concurrent Systems: > 1000 → Add worker nodes
```

---

## Support & Help

- 📖 Documentation: https://github.com/shibinbalakrishna/backup-insight-ai/docs
- 💬 Discussions: GitHub Discussions
- 🐛 Issues: GitHub Issues
- 📧 Email: support@example.com

---

**Last Updated**: 2026-05-14
