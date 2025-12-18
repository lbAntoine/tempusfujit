# Monitoring and Backup Strategy

## Overview

A robust monitoring and backup strategy ensures:
- **Visibility**: Know what's happening on your server
- **Alerting**: Get notified when things go wrong
- **Recovery**: Restore data when needed
- **Performance**: Track resource usage and optimize

## Monitoring Architecture

### Components

1. **Metrics Collection**: Prometheus
2. **Visualization**: Grafana
3. **Log Aggregation**: Promtail + Loki (optional)
4. **Alerting**: Alertmanager
5. **Uptime Monitoring**: Uptime Kuma

## Solution 1: Prometheus + Grafana (Recommended)

### What is Prometheus?

Time-series database that scrapes metrics from your services.

### What is Grafana?

Visualization platform that displays Prometheus data in beautiful dashboards.

### Why This Stack?

- ✅ **Industry standard**: Most popular monitoring stack
- ✅ **Powerful**: Can monitor everything
- ✅ **Free and open source**: No licensing costs
- ✅ **Extensive ecosystem**: Thousands of exporters and dashboards
- ✅ **Active development**: Well-maintained

### Architecture

```
Server Components → Exporters → Prometheus (scrapes) → Grafana (displays)
     ↓                                ↓
[Docker, System,               Alertmanager
 Caddy, etc.]                       ↓
                               Notifications
```

## Monitoring Implementation

### Step 1: Deploy Monitoring Stack with Docker Compose

Create monitoring docker-compose.yml:

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    restart: unless-stopped
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_SERVER_ROOT_URL=https://monitoring.antoinelb.fr
    ports:
      - "3001:3000"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    restart: unless-stopped
    networks:
      - monitoring
    depends_on:
      - prometheus

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    command:
      - '--path.rootfs=/host'
    ports:
      - "9100:9100"
    volumes:
      - '/:/host:ro,rslave'
    restart: unless-stopped
    networks:
      - monitoring

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    ports:
      - "8082:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro
    restart: unless-stopped
    networks:
      - monitoring

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml
      - alertmanager-data:/alertmanager
    restart: unless-stopped
    networks:
      - monitoring

volumes:
  prometheus-data:
  grafana-data:
  alertmanager-data:

networks:
  monitoring:
    driver: bridge
```

### Step 2: Configure Prometheus

Create `prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    monitor: 'server-monitor'

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

# Load rules once and periodically evaluate them
rule_files:
  - 'alerts.yml'

# Scrape configurations
scrape_configs:
  # Prometheus itself
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Node Exporter (system metrics)
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']

  # cAdvisor (container metrics)
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  # Caddy metrics (if enabled)
  - job_name: 'caddy'
    static_configs:
      - targets: ['host.docker.internal:2019']

  # Add your applications here
  - job_name: 'my-app'
    static_configs:
      - targets: ['host.docker.internal:3000']
    metrics_path: '/metrics'
```

### Step 3: Configure Alerting Rules

Create `prometheus/alerts.yml`:

```yaml
groups:
  - name: system_alerts
    interval: 30s
    rules:
      # High CPU usage
      - alert: HighCPUUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage detected"
          description: "CPU usage is above 80% for 5 minutes on {{ $labels.instance }}"

      # High memory usage
      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage detected"
          description: "Memory usage is above 85% on {{ $labels.instance }}"

      # Low disk space
      - alert: LowDiskSpace
        expr: (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 < 15
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Low disk space"
          description: "Disk space is below 15% on {{ $labels.instance }}"

      # Container down
      - alert: ContainerDown
        expr: up == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Container/service is down"
          description: "{{ $labels.job }} on {{ $labels.instance }} is down"

  - name: docker_alerts
    interval: 30s
    rules:
      # High container memory usage
      - alert: HighContainerMemory
        expr: (container_memory_usage_bytes / container_spec_memory_limit_bytes) * 100 > 90
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Container memory usage is high"
          description: "Container {{ $labels.name }} is using > 90% of its memory limit"

      # Container restart
      - alert: ContainerRestarted
        expr: rate(container_last_seen[5m]) > 0
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Container has restarted"
          description: "Container {{ $labels.name }} has restarted"
```

### Step 4: Configure Alertmanager

Create `alertmanager/alertmanager.yml`:

```yaml
global:
  resolve_timeout: 5m

# Email configuration (optional)
route:
  receiver: 'default'
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h

receivers:
  - name: 'default'
    email_configs:
      - to: 'you@antoinelb.fr'
        from: 'alertmanager@antoinelb.fr'
        smarthost: 'smtp.gmail.com:587'
        auth_username: 'your-email@gmail.com'
        auth_password: 'your-app-password'
        headers:
          Subject: '{{ .GroupLabels.alertname }} - {{ .CommonLabels.severity }}'

  # Discord webhook (alternative)
  - name: 'discord'
    discord_configs:
      - webhook_url: 'YOUR_DISCORD_WEBHOOK_URL'
        title: '{{ .GroupLabels.alertname }}'

  # Slack webhook (alternative)
  - name: 'slack'
    slack_configs:
      - api_url: 'YOUR_SLACK_WEBHOOK_URL'
        channel: '#alerts'
        title: '{{ .GroupLabels.alertname }}'

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'instance']
```

### Step 5: Deploy Monitoring Stack

```bash
# Create directory structure
mkdir -p ~/monitoring/{prometheus,grafana/provisioning,alertmanager}

# Create configuration files (as shown above)

# Set Grafana password
echo "GRAFANA_PASSWORD=$(openssl rand -base64 32)" > ~/monitoring/.env

# Deploy
cd ~/monitoring
docker compose up -d

# Check logs
docker compose logs -f
```

### Step 6: Configure Caddy for Access

Add to `/etc/caddy/Caddyfile`:

```caddyfile
# Grafana (main dashboard)
monitoring.antoinelb.fr {
    @notlocal {
        not remote_ip 100.64.0.0/10  # VPN only
    }
    respond @notlocal 403

    reverse_proxy localhost:3001
}

# Prometheus (optional, for debugging)
prometheus.antoinelb.fr {
    @notlocal {
        not remote_ip 100.64.0.0/10
    }
    respond @notlocal 403

    reverse_proxy localhost:9090
}
```

### Step 7: Configure Grafana

1. **Access Grafana**: https://monitoring.antoinelb.fr
2. **Login**: admin / (password from .env file)
3. **Add Data Source**:
   - Settings → Data Sources → Add data source
   - Choose Prometheus
   - URL: `http://prometheus:9090`
   - Click "Save & Test"

4. **Import Dashboards**:
   - Dashboards → Import
   - Use dashboard IDs:
     - **1860**: Node Exporter Full
     - **193**: Docker monitoring
     - **11074**: Caddy monitoring

5. **Create Custom Dashboard**: For your specific apps

## Pre-built Grafana Dashboards

### System Overview Dashboard

Includes:
- CPU usage per core
- Memory usage
- Disk I/O
- Network traffic
- System load
- Disk space

**Import ID**: 1860

### Docker Container Dashboard

Includes:
- Container CPU usage
- Container memory usage
- Container network I/O
- Container status
- Image information

**Import ID**: 193

### Caddy Metrics Dashboard

Enable Caddy metrics first:

```bash
# Add to Caddyfile (global options at top)
{
    servers {
        metrics
    }
}
```

Then reload Caddy and import dashboard ID: 11074

## Alternative: Uptime Kuma

### What is Uptime Kuma?

Simple, beautiful uptime monitoring tool. Great complement to Prometheus/Grafana.

### Features

- ✅ HTTP(s) monitoring
- ✅ TCP port monitoring
- ✅ Ping monitoring
- ✅ Status page
- ✅ Notifications (many providers)
- ✅ Beautiful UI

### Setup

Add to your monitoring docker-compose.yml:

```yaml
  uptime-kuma:
    image: louislam/uptime-kuma:latest
    container_name: uptime-kuma
    ports:
      - "3002:3001"
    volumes:
      - uptime-kuma-data:/app/data
    restart: unless-stopped
    networks:
      - monitoring
```

Add to Caddy:
```caddyfile
status.antoinelb.fr {
    reverse_proxy localhost:3002
}
```

Configure monitors:
1. Access https://status.antoinelb.fr
2. Add monitors for each service
3. Configure notifications
4. Share public status page (optional)

## Backup Strategy

### What to Backup

1. **Critical Data**:
   - Application databases
   - User-uploaded files
   - Configuration files
   - SSL certificates (Caddy handles renewal, but good to backup)

2. **Infrastructure Configuration**:
   - Docker volumes
   - Coolify database
   - Monitoring configuration
   - This documentation repo

3. **Not Critical** (can be recreated):
   - Docker images (pull from registry)
   - System packages (reinstall)
   - Application code (in Git)

### Backup Locations

#### Option 1: Local Backups (Basic)

Store backups on same server, different disk/partition.

**Pros**: Fast, simple, no external dependencies
**Cons**: Lost if server fails or disk fails

#### Option 2: Remote Backups (Recommended)

Store backups on remote storage.

**Options**:
- **S3-compatible storage**: Backblaze B2, Wasabi, AWS S3
- **rsync to another server**: Your NAS or another VPS
- **Cloud storage**: rclone to Google Drive, Dropbox, etc.

#### Option 3: Hybrid (Best)

Local + Remote: Local for quick restores, remote for disaster recovery.

### Backup Implementation

#### Method 1: Restic (Recommended)

Restic is a modern backup tool with encryption, deduplication, and multiple backends.

##### Install Restic

```bash
sudo dnf install -y restic
```

##### Initialize Repository

```bash
# Local repository (for testing)
restic init --repo /backup/restic-repo

# Or S3 (Backblaze B2 example)
export AWS_ACCESS_KEY_ID="your-b2-key-id"
export AWS_SECRET_ACCESS_KEY="your-b2-application-key"
export RESTIC_PASSWORD="your-restic-password"
export RESTIC_REPOSITORY="s3:s3.us-west-000.backblazeb2.com/your-bucket-name"

restic init
```

##### Create Backup Script

Create `/usr/local/bin/backup.sh`:

```bash
#!/bin/bash

# Configuration
export AWS_ACCESS_KEY_ID="your-key"
export AWS_SECRET_ACCESS_KEY="your-secret"
export RESTIC_PASSWORD="your-password"
export RESTIC_REPOSITORY="s3:s3.us-west-000.backblazeb2.com/your-bucket"

# Backup paths
BACKUP_DIRS=(
    "/home/rainreport/code-projects"
    "/var/lib/docker/volumes"
    "/etc/caddy"
    "/home/rainreport/monitoring"
)

# Create backup
echo "Starting backup at $(date)"

restic backup "${BACKUP_DIRS[@]}" \
    --exclude="*.log" \
    --exclude="node_modules" \
    --exclude=".git" \
    --exclude="*.tmp" \
    --tag automated

# Forget old backups (retention policy)
restic forget \
    --keep-daily 7 \
    --keep-weekly 4 \
    --keep-monthly 6 \
    --prune

# Check repository integrity (weekly)
if [ $(date +%u) -eq 1 ]; then
    echo "Running repository check..."
    restic check
fi

echo "Backup completed at $(date)"
```

Make executable:
```bash
sudo chmod +x /usr/local/bin/backup.sh
```

##### Automate with Systemd Timer

Create `/etc/systemd/system/backup.service`:

```ini
[Unit]
Description=Restic Backup Service
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
User=root
```

Create `/etc/systemd/system/backup.timer`:

```ini
[Unit]
Description=Restic Backup Timer

[Timer]
# Run daily at 3 AM
OnCalendar=daily
OnCalendar=03:00
Persistent=true

[Install]
WantedBy=timers.target
```

Enable timer:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now backup.timer

# Check status
sudo systemctl list-timers
```

##### Restore from Backup

```bash
# List snapshots
restic snapshots

# List files in snapshot
restic ls latest

# Restore specific files
restic restore latest --target /tmp/restore --include /path/to/file

# Restore entire snapshot
restic restore latest --target /restore
```

#### Method 2: Database-Specific Backups

##### PostgreSQL Backup

```bash
#!/bin/bash
# /usr/local/bin/backup-postgres.sh

BACKUP_DIR="/backup/postgres"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
CONTAINER="postgres-container-name"  # Get from: docker ps

# Create backup directory
mkdir -p $BACKUP_DIR

# Backup all databases
docker exec $CONTAINER pg_dumpall -U postgres > "$BACKUP_DIR/all_databases_$TIMESTAMP.sql"

# Or backup specific database
docker exec $CONTAINER pg_dump -U postgres dbname > "$BACKUP_DIR/dbname_$TIMESTAMP.sql"

# Compress
gzip "$BACKUP_DIR/all_databases_$TIMESTAMP.sql"

# Keep only last 14 days
find $BACKUP_DIR -name "*.sql.gz" -mtime +14 -delete

echo "PostgreSQL backup completed: $TIMESTAMP"
```

##### MySQL Backup

```bash
#!/bin/bash
# /usr/local/bin/backup-mysql.sh

BACKUP_DIR="/backup/mysql"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
CONTAINER="mysql-container-name"
MYSQL_ROOT_PASSWORD="your-password"

mkdir -p $BACKUP_DIR

# Backup all databases
docker exec $CONTAINER mysqldump -u root -p$MYSQL_ROOT_PASSWORD --all-databases > "$BACKUP_DIR/all_databases_$TIMESTAMP.sql"

# Compress
gzip "$BACKUP_DIR/all_databases_$TIMESTAMP.sql"

# Retention
find $BACKUP_DIR -name "*.sql.gz" -mtime +14 -delete

echo "MySQL backup completed: $TIMESTAMP"
```

##### MongoDB Backup

```bash
#!/bin/bash
# /usr/local/bin/backup-mongodb.sh

BACKUP_DIR="/backup/mongodb"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
CONTAINER="mongodb-container-name"

mkdir -p $BACKUP_DIR

# Backup
docker exec $CONTAINER mongodump --archive=/data/backup_$TIMESTAMP.archive

# Copy from container
docker cp $CONTAINER:/data/backup_$TIMESTAMP.archive $BACKUP_DIR/

# Compress
gzip "$BACKUP_DIR/backup_$TIMESTAMP.archive"

# Retention
find $BACKUP_DIR -name "*.archive.gz" -mtime +14 -delete

echo "MongoDB backup completed: $TIMESTAMP"
```

#### Method 3: Docker Volume Backups

```bash
#!/bin/bash
# /usr/local/bin/backup-volumes.sh

BACKUP_DIR="/backup/volumes"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR

# List all volumes
VOLUMES=$(docker volume ls -q)

for VOLUME in $VOLUMES; do
    echo "Backing up volume: $VOLUME"

    # Create tarball of volume
    docker run --rm \
        -v $VOLUME:/source:ro \
        -v $BACKUP_DIR:/backup \
        alpine \
        tar czf /backup/${VOLUME}_${TIMESTAMP}.tar.gz -C /source .
done

# Retention
find $BACKUP_DIR -name "*.tar.gz" -mtime +7 -delete

echo "Volume backup completed: $TIMESTAMP"
```

### Coolify Backups

Coolify has built-in backup features:

1. **Database Backups**:
   - Navigate to Database → Settings → Backups
   - Configure frequency and S3 destination
   - Coolify handles everything

2. **Coolify's Own Database**:
   ```bash
   docker exec coolify-db pg_dump -U coolify > coolify-backup.sql
   ```

3. **Application Files**:
   - Use Restic to backup Docker volumes
   - Or configure per-application backup scripts

### Backup Monitoring

Add backup status to your monitoring:

```bash
# Create backup check script
# /usr/local/bin/check-backups.sh

#!/bin/bash

# Check if backup ran today
LAST_BACKUP=$(restic snapshots --json | jq -r '.[0].time')
LAST_BACKUP_TS=$(date -d "$LAST_BACKUP" +%s)
NOW_TS=$(date +%s)
HOURS_SINCE=$(( ($NOW_TS - $LAST_BACKUP_TS) / 3600 ))

if [ $HOURS_SINCE -gt 30 ]; then
    echo "ERROR: Last backup was $HOURS_SINCE hours ago"
    # Send alert via webhook or email
    exit 1
else
    echo "OK: Last backup was $HOURS_SINCE hours ago"
    exit 0
fi
```

Add to Uptime Kuma as a script monitor or integrate with Prometheus.

### Backup Restoration Testing

**Test your backups regularly!**

```bash
# Monthly restoration test
# /usr/local/bin/test-restore.sh

#!/bin/bash

TEST_DIR="/tmp/restore-test-$(date +%Y%m%d)"

# Restore latest backup
restic restore latest --target $TEST_DIR

# Verify key files exist
if [ -f "$TEST_DIR/important-file" ]; then
    echo "Restoration test PASSED"
else
    echo "Restoration test FAILED"
    # Send alert
fi

# Cleanup
rm -rf $TEST_DIR
```

Schedule this monthly to ensure backups are valid.

## Log Management (Optional but Recommended)

### Loki + Promtail

For centralized log collection and viewing in Grafana.

Add to monitoring docker-compose.yml:

```yaml
  loki:
    image: grafana/loki:latest
    container_name: loki
    ports:
      - "3100:3100"
    volumes:
      - ./loki/loki-config.yml:/etc/loki/loki-config.yml
      - loki-data:/loki
    command: -config.file=/etc/loki/loki-config.yml
    restart: unless-stopped
    networks:
      - monitoring

  promtail:
    image: grafana/promtail:latest
    container_name: promtail
    volumes:
      - ./promtail/promtail-config.yml:/etc/promtail/promtail-config.yml
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    command: -config.file=/etc/promtail/promtail-config.yml
    restart: unless-stopped
    networks:
      - monitoring
    depends_on:
      - loki
```

Then add Loki as data source in Grafana and view logs alongside metrics!

## Maintenance Checklist

### Daily (Automated)
- ✅ Backups run
- ✅ Monitoring collects metrics
- ✅ Alerts sent if issues

### Weekly (Manual)
- Check Grafana dashboards for trends
- Review alert history
- Verify backups succeeded

### Monthly
- Test backup restoration
- Review disk space usage
- Update monitoring stack
- Rotate old logs

### Quarterly
- Audit backup retention policy
- Review and update alerting rules
- Performance tuning based on metrics

## Cost Considerations

### Storage Costs

**Backblaze B2** (recommended for backups):
- First 10GB: Free
- After: $0.005/GB/month (~$5/TB/month)
- Download: First 1GB/day free, then $0.01/GB

**For your setup**:
- Estimate 50-100GB of backups: ~$0.25-$0.50/month
- Very affordable for peace of mind!

### Alternative: Wasabi

- $5.99/TB/month (no egress fees)
- Good for larger backups

## Summary: Recommended Setup

1. **Monitoring**:
   - Prometheus + Grafana + Node Exporter + cAdvisor
   - Uptime Kuma for simple service monitoring
   - Alertmanager for notifications

2. **Backups**:
   - Restic to Backblaze B2 (daily at 3 AM)
   - Database-specific backups (daily)
   - Docker volume backups (weekly)
   - Coolify built-in backups for databases

3. **Retention**:
   - Daily backups: Keep 7 days
   - Weekly backups: Keep 4 weeks
   - Monthly backups: Keep 6 months

4. **Monitoring Backups**:
   - Uptime Kuma script check for backup status
   - Monthly restoration tests

## Next Steps

1. Deploy monitoring stack (Prometheus + Grafana)
2. Configure alerts and notifications
3. Set up Restic with B2/S3
4. Create and test backup scripts
5. Schedule automated backups
6. Import Grafana dashboards
7. Test restoration process
8. Move on to security hardening

## Resources

- **Prometheus**: https://prometheus.io/docs
- **Grafana**: https://grafana.com/docs
- **Restic**: https://restic.readthedocs.io
- **Uptime Kuma**: https://github.com/louislam/uptime-kuma
- **Backblaze B2**: https://www.backblaze.com/b2/cloud-storage.html
