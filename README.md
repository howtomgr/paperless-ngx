# Paperless-ngx Installation Guide

Paperless-ngx is a free and open-source document management system that transforms your physical documents into a searchable digital archive. It features OCR, automatic tagging, full-text search, and a modern web interface for managing your documents.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Supported Operating Systems](#supported-operating-systems)
3. [Installation](#installation)
4. [Configuration](#configuration)
5. [Service Management](#service-management)
6. [Troubleshooting](#troubleshooting)
7. [Security Considerations](#security-considerations)
8. [Performance Tuning](#performance-tuning)
9. [Backup and Restore](#backup-and-restore)
10. [System Requirements](#system-requirements)
11. [Support](#support)
12. [Contributing](#contributing)
13. [License](#license)
14. [Acknowledgments](#acknowledgments)
15. [Version History](#version-history)
16. [Appendices](#appendices)

## 1. Prerequisites

- Operating system: Linux (preferred), macOS, or Windows (WSL)
- Python 3.9 or later
- PostgreSQL 11+ or MariaDB 10.3+ (SQLite for testing only)
- Redis 6.0+ for task queue
- Tesseract OCR 4.0+ with language packs
- Minimum 2GB RAM (4GB+ recommended)
- Storage space for documents
- ImageMagick, Ghostscript, and other document processing tools


## 2. Supported Operating Systems

This guide supports installation on:
- RHEL 8/9 and derivatives (CentOS Stream, Rocky Linux, AlmaLinux)
- Debian 11/12
- Ubuntu 20.04/22.04/24.04 LTS
- Arch Linux (rolling release)
- Alpine Linux 3.18+
- openSUSE Leap 15.5+ / Tumbleweed
- SUSE Linux Enterprise Server (SLES) 15+
- macOS 12+ (Monterey and later) 
- FreeBSD 13+
- Windows 10/11/Server 2019+ (where applicable)

## 3. Installation

### RHEL/CentOS/Rocky Linux/AlmaLinux

```bash
# Enable required repositories
sudo dnf install -y epel-release
sudo dnf config-manager --set-enabled powertools  # CentOS/Rocky 8
sudo dnf config-manager --set-enabled crb         # AlmaLinux 9

# Install system dependencies
sudo dnf install -y \
    python39 python39-pip python39-devel \
    postgresql postgresql-server postgresql-contrib \
    redis \
    tesseract tesseract-langpack-eng tesseract-langpack-deu \
    poppler-utils ImageMagick ghostscript \
    unpaper optipng pngquant jbig2dec \
    gcc gcc-c++ make \
    libpq-devel zlib-devel libjpeg-devel \
    libwebp-devel libxml2-devel libxslt-devel \
    git

# Additional OCR languages (optional)
sudo dnf install -y tesseract-langpack-fra tesseract-langpack-spa tesseract-langpack-ita

# Initialize PostgreSQL
sudo postgresql-setup --initdb
sudo systemctl enable --now postgresql redis

# Install qpdf from source (if not available)
wget https://github.com/qpdf/qpdf/releases/download/v11.5.0/qpdf-11.5.0.tar.gz
tar xzf qpdf-11.5.0.tar.gz
cd qpdf-11.5.0
./configure --prefix=/usr/local
make
sudo make install
cd ..
```

### Debian/Ubuntu

```bash
# Update package list
sudo apt update

# Install system dependencies
sudo apt install -y \
    python3 python3-pip python3-dev python3-venv \
    postgresql postgresql-contrib \
    redis-server \
    tesseract-ocr tesseract-ocr-eng tesseract-ocr-deu \
    poppler-utils imagemagick ghostscript \
    unpaper optipng pngquant jbig2dec qpdf \
    libpq-dev libmagic-dev libxml2-dev libxslt1-dev \
    zlib1g-dev libjpeg-dev libwebp-dev \
    build-essential git

# Additional OCR languages (optional)
sudo apt install -y tesseract-ocr-fra tesseract-ocr-spa tesseract-ocr-ita

# Enable and start services
sudo systemctl enable --now postgresql redis-server
```

### Alpine Linux

```bash
# Install dependencies
apk add --no-cache \
    python3 py3-pip python3-dev \
    postgresql postgresql-client \
    redis \
    tesseract-ocr tesseract-ocr-data-eng \
    poppler-utils imagemagick ghostscript \
    unpaper optipng pngquant jbig2dec qpdf \
    gcc g++ musl-dev \
    postgresql-dev libmagic file-dev \
    libxml2-dev libxslt-dev \
    zlib-dev jpeg-dev libwebp-dev \
    git

# Start services
rc-update add postgresql default
rc-update add redis default
rc-service postgresql start
rc-service redis start
```

### macOS

```bash
# Install Homebrew dependencies
brew install \
    python@3.11 \
    postgresql@14 \
    redis \
    tesseract tesseract-lang \
    poppler imagemagick ghostscript \
    unpaper optipng pngquant jbig2 qpdf \
    libmagic libxml2 libxslt \
    zlib jpeg webp

# Start services
brew services start postgresql@14
brew services start redis

# Set Python 3.11 as default
brew link python@3.11
```

### FreeBSD

```bash
# Install from packages
pkg install -y \
    python39 py39-pip \
    postgresql13-server postgresql13-client \
    redis \
    tesseract tesseract-data \
    poppler-utils ImageMagick7 ghostscript \
    unpaper optipng pngquant jbig2dec qpdf \
    libmagic libxml2 libxslt

# Enable services
sysrc postgresql_enable="YES"
sysrc redis_enable="YES"
service postgresql initdb
service postgresql start
service redis start
```

## 3. Installation Steps

### Create System User

```bash
# Create paperless user
sudo useradd -r -m -d /opt/paperless -s /bin/bash paperless

# Create directory structure
sudo mkdir -p /opt/paperless/{consume,data,media,static,export}
sudo chown -R paperless:paperless /opt/paperless
```

### Database Setup

```bash
# PostgreSQL setup
sudo -u postgres psql <<EOF
CREATE USER paperless WITH PASSWORD 'secure_password';
CREATE DATABASE paperless OWNER paperless;
GRANT ALL PRIVILEGES ON DATABASE paperless TO paperless;
EOF

# MariaDB/MySQL alternative
sudo mysql <<EOF
CREATE DATABASE paperless CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'paperless'@'localhost' IDENTIFIED BY 'secure_password';
GRANT ALL PRIVILEGES ON paperless.* TO 'paperless'@'localhost';
FLUSH PRIVILEGES;
EOF
```

### Install Paperless-ngx

```bash
# Switch to paperless user
sudo -u paperless -i

# Create virtual environment
python3 -m venv /opt/paperless/venv
source /opt/paperless/venv/bin/activate

# Upgrade pip
pip install --upgrade pip setuptools wheel

# Install Paperless-ngx
pip install paperless-ngx

# Install additional dependencies
pip install psycopg2-binary  # For PostgreSQL
# OR
pip install mysqlclient      # For MariaDB/MySQL
```

### 4. Configuration

Create `/opt/paperless/paperless.conf`:

```bash
# Required Settings
PAPERLESS_REDIS=redis://localhost:6379
PAPERLESS_DBHOST=localhost
PAPERLESS_DBPORT=5432
PAPERLESS_DBNAME=paperless
PAPERLESS_DBUSER=paperless
PAPERLESS_DBPASS=secure_password
PAPERLESS_DBENGINE=postgresql  # or mysql

# Paths
PAPERLESS_CONSUMPTION_DIR=/opt/paperless/consume
PAPERLESS_DATA_DIR=/opt/paperless/data
PAPERLESS_MEDIA_ROOT=/opt/paperless/media
PAPERLESS_STATICDIR=/opt/paperless/static

# Security
PAPERLESS_SECRET_KEY=change-this-to-a-long-random-string
PAPERLESS_URL=https://paperless.example.com
PAPERLESS_ALLOWED_HOSTS=paperless.example.com,localhost
PAPERLESS_CORS_ALLOWED_HOSTS=https://paperless.example.com
PAPERLESS_FORCE_HTTPS=true

# OCR Settings
PAPERLESS_OCR_LANGUAGE=eng+deu
PAPERLESS_OCR_MODE=skip_noarchive
PAPERLESS_OCR_PAGES=0
PAPERLESS_OCR_IMAGE_DPI=300
PAPERLESS_OCR_MAX_IMAGE_PIXELS=0

# Document Processing
PAPERLESS_TASK_WORKERS=2
PAPERLESS_THREADS_PER_WORKER=2
PAPERLESS_TIME_ZONE=America/New_York
PAPERLESS_FILENAME_FORMAT={created_year}/{correspondent}/{title}
PAPERLESS_DATE_ORDER=MDY

# Features
PAPERLESS_CONSUMER_RECURSIVE=true
PAPERLESS_CONSUMER_SUBDIRS_AS_TAGS=true
PAPERLESS_TRASH_DIR=/opt/paperless/trash
PAPERLESS_EMPTY_TRASH_AGE=30

# Email Processing (optional)
PAPERLESS_EMAIL_HOST=imap.gmail.com
PAPERLESS_EMAIL_PORT=993
PAPERLESS_EMAIL_HOST_USER=your-email@gmail.com
PAPERLESS_EMAIL_HOST_PASSWORD=your-app-password
PAPERLESS_EMAIL_SSL=true

# Admin User
PAPERLESS_ADMIN_USER=admin
PAPERLESS_ADMIN_PASSWORD=secure-admin-password
PAPERLESS_ADMIN_MAIL=admin@example.com
```

### Initialize Application

```bash
# As paperless user with venv activated
cd /opt/paperless

# Run migrations
paperless-ngx migrate

# Create cache tables
paperless-ngx createcachetable

# Collect static files
paperless-ngx collectstatic --no-input

# Create superuser (if not set in config)
paperless-ngx createsuperuser

# Check configuration
paperless-ngx check
```

## Service Configuration

### Systemd Services

Create `/etc/systemd/system/paperless-webserver.service`:

```ini
[Unit]
Description=Paperless-ngx Webserver
After=network.target postgresql.service redis.service

[Service]
Type=simple
User=paperless
Group=paperless
WorkingDirectory=/opt/paperless
Environment="PATH=/opt/paperless/venv/bin:/usr/local/bin:/usr/bin:/bin"
EnvironmentFile=/opt/paperless/paperless.conf
ExecStart=/opt/paperless/venv/bin/gunicorn -c /opt/paperless/gunicorn.conf.py paperless.asgi:application
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Create `/etc/systemd/system/paperless-consumer.service`:

```ini
[Unit]
Description=Paperless-ngx Document Consumer
After=network.target postgresql.service redis.service

[Service]
Type=simple
User=paperless
Group=paperless
WorkingDirectory=/opt/paperless
Environment="PATH=/opt/paperless/venv/bin:/usr/local/bin:/usr/bin:/bin"
EnvironmentFile=/opt/paperless/paperless.conf
ExecStart=/opt/paperless/venv/bin/paperless-ngx document_consumer
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Create `/etc/systemd/system/paperless-scheduler.service`:

```ini
[Unit]
Description=Paperless-ngx Celery Beat
After=network.target postgresql.service redis.service

[Service]
Type=simple
User=paperless
Group=paperless
WorkingDirectory=/opt/paperless
Environment="PATH=/opt/paperless/venv/bin:/usr/local/bin:/usr/bin:/bin"
EnvironmentFile=/opt/paperless/paperless.conf
ExecStart=/opt/paperless/venv/bin/celery -A paperless beat -l INFO
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Create `/etc/systemd/system/paperless-task-queue.service`:

```ini
[Unit]
Description=Paperless-ngx Celery Workers
After=network.target postgresql.service redis.service

[Service]
Type=simple
User=paperless
Group=paperless
WorkingDirectory=/opt/paperless
Environment="PATH=/opt/paperless/venv/bin:/usr/local/bin:/usr/bin:/bin"
EnvironmentFile=/opt/paperless/paperless.conf
ExecStart=/opt/paperless/venv/bin/celery -A paperless worker -l INFO
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Enable and start services:

```bash
sudo systemctl daemon-reload
sudo systemctl enable paperless-webserver paperless-consumer paperless-scheduler paperless-task-queue
sudo systemctl start paperless-webserver paperless-consumer paperless-scheduler paperless-task-queue
```

### Gunicorn Configuration

Create `/opt/paperless/gunicorn.conf.py`:

```python
bind = "127.0.0.1:8000"
workers = 4
worker_class = "paperless.workers.PaperlessWorker"
timeout = 120
max_requests = 1000
max_requests_jitter = 50
keepalive = 5
errorlog = "-"
accesslog = "-"
loglevel = "info"
```

## Reverse Proxy Configuration

### nginx

```nginx
server {
    listen 80;
    server_name paperless.example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name paperless.example.com;

    ssl_certificate /etc/letsencrypt/live/paperless.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/paperless.example.com/privkey.pem;

    client_max_body_size 100M;

    location / {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_redirect off;
    }

    location /ws {
        proxy_pass http://localhost:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /static/ {
        alias /opt/paperless/static/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    location /media/ {
        alias /opt/paperless/media/;
        add_header Content-Disposition "attachment";
    }
}
```

### Apache

```apache
<VirtualHost *:80>
    ServerName paperless.example.com
    Redirect permanent / https://paperless.example.com/
</VirtualHost>

<VirtualHost *:443>
    ServerName paperless.example.com
    
    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/paperless.example.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/paperless.example.com/privkey.pem
    
    ProxyPreserveHost On
    ProxyRequests Off
    
    ProxyPass /static/ !
    ProxyPass /media/ !
    ProxyPass / http://localhost:8000/
    ProxyPassReverse / http://localhost:8000/
    
    # WebSocket support
    RewriteEngine On
    RewriteCond %{HTTP:Upgrade} websocket [NC]
    RewriteCond %{HTTP:Connection} upgrade [NC]
    RewriteRule ^/?(.*) "ws://localhost:8000/$1" [P,L]
    
    Alias /static /opt/paperless/static
    Alias /media /opt/paperless/media
    
    <Directory /opt/paperless/static>
        Require all granted
    </Directory>
    
    <Directory /opt/paperless/media>
        Require all granted
        Header set Content-Disposition "attachment"
    </Directory>
    
    LimitRequestBody 104857600
</VirtualHost>
```

## Document Processing

### Consumption Methods

1. **Filesystem Monitoring**:
```bash
# Documents placed in consume directory are automatically processed
cp document.pdf /opt/paperless/consume/

# With tags via filename
cp document.pdf "/opt/paperless/consume/document +tag1 +tag2.pdf"
```

2. **Email Processing**:
```bash
# Configure in paperless.conf
PAPERLESS_EMAIL_HOST=imap.gmail.com
PAPERLESS_EMAIL_PORT=993
PAPERLESS_EMAIL_HOST_USER=paperless@example.com
PAPERLESS_EMAIL_HOST_PASSWORD=app-specific-password
```

3. **API Upload**:
```bash
# Upload via API
curl -X POST https://paperless.example.com/api/documents/post_document/ \
  -H "Authorization: Token your-api-token" \
  -F "document=@document.pdf" \
  -F "title=My Document" \
  -F "tags=1,2,3"
```

### OCR Configuration

```bash
# Install additional languages
# RHEL/CentOS
sudo dnf install -y tesseract-langpack-jpn tesseract-langpack-chi-sim

# Debian/Ubuntu
sudo apt install -y tesseract-ocr-jpn tesseract-ocr-chi-sim

# Configure in paperless.conf
PAPERLESS_OCR_LANGUAGE=eng+deu+fra+jpn
PAPERLESS_OCR_MODE=force  # always, skip, skip_noarchive, force
```

## 9. Backup and Restore

### Backup Script

```bash
#!/bin/bash
# backup-paperless.sh

BACKUP_DIR="/backup/paperless/$(date +%Y%m%d_%H%M%S)"
mkdir -p "$BACKUP_DIR"

# Stop services
sudo systemctl stop paperless-consumer paperless-webserver

# Export documents and database
sudo -u paperless /opt/paperless/venv/bin/paperless-ngx document_exporter "$BACKUP_DIR/export"

# Backup media files
tar czf "$BACKUP_DIR/media.tar.gz" -C /opt/paperless media/

# Backup configuration
cp /opt/paperless/paperless.conf "$BACKUP_DIR/"

# Start services
sudo systemctl start paperless-consumer paperless-webserver

echo "Backup completed: $BACKUP_DIR"
```

### Restore Script

```bash
#!/bin/bash
# restore-paperless.sh

BACKUP_DIR="$1"
if [ -z "$BACKUP_DIR" ]; then
    echo "Usage: $0 <backup-directory>"
    exit 1
fi

# Stop services
sudo systemctl stop paperless-consumer paperless-webserver paperless-scheduler paperless-task-queue

# Import documents and database
sudo -u paperless /opt/paperless/venv/bin/paperless-ngx document_importer "$BACKUP_DIR/export"

# Restore media files
tar xzf "$BACKUP_DIR/media.tar.gz" -C /opt/paperless/

# Restore configuration
cp "$BACKUP_DIR/paperless.conf" /opt/paperless/

# Fix permissions
sudo chown -R paperless:paperless /opt/paperless

# Start services
sudo systemctl start paperless-consumer paperless-webserver paperless-scheduler paperless-task-queue

echo "Restore completed from: $BACKUP_DIR"
```

## Performance Optimization

### Database Optimization

```sql
-- PostgreSQL
VACUUM ANALYZE;
REINDEX DATABASE paperless;

-- Add indexes for common queries
CREATE INDEX CONCURRENTLY idx_documents_created ON documents_document(created);
CREATE INDEX CONCURRENTLY idx_documents_correspondent ON documents_document(correspondent_id);
CREATE INDEX CONCURRENTLY idx_documents_tags ON documents_document_tags(document_id, tag_id);
```

### Redis Configuration

```bash
# Edit /etc/redis/redis.conf or /etc/redis.conf
maxmemory 1gb
maxmemory-policy allkeys-lru
save ""  # Disable persistence for cache
```

### Worker Tuning

```bash
# In paperless.conf
PAPERLESS_TASK_WORKERS=4  # Adjust based on CPU cores
PAPERLESS_THREADS_PER_WORKER=2
PAPERLESS_WEBSERVER_WORKERS=4
```

## Security Hardening

### API Security

```bash
# Generate API token
sudo -u paperless /opt/paperless/venv/bin/paperless-ngx create_token username

# Disable API if not needed
PAPERLESS_DISABLE_API=true
```

### File Permissions

```bash
# Secure directories
sudo chmod 750 /opt/paperless
sudo chmod 770 /opt/paperless/consume
sudo chmod 750 /opt/paperless/media
sudo chown -R paperless:paperless /opt/paperless

# SELinux (if enabled)
sudo semanage fcontext -a -t httpd_sys_content_t "/opt/paperless/static(/.*)?"
sudo semanage fcontext -a -t httpd_sys_rw_content_t "/opt/paperless/media(/.*)?"
sudo restorecon -Rv /opt/paperless
```

### Firewall Rules

```bash
# UFW
sudo ufw allow from 192.168.1.0/24 to any port 8000

# firewalld
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" port port="8000" protocol="tcp" accept'
sudo firewall-cmd --reload
```

## Monitoring

### Health Checks

```bash
# Check service status
sudo systemctl status paperless-*

# API health check
curl -H "Authorization: Token your-token" https://paperless.example.com/api/

# Check logs
sudo journalctl -u paperless-webserver -f
```

### Prometheus Metrics

```python
# Add to settings
PROMETHEUS_EXPORT_ENABLED = True
```

## 6. Troubleshooting

### Common Issues

1. **OCR not working**:
```bash
# Test Tesseract
tesseract --version
tesseract test.png output -l eng

# Check available languages
tesseract --list-langs
```

2. **Permission errors**:
```bash
# Fix ownership
sudo chown -R paperless:paperless /opt/paperless

# Check service user
sudo -u paperless whoami
```

3. **Consumer not processing**:
```bash
# Check logs
sudo journalctl -u paperless-consumer -n 100

# Test consumer manually
sudo -u paperless /opt/paperless/venv/bin/paperless-ngx document_consumer --oneshot
```

## Integration

### Mobile Apps

- iOS: Paperless Mobile
- Android: Paperless Mobile
- Configure API access with token authentication

### Desktop Integration

```bash
# Install paperless-cli
pip install paperless-cli

# Configure
paperless-cli config set url https://paperless.example.com
paperless-cli config set token your-api-token

# Upload document
paperless-cli upload document.pdf --title "My Document" --tags "tag1,tag2"
```

## Additional Resources

- [Official Documentation](https://docs.paperless-ngx.com/)
- [GitHub Repository](https://github.com/paperless-ngx/paperless-ngx)
- [Community Forum](https://github.com/paperless-ngx/paperless-ngx/discussions)
- [API Documentation](https://docs.paperless-ngx.com/api/)
- [Docker Hub](https://hub.docker.com/r/paperlessngx/paperless-ngx) (reference only)

---

**Note:** This guide is part of the [HowToMgr](https://howtomgr.github.io) collection. Always refer to official documentation for the most up-to-date information.