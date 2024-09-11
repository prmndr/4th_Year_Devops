# Lab Lecture 2: Implementing Node Exporter and Nagios

## Objective
In this lab, students will extend their monitoring stack by configuring Node Exporter for detailed system metrics and set up Nagios for traditional infrastructure monitoring.

## Duration: 1 hour

## Prerequisites
- Completed Lab Lecture 1
- Basic understanding of system metrics and infrastructure monitoring concepts

## Steps

### 1. Adding Node Exporter to Docker Compose (10 minutes)

Update your `docker-compose.yml` to include Node Exporter:

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:v2.30.3
    volumes:
      - ./prometheus:/etc/prometheus
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
    ports:
      - 9090:9090
    restart: always

  grafana:
    image: grafana/grafana:8.2.2
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin_password
      - GF_USERS_ALLOW_SIGN_UP=false
    ports:
      - 3000:3000
    restart: always

  node-exporter:
    image: prom/node-exporter:v1.2.2
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.ignored-mount-points=^/(sys|proc|dev|host|etc)($$|/)'
    ports:
      - 9100:9100
    restart: always

  nagios:
    build:
      context: ./nagios
      dockerfile: Dockerfile
    ports:
      - "8080:80"
    environment:
      - NAGIOSADMIN_USER=nagiosadmin
      - NAGIOSADMIN_PASS=nagios_password
    restart: always

volumes:
  prometheus_data:
  grafana_data:
```

### 2. Configuring Prometheus to scrape Node Exporter (5 minutes)

Update `prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
```

### 3. Setting up Nagios (20 minutes)

Create a new directory for Nagios:

```bash
mkdir nagios
```

Create a `Dockerfile` for Nagios in the `nagios` directory:

```dockerfile
FROM jasonrivers/nagios:latest

COPY nagios.cfg /opt/nagios/etc/nagios.cfg
COPY commands.cfg /opt/nagios/etc/objects/commands.cfg
COPY localhost.cfg /opt/nagios/etc/objects/localhost.cfg
```

Create `nagios/nagios.cfg`:

```
log_file=/usr/local/nagios/var/nagios.log

cfg_file=/usr/local/nagios/etc/objects/commands.cfg
cfg_file=/usr/local/nagios/etc/objects/contacts.cfg
cfg_file=/usr/local/nagios/etc/objects/timeperiods.cfg
cfg_file=/usr/local/nagios/etc/objects/templates.cfg
cfg_file=/usr/local/nagios/etc/objects/localhost.cfg

nagios_user=nagios
nagios_group=nagios

check_external_commands=1
interval_length=60
```

Create `nagios/commands.cfg`:

```
define command {
    command_name check_http
    command_line $USER1$/check_http -H $HOSTADDRESS$ $ARG1$
}

define command {
    command_name check_ping
    command_line $USER1$/check_ping -H $HOSTADDRESS$ -w 3000.0,80% -c 5000.0,100% -p 5
}
```

Create `nagios/localhost.cfg`:

```
define host {
    use                     linux-server
    host_name               localhost
    alias                   localhost
    address                 127.0.0.1
    max_check_attempts      5
    check_period            24x7
    notification_interval   30
    notification_period     24x7
}

define service {
    use                     generic-service
    host_name               localhost
    service_description     PING
    check_command           check_ping!100.0,20%!500.0,60%
}

define service {
    use                     generic-service
    host_name               localhost
    service_description     HTTP
    check_command           check_http
}
```

### 4. Starting the services (5 minutes)

Restart the Docker containers:

```bash
docker-compose up -d
```

### 5. Creating a Node Exporter Dashboard in Grafana (10 minutes)

1. In Grafana, click "Create" > "Import"
2. Enter dashboard ID 1860 (a popular Node Exporter dashboard)
3. Select your Prometheus data source
4. Click "Import"

You should now see a comprehensive dashboard with CPU, memory, disk, and network metrics.

### 6. Accessing Nagios (5 minutes)

Access Nagios at `http://localhost:8080/nagios` (login with nagiosadmin/nagios_password)

## Exercise (15 minutes)
1. Add a new host to monitor in Nagios (e.g., your Prometheus server)
2. Create a new Nagios check for disk space usage
3. Create a new Grafana dashboard panel to display a Node Exporter metric of your choice (e.g., CPU usage)

## Conclusion
In this lab, you've extended your monitoring stack with detailed system metrics from Node Exporter and traditional infrastructure monitoring with Nagios. You've learned how to integrate these tools with your existing Prometheus and Grafana setup. This combination of modern and traditional monitoring techniques provides a comprehensive view of your infrastructure health and performance.

