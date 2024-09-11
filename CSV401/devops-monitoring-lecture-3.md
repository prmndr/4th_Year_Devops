# Lecture 3: Network Operations Center (NOC) and Its Role in DevOps

## 1. Introduction to Network Operations Center (NOC) (10 minutes) 🏢

A Network Operations Center (NOC) is a centralized location where IT teams monitor, manage, and control the network infrastructure. In the context of DevOps, the NOC plays a crucial role in maintaining system reliability, performance, and security.

Key responsibilities of a NOC:
- Network monitoring and management
- Incident response and troubleshooting
- Performance optimization
- Security monitoring
- Capacity planning and management

## 2. Setting Up a Simple NOC Dashboard (20 minutes) 📊

Let's create a comprehensive dashboard in Grafana that simulates a NOC environment for our Flask application.

1. Open Grafana and create a new dashboard named "NOC Dashboard"

2. Add the following panels to your dashboard:

   a. System Uptime
   ```
   up{job="flask-app-uptime"}
   ```

   b. Request Rate
   ```
   sum(rate(http_requests_total[1m])) by (endpoint)
   ```

   c. Error Rate
   ```
   rate(http_errors_total[1m])
   ```

   d. Average Latency
   ```
   rate(http_request_duration_seconds_sum[1m]) / rate(http_request_duration_seconds_count[1m])
   ```

   e. Failed Login Attempts
   ```
   rate(login_attempts_total{status="failure"}[5m])
   ```

   f. Total Purchase Amount (Last Hour)
   ```
   increase(purchase_amount_total[1h])
   ```

3. Arrange these panels in a logical order and adjust their sizes for better visibility.

## 3. Simulating Network Traffic (15 minutes) 🌐

To make our NOC dashboard more realistic, let's simulate some network traffic. We'll use a simple Python script to generate requests to our Flask app.

Create a new file named `traffic_simulator.py`:

```python
import requests
import time
import random

def simulate_traffic():
    endpoints = ['/', '/simulate-load', '/simulate-error']
    while True:
        endpoint = random.choice(endpoints)
        try:
            if endpoint == '/':
                requests.get('http://localhost:5000/')
            elif endpoint == '/simulate-load':
                requests.get('http://localhost:5000/simulate-load')
            elif endpoint == '/simulate-error':
                requests.get('http://localhost:5000/simulate-error')
            
            # Simulate login attempts
            if random.random() < 0.1:  # 10% chance of login attempt
                password = 'secret' if random.random() < 0.6 else 'wrong'
                requests.post('http://localhost:5000/login', data={'password': password})
            
            # Simulate purchases
            if random.random() < 0.05:  # 5% chance of purchase
                amount = random.uniform(10, 100)
                requests.post('http://localhost:5000/purchase', data={'amount': amount})
                
        except requests.RequestException as e:
            print(f"Request failed: {e}")
        
        time.sleep(random.uniform(0.1, 1))  # Random delay between requests

if __name__ == '__main__':
    simulate_traffic()
```

Run this script in a new terminal:

```bash
python traffic_simulator.py
```

Observe the changes in your NOC dashboard as the simulated traffic flows.

## 4. Incident Response in a NOC (20 minutes) 🚨

One of the primary responsibilities of a NOC is to respond to incidents. Let's simulate an incident and walk through the response process.

1. Simulate a high error rate:
   - Modify the `/simulate-error` endpoint in your Flask app to always return an error:

```python
@app.route('/simulate-error')
def simulate_error():
    REQUESTS.labels(method='GET', endpoint='/simulate-error').inc()
    ERRORS.inc()
    return "Simulated Error", 500
```

2. Rebuild and restart your Docker container:

```bash
docker build -t flask-app .
docker stop flask-container
docker rm flask-container
docker run -d -p 5000:5000 --name flask-container flask-app
```

3. Observe the spike in error rate on your NOC dashboard

4. Incident Response Process:
   a. Detect: Notice the spike in error rate
   b. Triage: Identify the source of the errors (the `/simulate-error` endpoint)
   c. Investigate: Check the logs of your Flask container

```bash
docker logs flask-container
```

   d. Resolve: Fix the code in the `/simulate-error` endpoint
   e. Verify: Rebuild and restart the container, then check the NOC dashboard to confirm the error rate has decreased

## 5. Performance Optimization in a NOC (20 minutes) 🚀

NOCs are also responsible for identifying and resolving performance issues. Let's simulate a performance problem and optimize it.

1. Simulate high latency:
   - Modify the `/simulate-load` endpoint to increase its processing time:

```python
import time

@app.route('/simulate-load')
def simulate_load():
    REQUESTS.labels(method='GET', endpoint='/simulate-load').inc()
    start_time = time.time()
    # Simulate CPU load
    x = 0
    for i in range(10000000):  # Increased iteration count
        x += i
    time.sleep(2)  # Added sleep to simulate I/O operations
    duration = time.time() - start_time
    LATENCY.observe(duration)
    return f"Load simulated, result: {x}"
```

2. Rebuild and restart your Docker container

3. Observe the increase in average latency on your NOC dashboard

4. Optimization Process:
   a. Identify: Notice the spike in latency for the `/simulate-load` endpoint
   b. Analyze: Use a profiling tool to identify the bottleneck
   c. Optimize: Reduce the iteration count and remove the sleep
   d. Implement: Update the code and rebuild the container
   e. Verify: Check the NOC dashboard to confirm the latency has decreased

## 6. Capacity Planning in a NOC (15 minutes) 📈

Capacity planning is crucial for maintaining system performance as demand grows. Let's simulate a capacity planning scenario.

1. Add a new metric to track system load:

```python
from prometheus_client import Gauge

SYSTEM_LOAD = Gauge('system_load', 'System load average')

@app.before_request
def before_request():
    # Simulate system load (this would typically come from actual system metrics)
    SYSTEM_LOAD.set(random.uniform(0, 1))
```

2. Add a new panel to your NOC dashboard to track system load:

```
system_load
```

3. Simulate increased load:
   - Modify your `traffic_simulator.py` to generate more requests

4. Capacity Planning Process:
   a. Monitor: Observe the trend in system load over time
   b. Predict: Use the trend to estimate future resource needs
   c. Plan: Determine when additional resources will be needed
   d. Implement: In a real scenario, this might involve scaling your application or infrastructure

## 7. Hands-on Exercise: Building a Custom NOC Tool (20 minutes) 🛠️

In this exercise, you'll create a simple custom NOC tool using Python and the Prometheus client library.

1. Create a new file named `noc_tool.py`:

```python
import time
import requests
from prometheus_client import CollectorRegistry, Gauge, push_to_gateway

# Initialize Prometheus metrics
registry = CollectorRegistry()
error_rate = Gauge('custom_error_rate', 'Custom error rate metric', registry=registry)
latency = Gauge('custom_latency', 'Custom latency metric', registry=registry)

def check_endpoint(url):
    try:
        start_time = time.time()
        response = requests.get(url)
        duration = time.time() - start_time
        
        latency.set(duration)
        if response.status_code != 200:
            error_rate.inc()
        
        return True
    except requests.RequestException:
        error_rate.inc()
        return False

def main():
    while True:
        check_endpoint('http://localhost:5000')
        check_endpoint('http://localhost:5000/simulate-load')
        check_endpoint('http://localhost:5000/simulate-error')
        
        # Push metrics to Prometheus Pushgateway
        push_to_gateway('localhost:9091', job='custom_noc_tool', registry=registry)
        
        time.sleep(5)

if __name__ == '__main__':
    main()
```

2. Run a Prometheus Pushgateway:

```bash
docker run -d --name pushgateway -p 9091:9091 prom/pushgateway
```

3. Update your `prometheus.yml` to scrape from the Pushgateway:

```yaml
scrape_configs:
  # ... (previous configs) ...
  - job_name: 'pushgateway'
    static_configs:
      - targets: ['host.docker.internal:9091']
```

4. Restart your Prometheus container

5. Run your custom NOC tool:

```bash
python noc_tool.py
```

6. Add the custom metrics to your NOC dashboard in Grafana

## Conclusion and Next Steps 🎯

In this lecture, we've explored the role of a Network Operations Center in a DevOps environment. We've set up a NOC dashboard, simulated network traffic and incidents, and even built a custom NOC tool. These skills are crucial for maintaining the health and performance of complex systems in a DevOps context.

For the next lecture, we'll dive deeper into telemetry and metrics, exploring how to gather and utilize more detailed system and application data.

## Additional Resources 📚

1. "The Practice of System and Network Administration" by Thomas A. Limoncelli
2. Prometheus Pushgateway: https://github.com/prometheus/pushgateway
3. Grafana Tutorials: https://grafana.com/tutorials/

