# Lab Lecture 1: Setting up Prometheus and Grafana

## Objective
In this lab, students will set up a basic monitoring stack using Prometheus and Grafana, including the configuration of a custom metric.

## Duration: 1 hour

## Prerequisites
- Basic understanding of Linux commands
- Docker and Docker Compose installed on the system
- Git installed on the system

## Steps

### 1. Setting up the project structure (5 minutes)

```bash
mkdir prometheus-grafana-lab
cd prometheus-grafana-lab
git init
```

### 2. Creating Docker Compose configuration (10 minutes)

Create a file named `docker-compose.yml` with the following content:

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

  custom-metric:
    build:
      context: .
      dockerfile: Dockerfile.custom-metric
    ports:
      - 8000:8000

volumes:
  prometheus_data:
  grafana_data:
```

### 3. Configuring Prometheus (10 minutes)

Create a directory for Prometheus configuration:

```bash
mkdir -p prometheus
```

Create a file `prometheus/prometheus.yml` with the following content:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'custom-metric'
    static_configs:
      - targets: ['custom-metric:8000']
```

### 4. Creating a custom metric (15 minutes)

Create a new Python file `custom_metric.py`:

```python
from prometheus_client import start_http_server, Counter
import random
import time

# Create a metric to count requests
REQUESTS = Counter('hello_worlds_total', 'Hello Worlds requested')

# Decorate function with metric
@REQUESTS.count_exceptions()
def process_request():
    # Simulate a 50% chance of an exception
    if random.random() < 0.5:
        raise Exception

if __name__ == '__main__':
    # Start up the server to expose the metrics.
    start_http_server(8000)
    # Generate some requests.
    while True:
        process_request()
        time.sleep(1)
```

Create a `Dockerfile.custom-metric`:

```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY custom_metric.py .
RUN pip install prometheus_client

CMD ["python", "custom_metric.py"]
```

### 5. Starting the services (5 minutes)

Start the Docker containers:

```bash
docker-compose up -d
```

### 6. Accessing Prometheus and Grafana (5 minutes)

- Access Prometheus UI at `http://localhost:9090`
- Access Grafana UI at `http://localhost:3000` (login with admin/admin_password)

### 7. Creating a Grafana dashboard (10 minutes)

1. In Grafana, click "Create" > "Dashboard"
2. Add a new panel
3. For the metric, use the PromQL query: `rate(hello_worlds_total[5m])`
4. Save the dashboard

## Exercise (10 minutes)
1. Add another custom metric to the Python script (e.g., a Gauge for simulated CPU usage)
2. Update the Prometheus configuration to scrape this new metric
3. Create a new Grafana dashboard panel to display this metric

## Conclusion
In this lab, you've set up a basic monitoring stack using Prometheus and Grafana. You've learned how to configure Prometheus, connect it to Grafana, create a custom metric, and build a simple dashboard. This foundation will be crucial for more advanced monitoring scenarios in future labs.

