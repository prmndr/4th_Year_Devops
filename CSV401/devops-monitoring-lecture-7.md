# Lecture 7: Infrastructure Monitoring - Overview and Components

## 1. Introduction to Infrastructure Monitoring (15 minutes) 🏗️

Infrastructure monitoring is a critical aspect of DevOps practices, focusing on tracking the performance, availability, and resource utilization of the systems that support your applications. It provides insights that are crucial for maintaining system reliability, optimizing performance, and planning for future growth.

Key aspects of infrastructure monitoring:
- Hardware and virtual resource monitoring
- Network performance tracking
- Operating system metrics
- Storage and database performance
- Capacity planning and forecasting
- Alerting and incident response

The importance of infrastructure monitoring in DevOps:
1. Proactive issue detection and resolution
2. Capacity planning and resource optimization
3. Performance bottleneck identification
4. Facilitating collaboration between development and operations teams
5. Supporting continuous improvement and deployment

## 2. Components of Infrastructure Monitoring (30 minutes) 🧩

### 2.1 Compute Resources

Monitoring compute resources is essential for understanding the workload on your systems and identifying potential bottlenecks.

Key metrics:
- CPU usage (overall and per-core)
- CPU load average (1, 5, and 15-minute averages)
- Memory utilization (used, free, cached, buffered)
- Swap usage
- Process counts and states
- Context switches and interrupts

Example: Monitoring CPU usage with Python's psutil library

```python
import psutil

def get_cpu_usage():
    return psutil.cpu_percent(interval=1, percpu=True)

def get_memory_usage():
    mem = psutil.virtual_memory()
    return {
        "total": mem.total,
        "available": mem.available,
        "percent": mem.percent,
        "used": mem.used,
        "free": mem.free
    }

print("CPU Usage:", get_cpu_usage())
print("Memory Usage:", get_memory_usage())
```

### 2.2 Network

Network monitoring is crucial for ensuring connectivity, identifying bottlenecks, and maintaining application performance.

Key metrics:
- Bandwidth utilization (inbound and outbound)
- Latency and packet loss
- Connection states (ESTABLISHED, TIME_WAIT, etc.)
- Error rates and packet drops
- DNS query performance
- SSL/TLS handshake times

Example: Basic network monitoring with Python

```python
import psutil
import time

def get_network_stats():
    net_io = psutil.net_io_counters()
    return {
        "bytes_sent": net_io.bytes_sent,
        "bytes_recv": net_io.bytes_recv,
        "packets_sent": net_io.packets_sent,
        "packets_recv": net_io.packets_recv,
        "errin": net_io.errin,
        "errout": net_io.errout,
        "dropin": net_io.dropin,
        "dropout": net_io.dropout
    }

start = get_network_stats()
time.sleep(1)
end = get_network_stats()

print("Network usage in the last second:")
print(f"Bytes sent: {end['bytes_sent'] - start['bytes_sent']}")
print(f"Bytes received: {end['bytes_recv'] - start['bytes_recv']}")
```

### 2.3 Storage

Storage monitoring helps prevent disk space issues and identifies I/O bottlenecks that can impact application performance.

Key metrics:
- Disk space usage (used, free, inodes)
- I/O operations per second (IOPS)
- Read/write latency
- Disk queue length
- File system metadata operations

Example: Monitoring disk usage with Python

```python
import psutil

def get_disk_usage():
    partitions = psutil.disk_partitions()
    usage = {}
    for partition in partitions:
        try:
            partition_usage = psutil.disk_usage(partition.mountpoint)
            usage[partition.mountpoint] = {
                "total": partition_usage.total,
                "used": partition_usage.used,
                "free": partition_usage.free,
                "percent": partition_usage.percent
            }
        except PermissionError:
            continue
    return usage

print("Disk Usage:", get_disk_usage())
```

### 2.4 Operating System

OS-level monitoring provides insights into the overall health and performance of your systems.

Key metrics:
- System load average
- System uptime
- Open file descriptors
- System logs and events
- User sessions
- Scheduled tasks and cron jobs

Example: Monitoring system load and uptime with Python

```python
import psutil
import datetime

def get_system_info():
    return {
        "load_avg": psutil.getloadavg(),
        "boot_time": datetime.datetime.fromtimestamp(psutil.boot_time()).strftime("%Y-%m-%d %H:%M:%S"),
        "uptime": datetime.timedelta(seconds=int(time.time() - psutil.boot_time()))
    }

print("System Info:", get_system_info())
```

### 2.5 Virtualization/Containerization

In modern infrastructures, monitoring virtualization and containerization layers is crucial for understanding resource allocation and usage.

Key metrics:
- Host resource usage and overcommitment
- Guest/container resource allocation and usage
- Hypervisor/container runtime metrics
- Container orchestration platform metrics (e.g., Kubernetes cluster state)

Example: Basic Docker container monitoring with Python

```python
import docker

client = docker.from_env()

def get_container_stats():
    stats = {}
    for container in client.containers.list():
        try:
            container_stats = container.stats(stream=False)
            stats[container.name] = {
                "cpu_usage": container_stats["cpu_stats"]["cpu_usage"]["total_usage"],
                "memory_usage": container_stats["memory_stats"]["usage"],
                "network_rx": container_stats["networks"]["eth0"]["rx_bytes"],
                "network_tx": container_stats["networks"]["eth0"]["tx_bytes"]
            }
        except:
            continue
    return stats

print("Container Stats:", get_container_stats())
```

## 3. Implementing Comprehensive Infrastructure Monitoring (40 minutes) 💻

Let's set up a more comprehensive infrastructure monitoring solution using Prometheus, Node Exporter, and cAdvisor for container monitoring.

1. Run Node Exporter:

```bash
docker run -d --name node-exporter --net="host" --pid="host" -v "/:/host:ro,rslave" quay.io/prometheus/node-exporter:latest --path.rootfs=/host
```

2. Run cAdvisor:

```bash
docker run -d --name cadvisor -v /:/rootfs:ro -v /var/run:/var/run:ro -v /sys:/sys:ro -v /var/lib/docker/:/var/lib/docker:ro -v /dev/disk/:/dev/disk:ro -p 8080:8080 gcr.io/cadvisor/cadvisor:v0.47.0
```

3. Update Prometheus configuration (`prometheus.yml`):

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['localhost:8080']

  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

4. Restart Prometheus:

```bash
docker restart prometheus
```

5. Add comprehensive infrastructure metrics to Grafana:

- CPU Usage (per core):
  ```
  100 - (avg by (cpu) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
  ```

- Memory Usage:
  ```
  100 * (1 - ((node_memory_MemAvailable_bytes{instance="localhost:9100"} or node_memory_Buffers_bytes{instance="localhost:9100"} + node_memory_Cached_bytes{instance="localhost:9100"} + node_memory_MemFree_bytes{instance="localhost:9100"}) / node_memory_MemTotal_bytes{instance="localhost:9100"}))
  ```

- Disk I/O:
  ```
  rate(node_disk_read_bytes_total[5m])
  rate(node_disk_written_bytes_total[5m])
  ```

- Network Traffic (per interface):
  ```
  rate(node_network_receive_bytes_total[5m])
  rate(node_network_transmit_bytes_total[5m])
  ```

- Container CPU Usage:
  ```
  sum(rate(container_cpu_usage_seconds_total{image!=""}[5m])) by (name)
  ```

- Container Memory Usage:
  ```
  container_memory_usage_bytes{image!=""}
  ```

## 4. Hands-on Exercise: Advanced Infrastructure Dashboard (30 minutes) 🏗️

Create an advanced infrastructure dashboard in Grafana that includes:

1. System Overview:
   - CPU, Memory, Disk, and Network usage overview
   - System load and uptime

2. Detailed CPU Metrics:
   - Usage per core
   - User vs. System time
   - Load average trends

3. Memory Deep Dive:
   - Used, Free, Cached, and Buffer memory
   - Swap usage and trend

4. Disk Performance:
   - I/O operations per second
   - Read/Write latency
   - Disk space usage per mount point

5. Network Analysis:
   - Traffic per interface
   - Packet rate and errors
   - Connection states

6. Container Overview:
   - Number of running containers
   - Top CPU and Memory consuming containers
   - Container restart count

Example dashboard layout:

```
+---------------------------+
|     System Overview       |
+-------------+-------------+
|  CPU Detail | Memory Deep |
|             |    Dive     |
+-------------+-------------+
|    Disk     |   Network   |
| Performance |  Analysis   |
+-------------+-------------+
|    Container Overview     |
+---------------------------+
```

## 5. Best Practices for Infrastructure Monitoring (15 minutes) 🌟

1. Define clear monitoring objectives and SLOs
2. Implement a tiered monitoring approach (infrastructure, application, business metrics)
3. Use automation for monitoring setup and configuration
4. Implement proper alerting with clear escalation paths
5. Regularly review and update monitoring strategies
6. Correlate metrics across different components for better insights
7. Implement log aggregation alongside metric monitoring
8. Use visualization and dashboards effectively
9. Ensure monitoring system scalability
10. Integrate monitoring into the CI/CD pipeline

## Conclusion and Next Steps 🎯

We've covered comprehensive infrastructure monitoring, including key components, metrics, and a practical implementation using Prometheus, Node Exporter, and cAdvisor. We've also created an advanced Grafana dashboard and discussed best practices. In the next lecture, we'll explore advanced infrastructure monitoring concepts, including distributed systems monitoring and cloud-native monitoring approaches.

## Additional Resources 📚

1. Prometheus documentation: https://prometheus.io/docs/introduction/overview/
2. Grafana dashboard examples: https://grafana.com/grafana/dashboards/
3. "Site Reliability Engineering" book by Google - Chapter on Monitoring Distributed Systems
4. "Effective Monitoring and Alerting" by Slawek Ligus
5. cAdvisor documentation: https://github.com/google/cadvisor

