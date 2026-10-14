# Deployment Guide

This guide covers deploying your Docker Compose stack to production on common hosting providers.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [AWS EC2 Deployment](#aws-ec2-deployment)
3. [DigitalOcean Droplet Deployment](#digitalocean-droplet-deployment)
4. [Hetzner Cloud Deployment](#hetzner-cloud-deployment)
5. [Security Best Practices](#security-best-practices)
6. [SSL/TLS Setup](#ssltls-setup)
7. [Monitoring Setup](#monitoring-setup)
8. [Backup Strategy](#backup-strategy)
9. [Troubleshooting](#troubleshooting)

## Prerequisites

- Docker 24.0+ and Docker Compose 2.0+ installed on server
- Domain name pointing to your server
- SSH access to server
- At least 2GB RAM (4GB recommended)

## AWS EC2 Deployment

### 1. Launch EC2 Instance

```bash
# Amazon Linux 2 AMI
# Instance type: t3.medium (2 vCPU, 4GB RAM)
# Security group: Allow ports 22, 80, 443
```

### 2. Install Docker

```bash
ssh ec2-user@your-server-ip

# Install Docker
sudo yum update -y
sudo yum install -y docker
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Log out and back in for group changes to take effect
exit
```

### 3. Deploy Stack

```bash
ssh ec2-user@your-server-ip

# Clone your code or upload files
git clone https://github.com/yourusername/yourproject.git
cd yourproject

# Create secrets directory
mkdir -p secrets
echo "your_strong_password_here" > secrets/db_password.txt
echo "your_secret_key_here" > secrets/secret_key.txt
chmod 600 secrets/*

# Start services
docker-compose -f lamp-stack.yml up -d

# Check logs
docker-compose -f lamp-stack.yml logs -f
```

### 4. Configure DNS

Point your domain A record to your EC2 instance's Elastic IP.

## DigitalOcean Droplet Deployment

### 1. Create Droplet

```bash
# Use Docker marketplace image
# Size: Basic, $24/mo (2 vCPU, 4GB RAM)
# Region: Choose closest to your users
```

### 2. Initial Setup

```bash
ssh root@your-droplet-ip

# Update system
apt-get update && apt-get upgrade -y

# Docker is pre-installed on marketplace image
docker --version
docker-compose --version
```

### 3. Deploy Stack

```bash
# Upload your project files
scp -r ./docker-compose-starter-kit root@your-droplet-ip:/opt/

ssh root@your-droplet-ip
cd /opt/docker-compose-starter-kit

# Create secrets
mkdir -p secrets
openssl rand -base64 32 > secrets/db_password.txt
openssl rand -base64 64 > secrets/secret_key.txt
chmod 600 secrets/*

# Start services
docker-compose -f wordpress-stack.yml up -d

# Enable firewall
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw enable
```

## Hetzner Cloud Deployment

### 1. Create Server

```bash
# CX21: 2 vCPU, 4GB RAM, €4.90/mo
# OS: Ubuntu 22.04
# Datacenter: Choose closest to your users
```

### 2. Install Docker

```bash
ssh root@your-server-ip

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# Install Docker Compose
apt-get install -y docker-compose-plugin

# Verify installation
docker --version
docker compose version
```

### 3. Deploy Stack

```bash
# Upload files
scp -r ./docker-compose-starter-kit root@your-server-ip:/opt/

ssh root@your-server-ip
cd /opt/docker-compose-starter-kit

# Generate secrets
mkdir -p secrets
tr -dc A-Za-z0-9 </dev/urandom | head -c 32 > secrets/db_password.txt
tr -dc A-Za-z0-9 </dev/urandom | head -c 64 > secrets/secret_key.txt
chmod 600 secrets/*

# Start services
docker compose -f rails-stack.yml up -d
```

## Security Best Practices

### 1. Firewall Configuration

```bash
# UFW (Ubuntu/Debian)
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw enable

# Firewalld (CentOS/RHEL)
firewall-cmd --permanent --add-service=ssh
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
```

### 2. SSH Hardening

```bash
# Edit /etc/ssh/sshd_config
sudo nano /etc/ssh/sshd_config

# Disable root login
PermitRootLogin no

# Disable password auth (use SSH keys)
PasswordAuthentication no

# Restart SSH
sudo systemctl restart sshd
```

### 3. Automatic Security Updates

```bash
# Ubuntu/Debian
sudo apt-get install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades

# CentOS/RHEL
sudo yum install -y yum-cron
sudo systemctl enable --now yum-cron
```

## SSL/TLS Setup

### Using Let's Encrypt (Recommended)

```bash
# Install Certbot
sudo apt-get install -y certbot python3-certbot-nginx

# Generate certificate
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com

# Auto-renewal (already set up by certbot)
sudo certbot renew --dry-run
```

### Nginx SSL Configuration

Create `nginx/conf.d/ssl.conf`:

```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name yourdomain.com www.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    location / {
        proxy_pass http://app:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## Monitoring Setup

### Access Grafana

1. Open `http://your-server-ip:3000`
2. Login with admin / (password from secrets/grafana_password.txt)
3. Add Prometheus data source: `http://prometheus:9090`
4. Import dashboards from `grafana/provisioning/dashboards/`

### Access Prometheus

1. Open `http://your-server-ip:9090`
2. Query metrics, set up alerts
3. Targets: `http://your-server-ip:9090/targets`

### Health Check Monitoring

```bash
# Check all services are healthy
docker-compose -f lamp-stack.yml ps

# Check specific service logs
docker-compose -f lamp-stack.yml logs -f mysql

# Monitor resource usage
docker stats
```

## Backup Strategy

### Automated Database Backups

The backup service runs daily at midnight UTC. Backups are stored in named volumes.

**View backups:**

```bash
# List backup volume contents
docker run --rm -v postgres_backups:/backups alpine ls -lh /backups
```

**Restore from backup:**

```bash
# Stop application
docker-compose -f rails-stack.yml stop rails sidekiq

# Restore database
docker run --rm \
  -v postgres_backups:/backups \
  -v postgres_data:/var/lib/postgresql/data \
  postgres:16-alpine \
  sh -c "pg_restore -U postgres -d rails_production /backups/backup-YYYY-MM-DD.dump"

# Start application
docker-compose -f rails-stack.yml start rails sidekiq
```

### Volume Backups

```bash
# Backup all volumes to tar
docker run --rm \
  -v mysql_data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/mysql-data-$(date +%Y%m%d).tar.gz -C /data .

# Restore volume from tar
docker run --rm \
  -v mysql_data:/data \
  -v $(pwd):/backup \
  alpine sh -c "cd /data && tar xzf /backup/mysql-data-20241014.tar.gz"
```

### Off-site Backup (S3)

```bash
# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Configure AWS credentials
aws configure

# Sync backups to S3 (daily cron)
aws s3 sync /var/lib/docker/volumes/ s3://your-backup-bucket/docker-volumes/ \
  --exclude "*" --include "*backups*"
```

## Troubleshooting

### Services Won't Start

```bash
# Check logs
docker-compose -f lamp-stack.yml logs

# Check specific service
docker-compose -f lamp-stack.yml logs mysql

# Inspect container
docker inspect mysql
```

### Database Connection Errors

```bash
# Verify database is healthy
docker-compose -f lamp-stack.yml ps mysql

# Test connection
docker-compose -f lamp-stack.yml exec mysql mysqladmin ping

# Check credentials
cat secrets/db_password.txt
```

### Port Already in Use

```bash
# Find process using port 80
sudo lsof -i :80

# Kill process (if safe)
sudo kill -9 <PID>

# Or change port in docker-compose.yml
ports:
  - "8080:80"  # Use port 8080 instead
```

### Out of Disk Space

```bash
# Check disk usage
df -h

# Clean up Docker
docker system prune -a --volumes

# Remove old images
docker image prune -a

# Remove stopped containers
docker container prune
```

### High Memory Usage

```bash
# Check container resource usage
docker stats

# Set memory limits in docker-compose.yml
services:
  mysql:
    mem_limit: 1g
    mem_reservation: 512m
```

### SSL Certificate Issues

```bash
# Verify certificate
sudo certbot certificates

# Renew certificate manually
sudo certbot renew

# Check Nginx config
sudo nginx -t

# Reload Nginx
sudo nginx -s reload
```

## Performance Optimization

### Database Tuning

Edit `mysql/my.cnf`:

```ini
[mysqld]
max_connections = 200
innodb_buffer_pool_size = 2G
innodb_log_file_size = 256M
```

### Nginx Caching

Add to `nginx/conf.d/default.conf`:

```nginx
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m max_size=1g inactive=60m;

location / {
    proxy_cache my_cache;
    proxy_cache_valid 200 60m;
    proxy_cache_use_stale error timeout http_500 http_502 http_503 http_504;
}
```

### Container Resource Limits

```yaml
services:
  app:
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G
        reservations:
          cpus: '1'
          memory: 1G
```

## Support

For issues specific to these templates, email: n.mcisaac@iqcn.live

For general Docker Compose questions, see: https://docs.docker.com/compose/
