# Lecture 4: Telemetry, Metrics, and Types of Monitoring

## 1. Introduction to Telemetry and Metrics (10 minutes) 📊

Telemetry is the process of collecting data from remote or inaccessible points and transmitting it to receiving equipment for monitoring. In DevOps, telemetry refers to collecting metrics, logs, and traces from various components of our systems.

Metrics are numerical measurements collected at regular intervals. They help us understand the behavior and performance of our systems over time.

Key concepts:
- Time-series data
- Labels/Tags
- Aggregation

## 2. Enhancing Our Flask Application with More Metrics (20 minutes) 💻

Let's expand our Flask application to collect more detailed metrics. We'll use the Prometheus client library to implement various types of metrics.

Update your `app.py`:

```python
from flask import Flask, Response, request
import time
import random
import psutil
from prometheus_client import Counter, Histogram, Gauge, Summary, generate_latest, CONTENT_TYPE_LATEST

app = Flask(__name__)

# Counter
REQUESTS = Counter('http_requests_total', 'Total HTTP Requests', ['method', 'endpoint'])
ERRORS = Counter('http_errors_total', 'Total HTTP Errors')

# Histogram
LATENCY = Histogram('http_request_duration_seconds', 'HTTP request latency in seconds')

# Gauge
SYSTEM_LOAD = Gauge('system_load', 'System load average')
MEMORY_USAGE = Gauge('memory_usage_bytes', 'Memory usage in bytes')

# Summary
RESPONSE_SIZE = Summary('http_response_size_bytes', 'Response size in bytes')

@app.before_request
def before_request():
    SYSTEM_LOAD.set(psutil.getloadavg()[0])
    MEMORY_USAGE.set(psutil.virtual_memory().used)

@app.after_request
def after_request(response):
    RESPONSE_SIZE.observe(len(response.data))
    return response

@app.route('/')
def hello():
    REQUESTS.labels(method='GET', endpoint='/').inc()
    return "Hello, DevOps Monitoring!"

@app.route('/simulate-load')
def simulate_load():
    REQUESTS.labels(method='GET', endpoint='/simulate-load').inc()
    start_time = time.time()
    # Simulate CPU load
    x = 0
    for i in range(1000000):
        x += i
    duration = time.time() - start_time
    LATENCY.observe(duration)
    return f"Load simulated, result: {x}"

@app.route('/simulate-error')
def simulate_error():
    REQUESTS.labels(method='GET', endpoint='/simulate-error').inc()
    if random.random() < 0.5:  # 50% chance of error
        ERRORS.inc()
        return "Simulated Error", 500
    return "No error this time"

@app.route('/metrics')
def metrics():
    return Response(generate_latest(), mimetype=CONTENT_TYPE_LATEST)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

Don't forget to add `psutil` to your `requirements.txt` or install it in your Docker container.

Rebuild and restart your Docker container:

```bash
docker build -t flask-app .
docker stop flask-container
docker rm flask-container
docker run -d -p 5000:5000 --name flask-container flask-app
```

## 3. Understanding Different Types of Metrics (15 minutes) 🧮

1. **Counter**: A cumulative metric that only increases (except when it resets to zero on restart)
   Example: Total number of requests

2. **Gauge**: A metric that can increase and decrease
   Example: Current memory usage

3. **Histogram**: Samples observations and counts them in configurable buckets
   Example: Request duration

4. **Summary**: Similar to histogram, but calculates configurable quantiles over a sliding time window
   Example: Response size

Let's add these metrics to our Grafana dashboard:

- Add a panel for memory usage: `memory_usage_bytes`
- Add a panel for system load: `system_load`
- Add a panel for response size: `http_response_size_bytes{quantile="0.9"}`

## 4. Types of Monitoring (20 minutes) 🔍

### 4.1 Infrastructure Monitoring

Infrastructure monitoring focuses on the health and performance of your hardware and software infrastructure.

Let's add some infrastructure monitoring to our setup:

1. Install Node Exporter on your host machine:

```bash
docker run -d --name node-exporter --net="host" --pid="host" -v "/:/host:ro,rslave" quay.io/prometheus/node-exporter:latest --path.rootfs=/host
```

2. Add Node Exporter to your Prometheus configuration (`prometheus.yml`):

```yaml
scrape_configs:
  # ... (previous configs) ...
  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']
```

3. Restart your Prometheus container

4. Add infrastructure metrics to your Grafana dashboard:
   - CPU Usage: `100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)`
   - Disk Usage: `100 - ((node_filesystem_avail_bytes{mountpoint="/"} * 100) / node_filesystem_size_bytes{mountpoint="/"})`

### 4.2 Application Monitoring

Application monitoring focuses on the internal operations and performance of your application.

We've already implemented some application monitoring with our Flask metrics. Let's add a few more:

1. Add a new metric to track database query time (simulated):

```python
DB_QUERY_TIME = Histogram('db_query_duration_seconds', 'Database query duration in seconds')

@app.route('/db-query')
def db_query():
    REQUESTS.labels(method='GET', endpoint='/db-query').inc()
    query_time = random.uniform(0.1, 2.0)
    DB_QUERY_TIME.observe(query_time)
    time.sleep(query_time)  # Simulate DB query
    return f"DB query completed in {query_time:.2f} seconds"
```

2. Add this metric to your Grafana dashboard:
   - DB Query Time: `rate(db_query_duration_seconds_sum[5m]) / rate(db_query_duration_seconds_count[5m])`

### 4.3 Log Monitoring

Log monitoring involves collecting, storing, and analyzing log data from your applications and infrastructure.

Let's add some basic log monitoring:

1. Add logging to your Flask app:

```python
import logging

logging.basicConfig(filename='app.log', level=logging.INFO)

@app.route('/log-test')
def log_test():
    REQUESTS.labels(method='GET', endpoint='/log-test').inc()
    logging.info('This is a test log entry')
    return "Log entry created"
```

2. Use `docker logs` to view the logs:

```bash
docker logs flask-container
```

In a production environment, you would typically use a log aggregation tool like ELK Stack (Elasticsearch, Logstash, Kibana) or Graylog.

### 4.4 Network Monitoring

Network monitoring focuses on the performance and reliability of network connections.

Let's add a simple network monitoring component:

1. Add a new endpoint to check network connectivity:

```python
import subprocess

NETWORK_LATENCY = Gauge('network_latency_ms', 'Network latency in milliseconds', ['destination'])

@app.route('/network-check')
def network_check():
    REQUESTS.labels(method='GET', endpoint='/network-check').inc()
    destinations = ['google.com', 'github.com', 'example.com']
    results = {}
    for dest in destinations:
        try:
            output = subprocess.check_output(['ping', '-c', '1', dest]).decode()
            latency = float(output.split('time=')[1].split()[0])
            NETWORK_LATENCY.labels(destination=dest).set(latency)
            results[dest] = latency
        except:
            results[dest] = 'Failed'
    return str(results)
```

2. Add network latency to your Grafana dashboard:
   - Network Latency: `network_latency_ms`

## 5. Hands-on Exercise: Implementing End-to-End Monitoring (30 minutes) 🏗️

In this exercise, you'll implement a basic end-to-end monitoring solution for a multi-service application.

1. Create a new Docker Compose file `docker-compose.yml`:

```yaml
version: '3'
services:
  web:
    build: .
    ports:
      - "5000:5000"
  db:
    image: postgres:13
    environment:
      POSTGRES_PASSWORD: example
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
```

2. Update your `prometheus.yml`:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'flask-app'
    static_configs:
      - targets: ['web:5000']
  - job_name: 'postgres'
    static_configs:
      - targets: ['db:9187']
```

3. Add a Postgres exporter to your `docker-compose.yml`:

```yaml
  postgres-exporter:
    image: wrouesnel/postgres_exporter
    environment:
      DATA_SOURCE_NAME: "postgresql://postgres:example@db:5432/postgres?sslmode=disable"
    ports:
      - "9187:9187"
```

4. Start your multi-service application:

```bash
docker-compose up -d
```

5. Configure Grafana to use Prometheus as a data source

6. Create a new dashboard in Grafana with panels for:
   - Application metrics (request rate, error rate, latency)
   - Database metrics (active connections, transaction rate)
   - Infrastructure metrics (CPU usage, memory usage)

## Conclusion and Next Steps 🎯

In this lecture, we've explored telemetry, metrics, and various types of monitoring in a DevOps context. We've enhanced our Flask application with more detailed metrics, set up infrastructure monitoring, and implemented a basic end-to-end monitoring solution for a multi-service application.

For the next lecture, we'll dive deeper into end-user monitoring, exploring techniques for monitoring the user experience and front-end performance.

## Additional Resources 📚

1. Prometheus Documentation: https://prometheus.io/docs/introduction/overview/
2. Grafana Documentation: https://grafana.com/docs/grafana/latest/
3. "Site Reliability Engineering" by Google: https://sre.google/sre-book/monitoring-distributed-systems/
4. "Practical Monitoring" by Mike Julian

