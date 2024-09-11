# Lecture 1: Introduction to DevOps and Monitoring with Docker

## 1. Brief Introduction to DevOps and Monitoring (10 minutes) 🔄

DevOps combines software development (Dev) and IT operations (Ops), aiming to shorten the development lifecycle and provide continuous delivery with high software quality. Monitoring is a crucial aspect of DevOps, allowing teams to observe and track the performance of applications and infrastructure in real-time.

## 2. Setting Up Our Docker Environment (15 minutes) 🐳

Let's start by ensuring everyone has Docker Desktop running and is comfortable with basic Docker commands.

1. Open Docker Desktop
2. Open a terminal or command prompt
3. Verify Docker installation:

```bash
docker --version
docker-compose --version
```

4. Pull a simple test image:

```bash
docker pull hello-world
docker run hello-world
```

## 3. Creating a Simple Web Application to Monitor (20 minutes) 💻

We'll create a basic Flask web application that we'll use for monitoring.

1. Create a new directory for our project:

```bash
mkdir flask-monitoring-demo
cd flask-monitoring-demo
```

2. Create a file named `app.py`:

```python
from flask import Flask
import random

app = Flask(__name__)

@app.route('/')
def hello():
    return "Hello, DevOps Monitoring!"

@app.route('/simulate-load')
def simulate_load():
    # Simulate CPU load
    x = 0
    for i in range(1000000):
        x += i
    return f"Load simulated, result: {x}"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

3. Create a `Dockerfile`:

```dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY . .
RUN pip install flask prometheus_client
EXPOSE 5000
CMD ["python", "app.py"]
```

4. Build and run the Docker image:

```bash
docker build -t flask-app .
docker run -d -p 5000:5000 --name flask-container flask-app
```

5. Test the application:
   - Open a web browser and go to `http://localhost:5000`
   - You should see "Hello, DevOps Monitoring!"

## 4. Introducing Prometheus for Monitoring (30 minutes) 📊

Prometheus is an open-source monitoring and alerting toolkit. We'll set it up to monitor our Flask application.

1. Add Prometheus client library to our Flask app. Update `app.py`:

```python
from flask import Flask, Response
import random
from prometheus_client import Counter, generate_latest, CONTENT_TYPE_LATEST

app = Flask(__name__)

# Define a counter metric
REQUESTS = Counter('http_requests_total', 'Total HTTP Requests')

@app.route('/')
def hello():
    # Increment the counter
    REQUESTS.inc()
    return "Hello, DevOps Monitoring!"

@app.route('/simulate-load')
def simulate_load():
    # Increment the counter
    REQUESTS.inc()
    # Simulate CPU load
    x = 0
    for i in range(1000000):
        x += i
    return f"Load simulated, result: {x}"

@app.route('/metrics')
def metrics():
    return Response(generate_latest(), mimetype=CONTENT_TYPE_LATEST)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

2. Rebuild and run the Docker image:

```bash
docker build -t flask-app .
docker stop flask-container
docker rm flask-container
docker run -d -p 5000:5000 --name flask-container flask-app
```

3. Create a `prometheus.yml` file:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'flask-app'
    static_configs:
      - targets: ['host.docker.internal:5000']
```

4. Run Prometheus in a Docker container:

```bash
docker run -d --name prometheus -p 9090:9090 -v ${PWD}/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus
```

5. Access Prometheus UI:
   - Open a web browser and go to `http://localhost:9090`
   - You should see the Prometheus interface

6. Query the metrics:
   - In the Prometheus UI, try the query: `http_requests_total`
   - You should see the total number of requests to your Flask app

## 5. Visualizing Metrics with Grafana (30 minutes) 📈

Grafana is an open-source platform for monitoring and observability. We'll use it to create dashboards for our Prometheus metrics.

1. Run Grafana in a Docker container:

```bash
docker run -d --name grafana -p 3000:3000 grafana/grafana
```

2. Access Grafana UI:
   - Open a web browser and go to `http://localhost:3000`
   - Log in with default credentials (admin/admin)

3. Add Prometheus as a data source:
   - Go to Configuration > Data Sources > Add data source
   - Select Prometheus
   - Set URL to `http://host.docker.internal:9090`
   - Click "Save & Test"

4. Create a dashboard:
   - Click "+ Create" > Dashboard
   - Add a new panel
   - In the query, enter: `rate(http_requests_total[1m])`
   - This will show the rate of requests per minute

5. Customize the dashboard:
   - Add more panels for different metrics
   - Experiment with different visualization types

## 6. Hands-on Exercise: Monitoring in Action (15 minutes) 🏋️

1. Generate some traffic to your Flask app:
   - Open several browser tabs and refresh `http://localhost:5000` and `http://localhost:5000/simulate-load` multiple times

2. Observe the changes in Prometheus and Grafana:
   - Watch the `http_requests_total` metric increase in Prometheus
   - See the request rate change in your Grafana dashboard

3. Challenge: Add a new metric to the Flask app (e.g., response time) and visualize it in Grafana

## Conclusion and Next Steps 🎯

In this lecture, we've:
- Set up a simple Flask application in Docker
- Implemented basic monitoring with Prometheus
- Visualized metrics using Grafana

For the next lecture, we'll dive deeper into advanced Prometheus and Grafana features, and explore alerting mechanisms.

## Additional Resources 📚

1. Docker Documentation: https://docs.docker.com/
2. Prometheus Documentation: https://prometheus.io/docs/
3. Grafana Documentation: https://grafana.com/docs/
4. Flask Documentation: https://flask.palletsprojects.com/

