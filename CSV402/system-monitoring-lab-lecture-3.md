# Lab Lecture 3: Implementing ELK Stack for Log Analysis and Custom Prometheus Exporters

## Objective
In this lab, students will set up the ELK (Elasticsearch, Logstash, Kibana) stack for log analysis and create a custom Prometheus exporter for application-specific metrics.

## Duration: 1 hour

## Prerequisites
- Completed Lab Lectures 1 and 2
- Basic understanding of log formats and JSON

## Steps

### 1. Setting up the ELK Stack (20 minutes)

Update your `docker-compose.yml` to include the ELK stack:

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

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.15.1
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
    healthcheck:
      test: ["CMD-SHELL", "curl -s http://localhost:9200 >/dev/null || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5

  logstash:
    image: docker.elastic.co/logstash/logstash:7.15.1
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
    ports:
      - "5000:5000/tcp"
      - "5000:5000/udp"
      - "9600:9600"
    environment:
      LS_JAVA_OPTS: "-Xmx256m -Xms256m"
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:7.15.1
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_URL=http://elasticsearch:9200
    depends_on:
      - elasticsearch

volumes:
  prometheus_data:
  grafana_data:
  elasticsearch_data:
```

Create a directory for Logstash pipeline configuration:

```bash
mkdir -p logstash/pipeline
```

Create `logstash/pipeline/logstash.conf`:

```
input {
  tcp {
    port => 5000
    codec => json
  }
}

filter {
  if [type] == "nginx_access" {
    grok {
      match => { "message" => "%{COMBINEDAPACHELOG}" }
    }
    date {
      match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logstash-%{+YYYY.MM.dd}"
  }
}
```

### 2. Creating a Custom Prometheus Exporter (20 minutes)

Create a new Python file `custom_exporter.py`:

```python
from prometheus_client import start_http_server, Gauge
import random
import time

# Create a metric to track time spent and requests made.
REQUEST_TIME = Gauge('app_request_processing_seconds', 'Time spent processing request')
RANDOM_VALUE = Gauge('app_random_value', 'Random value between 0 and 100')

# Decorate function with metric.
@REQUEST_TIME.time()
def process_request():
    """A dummy function that takes some time."""
    time.sleep(random.random())

if __name__ == '__main__':
    # Start up the server to expose the metrics.
    start_http_server(8000)
    # Generate some requests.
    while True:
        process_request()
        RANDOM_VALUE.set(random.uniform(0, 100))
        time.sleep(1)
```

Create a `Dockerfile.custom-exporter`:

```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY custom_exporter.py .
RUN pip install prometheus_client

CMD ["python", "custom_exporter.py"]
```

Update your `docker-compose.yml` to include the custom exporter:

```yaml
  custom-exporter:
    build:
      context: .
      dockerfile: Dockerfile.custom-exporter
    ports:
      - 8000:8000
```

Update `prometheus/prometheus.yml` to scrape the custom exporter:

```yaml
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'custom-exporter'
    static_configs:
      - targets: ['custom-exporter:8000']
```

### 3. Starting the Services (5 minutes)

Start the Docker containers:

```bash
docker-compose up -d
```

### 4. Configuring Kibana (10 minutes)

1. Access Kibana at `http://localhost:5601`
2. Go to "Management" > "Stack Management" > "Index Patterns"
3. Create a new index pattern with `logstash-*`
4. Go to "Discover" to view incoming logs

### 5. Creating a Grafana Dashboard for Custom Metrics (5 minutes)

1. In Grafana, create a new dashboard
2. Add a panel for `app_request_processing_seconds`
3. Add another panel for `app_random_value`

## Exercise (10 minutes)
1. Modify the custom exporter to include a new metric of your choice
2. Update the Prometheus configuration to scrape this new metric
3. Create a new panel in your Grafana dashboard to display this metric

## Conclusion
In this lab, you've set up the ELK stack for log analysis and created a custom Prometheus exporter. You've learned how to configure Logstash for log processing, use Kibana for log visualization, and create custom metrics with Prometheus. These skills are crucial for comprehensive monitoring and log analysis in modern infrastructure environments.

