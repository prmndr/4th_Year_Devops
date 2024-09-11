# Lecture 2: Goals and Approaches to Monitoring in DevOps

## 1. Introduction (5 minutes) 🎯

In this lecture, we'll explore the specific goals of monitoring in a DevOps environment and examine various approaches to achieve these goals. We'll continue to use Docker and the tools we set up in the previous lecture (Flask, Prometheus, and Grafana) to demonstrate these concepts practically.

## 2. Goals of Monitoring in DevOps (15 minutes) 🎯

The primary goals of monitoring in DevOps include:

1. **Ensuring System Reliability**: Detect and prevent outages
2. **Performance Optimization**: Identify and resolve bottlenecks
3. **Capacity Planning**: Predict and manage resource needs
4. **Security**: Detect and respond to security threats
5. **Business Intelligence**: Derive insights for decision-making
6. **Continuous Improvement**: Support the feedback loop in CI/CD

Let's enhance our Flask application to demonstrate these goals.

## 3. Enhancing Our Flask Application (20 minutes) 💻

We'll add more endpoints and metrics to our Flask app to simulate different monitoring scenarios.

Update your `app.py`:

```python
from flask import Flask, Response, request
import time
import random
from prometheus_client import Counter, Histogram, generate_latest, CONTENT_TYPE_LATEST

app = Flask(__name__)

# Define metrics
REQUESTS = Counter('http_requests_total', 'Total HTTP Requests', ['method', 'endpoint'])
LATENCY = Histogram('http_request_duration_seconds', 'HTTP request latency in seconds')
ERRORS = Counter('http_errors_total', 'Total HTTP Errors')

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

Rebuild and restart your Docker container:

```bash
docker build -t flask-app .
docker stop flask-container
docker rm flask-container
docker run -d -p 5000:5000 --name flask-container flask-app
```

## 4. Monitoring for System Reliability (20 minutes) ⚠️

System reliability is about ensuring your application is available and functioning correctly.

1. Set up a simple uptime check in Prometheus. Add to your `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: 'flask-app'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['host.docker.internal:5000']
  - job_name: 'flask-app-uptime'
    metrics_path: '/'
    static_configs:
      - targets: ['host.docker.internal:5000']
```

Restart your Prometheus container:

```bash
docker stop prometheus
docker rm prometheus
docker run -d --name prometheus -p 9090:9090 -v ${PWD}/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus
```

2. In Grafana, create a new panel to monitor the up/down status of your app.
   - Query: `up{job="flask-app-uptime"}`

## 5. Performance Optimization (20 minutes) 🚀

Performance optimization involves identifying and resolving bottlenecks.

1. In Grafana, create a panel to monitor request latency:
   - Query: `rate(http_request_duration_seconds_sum[1m]) / rate(http_request_duration_seconds_count[1m])`

2. Create another panel to monitor request rate:
   - Query: `sum(rate(http_requests_total[1m])) by (endpoint)`

3. Generate some load:

```bash
for i in {1..100}; do curl http://localhost:5000/simulate-load & done
```

Observe the changes in your Grafana dashboard.

## 6. Capacity Planning (15 minutes) 📊

Capacity planning involves predicting and managing resource needs.

1. In Grafana, create a panel to show the total number of requests over time:
   - Query: `sum(increase(http_requests_total[1h]))`

2. Set up alerting in Grafana:
   - Create a new alert rule on your request rate panel
   - Set it to trigger if the rate exceeds a certain threshold

## 7. Security Monitoring (15 minutes) 🔒

Security monitoring helps detect and respond to security threats.

1. Add a new endpoint to simulate a login attempt:

```python
LOGIN_ATTEMPTS = Counter('login_attempts_total', 'Total login attempts', ['status'])

@app.route('/login', methods=['POST'])
def login():
    REQUESTS.labels(method='POST', endpoint='/login').inc()
    if request.form.get('password') == 'secret':
        LOGIN_ATTEMPTS.labels(status='success').inc()
        return "Login successful"
    else:
        LOGIN_ATTEMPTS.labels(status='failure').inc()
        return "Login failed", 401
```

2. In Grafana, create a panel to monitor failed login attempts:
   - Query: `rate(login_attempts_total{status="failure"}[5m])`

## 8. Business Intelligence (15 minutes) 💼

Monitoring can provide valuable business insights.

1. Add a simulated "purchase" endpoint:

```python
PURCHASE_AMOUNT = Counter('purchase_amount_total', 'Total purchase amount')

@app.route('/purchase', methods=['POST'])
def purchase():
    REQUESTS.labels(method='POST', endpoint='/purchase').inc()
    amount = float(request.form.get('amount', 0))
    PURCHASE_AMOUNT.inc(amount)
    return f"Purchase of ${amount} recorded"
```

2. In Grafana, create a panel to show total purchase amount over time:
   - Query: `increase(purchase_amount_total[1h])`

## 9. Hands-on Exercise: Implementing Continuous Improvement (15 minutes) 🔄

1. Analyze the metrics we've set up
2. Identify a "problem area" (e.g., high latency, error rate)
3. Propose and implement a solution
4. Use Grafana to verify the improvement

## Conclusion and Next Steps 🎯

In this lecture, we've explored the key goals of monitoring in DevOps and implemented practical examples for each. We've seen how to:
- Ensure system reliability
- Optimize performance
- Plan for capacity needs
- Monitor security
- Derive business insights
- Support continuous improvement

For the next lecture, we'll dive deeper into the Network Operations Center (NOC) and its role in the DevOps world.

## Additional Resources 📚

1. Site Reliability Engineering book by Google: https://sre.google/sre-book/table-of-contents/
2. Prometheus Best Practices: https://prometheus.io/docs/practices/naming/
3. Grafana Alerting: https://grafana.com/docs/grafana/latest/alerting/

