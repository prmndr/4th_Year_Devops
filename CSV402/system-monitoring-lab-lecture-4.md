# Lab Lecture 4: Advanced Grafana Dashboards and Synthetic Monitoring

## Objective
In this lab, students will create advanced Grafana dashboards for various metrics and implement synthetic monitoring using Puppeteer.

## Duration: 1 hour

## Prerequisites
- Completed Lab Lectures 1, 2, and 3
- Basic understanding of JavaScript

## Steps

### 1. Creating Advanced Grafana Dashboards (25 minutes)

1. Access Grafana at `http://localhost:3000` and log in

2. Create a new dashboard by clicking "Create" > "Dashboard"

3. Add a System Overview panel:
   - Click "Add new panel"
   - For the metric, use: `100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)`
   - Panel title: "CPU Usage"
   - Visualization: Gauge

4. Add a Memory Usage panel:
   - Click "Add new panel"
   - For the metric, use: `100 * (1 - ((node_memory_MemFree_bytes + node_memory_Cached_bytes + node_memory_Buffers_bytes) / node_memory_MemTotal_bytes))`
   - Panel title: "Memory Usage"
   - Visualization: Gauge

5. Add a Disk Usage panel:
   - Click "Add new panel"
   - For the metric, use: `100 - ((node_filesystem_avail_bytes{mountpoint="/"} * 100) / node_filesystem_size_bytes{mountpoint="/"})`
   - Panel title: "Disk Usage"
   - Visualization: Gauge

6. Add a Network Traffic panel:
   - Click "Add new panel"
   - For the metric, use: `rate(node_network_receive_bytes_total[5m])`
   - Add another query with: `rate(node_network_transmit_bytes_total[5m])`
   - Panel title: "Network Traffic"
   - Visualization: Graph

7. Add a panel for your custom metric:
   - Click "Add new panel"
   - For the metric, use: `app_random_value`
   - Panel title: "Custom Metric"
   - Visualization: Graph

8. Arrange the panels in a logical order and adjust their sizes

9. Save the dashboard

### 2. Setting up Synthetic Monitoring with Puppeteer (25 minutes)

1. Create a new directory for synthetic monitoring:

```bash
mkdir synthetic-monitoring
cd synthetic-monitoring
```

2. Initialize a new Node.js project:

```bash
npm init -y
```

3. Install Puppeteer and Prometheus client:

```bash
npm install puppeteer prom-client
```

4. Create a new file `synthetic_monitor.js`:

```javascript
const puppeteer = require('puppeteer');
const prometheus = require('prom-client');
const http = require('http');

// Create a Registry to register the metrics
const register = new prometheus.Registry();

// Create a gauge for page load time
const pageLoadTimeGauge = new prometheus.Gauge({
  name: 'page_load_time_ms',
  help: 'Time taken to load the page in milliseconds',
  registers: [register]
});

// Create a gauge for HTTP status code
const httpStatusGauge = new prometheus.Gauge({
  name: 'http_status_code',
  help: 'HTTP status code of the page',
  registers: [register]
});

async function runTest() {
  const browser = await puppeteer.launch();
  const page = await browser.newPage();

  const startTime = Date.now();
  const response = await page.goto('http://localhost:3000');
  const loadTime = Date.now() - startTime;

  pageLoadTimeGauge.set(loadTime);
  httpStatusGauge.set(response.status());

  await browser.close();
}

// Run the test every minute
setInterval(runTest, 60000);

// Expose the metrics
const server = http.createServer(async (req, res) => {
  if (req.url === '/metrics') {
    res.setHeader('Content-Type', register.contentType);
    res.end(await register.metrics());
  }
});

server.listen(8080);
console.log('Synthetic monitoring server listening on port 8080');
```

5. Create a `Dockerfile` for the synthetic monitoring:

```dockerfile
FROM node:14

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

CMD ["node", "synthetic_monitor.js"]
```

6. Update your `docker-compose.yml` to include the synthetic monitoring service:

```yaml
  synthetic-monitor:
    build:
      context: ./synthetic-monitoring
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
```

7. Update `prometheus/prometheus.yml` to scrape the synthetic monitoring metrics:

```yaml
scrape_configs:
  # ... (previous jobs)
  - job_name: 'synthetic-monitor'
    static_configs:
      - targets: ['synthetic-monitor:8080']
```

### 3. Starting the Services (5 minutes)

Restart the Docker containers:

```bash
docker-compose up -d
```

### 4. Creating a Synthetic Monitoring Dashboard (5 minutes)

1. In Grafana, create a new dashboard
2. Add a panel for `page_load_time_ms`
3. Add another panel for `http_status_code`

## Exercise (10 minutes)
1. Modify the synthetic monitoring script to check another endpoint (e.g., your custom exporter)
2. Add a new metric to the synthetic monitoring (e.g., check if a specific element is present on the page)
3. Create a new panel in your Grafana dashboard to display this new metric

## Conclusion
In this lab, you've created advanced Grafana dashboards that provide a comprehensive view of your system's performance. You've also implemented synthetic monitoring using Puppeteer, allowing you to proactively check the availability and performance of your web applications. These skills are essential for maintaining high-quality, reliable systems in a modern DevOps environment.

