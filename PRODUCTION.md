# Production Deployment Plan

## Overview

This document outlines the deployment plan for the Trade Journal application using **AWS EC2 + RDS PostgreSQL**.

## Architecture Decision

**Chosen: EC2 + RDS (Option B)**

```
                    ┌─────────────────────────────────────────┐
                    │              AWS VPC                     │
                    │                                          │
Internet ──────────►│  ┌─────────────────────────────────┐    │
        HTTPS:443   │  │     EC2 (Public Subnet)         │    │
                    │  │  ┌─────────────────────────┐    │    │
                    │  │  │   Nginx (SSL/Proxy)     │    │    │
                    │  │  └──────────┬──────────────┘    │    │
                    │  │             │                    │    │
                    │  │  ┌─────────┴─────────┐          │    │
                    │  │  │                   │          │    │
                    │  │  ▼                   ▼          │    │
                    │  │ ┌────────┐    ┌───────────┐    │    │
                    │  │ │  API   │    │ Frontend  │    │    │
                    │  │ │ :8080  │    │  :3000    │    │    │
                    │  │ └───┬────┘    └───────────┘    │    │
                    │  │     │                           │    │
                    │  │     ▼                           │    │
                    │  │ ┌────────┐    ┌────────┐       │    │
                    │  │ │ Worker │◄───│ Redis  │       │    │
                    │  │ └───┬────┘    └────────┘       │    │
                    │  └─────┼───────────────────────────┘    │
                    │        │                                 │
                    │        ▼                                 │
                    │  ┌─────────────────────────────────┐    │
                    │  │    RDS PostgreSQL               │    │
                    │  │    (Private Subnet)             │    │
                    │  └─────────────────────────────────┘    │
                    └─────────────────────────────────────────┘
```

---

## Step-by-Step Deployment Guide

### Phase 1: AWS Infrastructure Setup

#### 1.1 Prerequisites
- [ ] AWS Account created
- [ ] AWS CLI installed and configured
- [ ] Domain name registered (or ready to use)
- [ ] Set up billing alerts ($50 threshold recommended)

#### 1.2 VPC & Networking
- [ ] Create VPC (10.0.0.0/16)
- [ ] Create public subnet (10.0.1.0/24) - for EC2
- [ ] Create private subnet (10.0.2.0/24) - for RDS
- [ ] Create Internet Gateway, attach to VPC
- [ ] Create route table for public subnet (0.0.0.0/0 → IGW)

#### 1.3 Security Groups

**EC2 Security Group (sg-ec2):**
| Type | Port | Source | Description |
|------|------|--------|-------------|
| SSH | 22 | Your IP | Admin access |
| HTTP | 80 | 0.0.0.0/0 | Redirect to HTTPS |
| HTTPS | 443 | 0.0.0.0/0 | Web traffic |

**RDS Security Group (sg-rds):**
| Type | Port | Source | Description |
|------|------|--------|-------------|
| PostgreSQL | 5432 | sg-ec2 | Only from EC2 |

#### 1.4 RDS PostgreSQL Setup
- [ ] Create RDS subnet group (private subnets)
- [ ] Launch RDS instance:
  - Engine: PostgreSQL 15
  - Instance: `db.t3.micro` (free tier) or `db.t3.small`
  - Storage: 20GB gp3, auto-scaling to 100GB
  - Multi-AZ: No (cost savings)
  - Public access: No
  - Security group: sg-rds
  - Initial database: `tradejournal`
- [ ] Note the RDS endpoint URL

#### 1.5 EC2 Instance Setup
- [ ] Launch EC2 instance:
  - AMI: Amazon Linux 2023 or Ubuntu 22.04
  - Instance: `t3.small` (2 vCPU, 2GB RAM)
  - Subnet: Public subnet
  - Security group: sg-ec2
  - Storage: 30GB gp3
- [ ] Allocate and associate Elastic IP
- [ ] Note the Elastic IP address

---

### Phase 2: Domain & DNS

- [ ] Point domain A record to Elastic IP
- [ ] Wait for DNS propagation (check with `dig yourdomain.com`)

---

### Phase 3: EC2 Server Configuration

SSH into EC2 and run:

```bash
# Update system
sudo yum update -y  # Amazon Linux
# OR
sudo apt update && sudo apt upgrade -y  # Ubuntu

# Install Docker
sudo yum install docker -y  # Amazon Linux
# OR
sudo apt install docker.io -y  # Ubuntu

sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Install Nginx
sudo yum install nginx -y  # Amazon Linux
# OR
sudo apt install nginx -y  # Ubuntu

# Install Certbot
sudo yum install certbot python3-certbot-nginx -y  # Amazon Linux
# OR
sudo apt install certbot python3-certbot-nginx -y  # Ubuntu

# Log out and back in for docker group to take effect
exit
```

---

### Phase 4: Application Deployment

#### 4.1 Create deployment directory

```bash
mkdir -p ~/trade-journal
cd ~/trade-journal
```

#### 4.2 Create docker-compose.prod.yml

```yaml
version: "3.8"

services:
  api:
    image: ghcr.io/YOUR_USERNAME/trade-journal-api:latest
    restart: unless-stopped
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=redis://redis:6379
      - JWT_SECRET=${JWT_SECRET}
      - GIN_MODE=release
      - PORT=8080
    ports:
      - "8080:8080"
    depends_on:
      - redis
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  worker:
    image: ghcr.io/YOUR_USERNAME/trade-journal-worker:latest
    restart: unless-stopped
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=redis://redis:6379
    depends_on:
      - redis
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3

  frontend:
    image: ghcr.io/YOUR_USERNAME/trade-journal-frontend:latest
    restart: unless-stopped
    ports:
      - "3000:80"
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  redis_data:
```

#### 4.3 Create .env.production

```bash
# Database (RDS)
DATABASE_URL=postgres://username:password@your-rds-endpoint.region.rds.amazonaws.com:5432/tradejournal?sslmode=require

# Security
JWT_SECRET=your-secure-random-string-at-least-32-chars

# Optional
BINANCE_API_KEY=your-binance-key
```

#### 4.4 Create Nginx configuration

```bash
sudo nano /etc/nginx/conf.d/tradejournal.conf
```

```nginx
# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;
    return 301 https://$server_name$request_uri;
}

# Main HTTPS server
server {
    listen 443 ssl http2;
    server_name yourdomain.com www.yourdomain.com;

    # SSL certificates (managed by certbot)
    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    # SSL settings
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # Gzip compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;
    gzip_min_length 1000;

    # API proxy
    location /api/ {
        proxy_pass http://localhost:8080;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # Frontend
    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Cache static assets
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
            proxy_pass http://localhost:3000;
            expires 1y;
            add_header Cache-Control "public, immutable";
        }
    }
}
```

#### 4.5 Get SSL Certificate

```bash
# Stop nginx temporarily
sudo systemctl stop nginx

# Get certificate (standalone mode)
sudo certbot certonly --standalone -d yourdomain.com -d www.yourdomain.com

# Start nginx
sudo systemctl start nginx
sudo systemctl enable nginx

# Set up auto-renewal
sudo systemctl enable certbot-renew.timer
```

---

### Phase 5: Deploy Application

```bash
cd ~/trade-journal

# Load environment variables
export $(cat .env.production | xargs)

# Pull and start containers
docker-compose -f docker-compose.prod.yml pull
docker-compose -f docker-compose.prod.yml up -d

# Check status
docker-compose -f docker-compose.prod.yml ps
docker-compose -f docker-compose.prod.yml logs -f
```

---

### Phase 6: Database Migration

```bash
# Run migrations (if not auto-run)
docker-compose -f docker-compose.prod.yml exec api ./migrate up
```

---

### Phase 7: CI/CD Setup

Update `.github/workflows/ci.yml` to add deployment:

```yaml
  deploy:
    needs: [api-build, frontend-build, worker-build]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - name: Deploy to EC2
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            cd ~/trade-journal
            docker-compose -f docker-compose.prod.yml pull
            docker-compose -f docker-compose.prod.yml up -d --remove-orphans
            docker system prune -f
```

Add GitHub Secrets:
- `EC2_HOST`: Your Elastic IP
- `EC2_USER`: `ec2-user` or `ubuntu`
- `EC2_SSH_KEY`: Private SSH key

---

## Execution Checklist

| # | Task | Status |
|---|------|--------|
| 1 | Create AWS account, set billing alerts | [ ] |
| 2 | Create VPC with public/private subnets | [ ] |
| 3 | Create security groups (EC2, RDS) | [ ] |
| 4 | Launch RDS PostgreSQL instance | [ ] |
| 5 | Launch EC2 instance with Elastic IP | [ ] |
| 6 | Point domain DNS to Elastic IP | [ ] |
| 7 | SSH to EC2, install Docker/Nginx/Certbot | [ ] |
| 8 | Get SSL certificate with Certbot | [ ] |
| 9 | Configure Nginx reverse proxy | [ ] |
| 10 | Create docker-compose.prod.yml | [ ] |
| 11 | Create .env.production with secrets | [ ] |
| 12 | Update CI/CD to push to GHCR | [ ] |
| 13 | Deploy containers | [ ] |
| 14 | Run database migrations | [ ] |
| 15 | Test application end-to-end | [ ] |
| 16 | Set up GitHub Secrets for auto-deploy | [ ] |
| 17 | Configure RDS automated backups | [ ] |

---

## Cost Estimate

| Service | Spec | Monthly Cost |
|---------|------|--------------|
| EC2 | t3.small (2 vCPU, 2GB) | ~$15 |
| RDS | db.t3.micro (free tier eligible) | ~$13 |
| Elastic IP | (attached to running instance) | $0 |
| EBS | 30GB gp3 | ~$3 |
| Route 53 | Hosted zone | ~$0.50 |
| Data Transfer | ~10GB out | ~$1 |
| **Total** | | **~$30-35/month** |

*Note: RDS db.t3.micro is free tier eligible for 12 months*

---

## Maintenance Runbook

### View logs
```bash
docker-compose -f docker-compose.prod.yml logs -f api
docker-compose -f docker-compose.prod.yml logs -f worker
```

### Restart services
```bash
docker-compose -f docker-compose.prod.yml restart api
```

### Deploy new version
```bash
docker-compose -f docker-compose.prod.yml pull
docker-compose -f docker-compose.prod.yml up -d
```

### Rollback
```bash
docker-compose -f docker-compose.prod.yml pull ghcr.io/user/api:previous-tag
docker-compose -f docker-compose.prod.yml up -d
```

### Database backup (manual)
```bash
pg_dump -h your-rds-endpoint -U username -d tradejournal > backup_$(date +%Y%m%d).sql
```

### SSL certificate renewal
```bash
sudo certbot renew --dry-run  # Test
sudo certbot renew            # Actual renewal
```

---

## Security Checklist

- [ ] SSH key-only authentication (no password)
- [ ] Security groups restrict access appropriately
- [ ] RDS not publicly accessible
- [ ] All traffic over HTTPS
- [ ] Secrets not in code or Docker images
- [ ] Regular security updates (`yum update`)
- [ ] CloudWatch alarms for unusual activity (optional)

---

## Future Improvements

1. **Add CloudWatch monitoring** - CPU, memory, disk alerts
2. **Migrate Redis to ElastiCache** - Better reliability
3. **Add Application Load Balancer** - Prepare for scaling
4. **Move to ECS/Fargate** - Auto-scaling, no server management
5. **Add WAF** - Web Application Firewall for security
