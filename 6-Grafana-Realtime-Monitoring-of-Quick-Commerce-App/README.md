# Exercise – Real-Time Operations Monitoring and Alerting

## Objective

The objective of this exercise was to build a monitoring pipeline for the ZAPPTTO delivery service using Python, Prometheus, Grafana and Jenkins: simulate delivery metrics, scrape them with Prometheus, visualize them in a Grafana dashboard, and trigger automated alerts when pending deliveries or delivery time cross a threshold.

## 1. Installed Prerequisites and Created the Project Directory

Verified that Docker, Python 3.10+ and Jenkins were available, installed the Prometheus client library for Python, and created the project directory.

```bash
pip3 install prometheus-client

mkdir delivery_monitoring
cd delivery_monitoring
```

Directory structure used in the exercise:

```text
delivery_monitoring/
├── delivery_metrics.py   # Python script to simulate metrics
├── prometheus.yml        # Prometheus configuration
├── alert_rules.yml       # Alerts configuration
├── Dockerfile            # Image for the metrics app (used by the Jenkins pipeline)
└── Jenkinsfile           # Jenkins pipeline script
```

## 2. Wrote the Python Metrics Application

Created `delivery_metrics.py`, which defines four metrics (three `Gauge`s and one `Summary`), simulates delivery statuses every second, and exposes them on port `8000` through `start_http_server`.

```python
from prometheus_client import start_http_server, Summary, Gauge
import random
import time

# Metrics
total_deliveries = Gauge("total_deliveries", "Total number of deliveries")
pending_deliveries = Gauge("pending_deliveries", "Number of pending deliveries")
on_the_way_deliveries = Gauge("on_the_way_deliveries", "Number of deliveries on the way")
average_delivery_time = Summary("average_delivery_time", "Average delivery time in seconds")

# Simulate delivery statuses
def simulate_delivery():
    pending = random.randint(10, 20)  # Ensure values exceed the threshold
    on_the_way = random.randint(5, 20)
    delivered = random.randint(30, 70)
    avg_time = random.uniform(15, 45)

    total = pending + on_the_way + delivered

    print(f"[DEBUG] Total deliveries: {total}")
    print(f"[DEBUG] Pending deliveries: {pending}")
    print(f"[DEBUG] On-the-way deliveries: {on_the_way}")
    print(f"[DEBUG] Average delivery time: {avg_time:.2f} seconds")

    total_deliveries.set(total)
    pending_deliveries.set(pending)
    on_the_way_deliveries.set(on_the_way)
    average_delivery_time.observe(avg_time)

if __name__ == "__main__":
    print("[INFO] Starting the HTTP server on port 8000...")
    start_http_server(8000, addr="0.0.0.0")
    print("[INFO] HTTP server started. Simulating deliveries...")
    while True:
        simulate_delivery()
        print("[INFO] Sleeping for 1 seconds...")
        time.sleep(1)
```

Ran the script:

```bash
python3 delivery_metrics.py
```

Output recorded in the exercise:

```text
[INFO] Sleeping for 1 seconds...
[DEBUG] Total deliveries: 69
[DEBUG] Pending deliveries: 20
[DEBUG] On-the-way deliveries: 17
[DEBUG] Average delivery time: 28.26 seconds
[INFO] Sleeping for 1 seconds...
[DEBUG] Total deliveries: 96
[DEBUG] Pending deliveries: 13
[DEBUG] On-the-way deliveries: 16
[DEBUG] Average delivery time: 41.62 sec
```

Checked the metrics endpoint at `http://localhost:8000/metrics`:

```bash
curl http://localhost:8000/metrics
```

```text
# HELP python_gc_objects_collected_total Objects collected during gc
# TYPE python_gc_objects_collected_total counter
python_gc_objects_collected_total{generation="0"} 242.0
python_gc_objects_collected_total{generation="1"} 18.0
python_gc_objects_collected_total{generation="2"} 0.0
...
# HELP python_info Python platform information
# TYPE python_info gauge
python_info{implementation="CPython",major="3",minor="12",patchlevel="3",version="3.12.3"} 1.0
# HELP process_virtual_memory_bytes Virtual memory size in bytes.
# TYPE process_virtual_memory_bytes gauge
process_virtual_memory_bytes 1.81510144e+08
# HELP process_resident_memory_bytes Resident memory size in bytes.
# TYPE process_resident_memory_bytes gauge
process_resident_memory_bytes 2.3887872e+07
```

The `/metrics` endpoint is created automatically by `start_http_server` from the `prometheus_client` library. Each metric is defined by a name, a description and a type:

- **Counter** – for values that only count up.
- **Gauge** – for values that can increase or decrease.
- **Histogram** – for distributions across a fixed number of buckets.
- **Summary** – for quantiles (e.g. percentiles), and exposes `_sum` and `_count` series, which are used later to compute the average delivery time.

## 3. Configured Prometheus and Alert Rules

Created `prometheus.yml` with two scrape jobs (Prometheus itself and the delivery service) and a reference to the alert rules file.

```yaml
scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "delivery_service"
    static_configs:
      - targets: ["172.17.0.1:8000"]

rule_files:
  - /etc/prometheus/alert_rules.yml
```

Created `alert_rules.yml` with two alerts: one for high pending deliveries and one for high average delivery time.

```yaml
groups:
  - name: delivery_alerts
    rules:
      - alert: HighPendingDeliveries
        expr: pending_deliveries > 10
        for: 15s
        labels:
          severity: warning
        annotations:
          summary: "High pending deliveries"
          description: "Pending deliveries are above 10 for the last 15 second."

      - alert: HighAverageDeliveryTime
        expr: (average_delivery_time_sum / average_delivery_time_count) > 30
        labels:
          severity: critical
        annotations:
          summary: "High average delivery time"
          description: "Average delivery time is above 30 seconds for the last 15 seconds."
```

Notes on the configuration:

- `scrape_configs` is the top-level key that defines scraping jobs, `job_name` names a job, and `targets` lists the endpoints Prometheus reads metrics from.
- Prometheus sends an HTTP GET request to `/metrics` on each target at the default interval of 15 seconds.
- Because Prometheus runs in a container, the target must be the host's address. On Linux or WSL this is the `docker0` bridge IP (`172.17.0.1`); on macOS and Windows `host.docker.internal` resolves to the host machine.

Started Prometheus with both configuration files mounted:

```bash
docker run -d --name prometheus --network=host \
  -v ./prometheus.yml:/etc/prometheus/prometheus.yml \
  -v ./alert_rules.yml:/etc/prometheus/alert_rules.yml \
  prom/prometheus
```

Output recorded in the exercise:

```text
44cfd54b489e8f813ad5645095833426b12d562dbf4062226f12dc5b1be3f9465
```

Verified that the container was running:

```bash
docker ps -a | grep prom
```

```text
44cfd54b489e   prom/prometheus "/bin/prometheus --c…"   5 minutes ago    Up 5 minutes prometheus
```

Using `--network=host` makes the container behave like a native application on the host, so it is reachable through the host IP on the `docker0` interface:

```bash
ip addr show
```

```text
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:34:20:61:75 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
```

Opened Prometheus at `http://localhost:9090` and confirmed under **Status → Targets** that both the `prometheus` and `delivery_service` targets were up.

## 4. Set Up Grafana and Built the Dashboard

Started Grafana in a container on port `3000`:

```bash
docker run -d --name grafana -p 3000:3000 grafana/grafana
```

Output recorded in the exercise:

```text
d15b35389cd54e91aa9451f3093a6701934968d022ddd20e8fb382cb1f05736f
```

Noted the host IP address of the `docker0` interface, which is used as the Prometheus URL inside Grafana:

```bash
ip addr show docker0
```

```text
4: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:11:26:95:13 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
```

Steps followed in the Grafana UI:

1. Opened `http://localhost:3000` and logged in with the default credentials (`admin` / `admin`), then skipped the password change.
2. Went to **Home → Connections → Data sources**, selected **Prometheus**, set the URL to `http://172.17.0.1:9090`, left the other fields as default, and clicked **Save & Test** to verify the connection.
3. Verified targets and alerts in the Prometheus UI.
4. Created a new dashboard with the following panels:

| Panel | Title | Query |
|-------|-------|-------|
| 1 | Total Deliveries | `total_deliveries` |
| 2 | Pending Deliveries | `pending_deliveries` |
| 3 | On-the-Way Deliveries | `on_the_way_deliveries` |
| 4 | Average Delivery Time | `average_delivery_time_sum / average_delivery_time_count` |

## 5. Created the Jenkins Pipeline

Checked whether a Jenkins container already existed, and removed it if it had exited or a fresh instance was needed:

```bash
docker ps -a | grep jenkins
docker rm d350facb3a26
```

Started a fresh Jenkins instance:

```bash
docker run -d -p 8080:8080 -p 50000:50000 --name jenkins -v jenkins_home:/var/jenkins_home jenkins/jenkins:lts
```

- `-p 8080:8080` exposes the web interface on the host's port 8080.
- `-p 50000:50000` exposes the agent communication port.
- `-v jenkins_home:/var/jenkins_home` persists Jenkins data in a named volume.

Output recorded in the exercise:

```text
3ce8ccb39ac56214b1bedcf7f044a5bb33bf6c07acce124d2c21e50c1b5629f5
```

Verified that Jenkins was up and running:

```bash
docker ps -a | grep jenkins
```

```text
3ce8ccb39ac5   jenkins/jenkins:lts  "/usr/bin/tini -- /u…"   9 seconds ago   Up 8 seconds 0.0.0.0:8080->8080/tcp, :::8080->8080/tcp, 0.0.0.0:50000->50000/tcp, :::50000->50000/tcp
```

Retrieved the initial admin password from inside the container and used it to unlock Jenkins at `http://localhost:8080`:

```bash
docker exec -it jenkins bash
cat /var/jenkins_home/secrets/initialAdminPassword
exit
```

Created a `Dockerfile` so that the pipeline's `docker build -t delivery_metrics .` stage can package the metrics application:

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY delivery_metrics.py /app/
RUN pip install prometheus-client
EXPOSE 8000
CMD ["python3", "delivery_metrics.py"]
```

Created the `Jenkinsfile` with four stages: pre-check Docker, set up the workspace, build and run the application, and run Prometheus and Grafana.

```groovy
pipeline {
    agent any
    stages {
        stage('Pre-check Docker') {
            steps {
                script {
                    try {
                        // Check if Docker is installed
                        def dockerVersion = sh(script: 'docker --version', returnStdout: true).trim()
                        if (!dockerVersion) {
                            error "Docker is not installed or not in the PATH. Please install Docker."
                        }

                        // Check if Docker daemon is running
                        def dockerInfo = sh(script: 'docker info', returnStatus: true)
                        if (dockerInfo != 0) {
                            error "Docker daemon is not running. Please start the Docker service."
                        }

                        echo "Docker is available and running: ${dockerVersion}"
                    } catch (Exception e) {
                        error "Pre-check failed: ${e.message}"
                    }
                }
            }
        }
        stage('Setup Workspace') {
            steps {
                script {
                    // Define the local source path where the files are located
                    def localSourcePath = '/path/to/your/local/files' // Replace with your actual path
                    def workspacePath = env.WORKSPACE

                    // Copy necessary files into the Jenkins workspace
                    sh """
                    cp ${localSourcePath}/delivery_metrics.py ${workspacePath}/
                    cp ${localSourcePath}/prometheus.yml ${workspacePath}/
                    cp ${localSourcePath}/alert_rules.yml ${workspacePath}/
                    cp ${localSourcePath}/Dockerfile ${workspacePath}/
                    """
                }
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t delivery_metrics .'
            }
        }
        stage('Run Application') {
            steps {
                sh 'docker run -d -p 8000:8000 --name delivery_metrics delivery_metrics'
            }
        }
        stage('Run Prometheus & Grafana') {
            steps {
                sh '''
                docker run -d --name prometheus -p 9090:9090 \
                  -v $WORKSPACE/prometheus.yml:/etc/prometheus/prometheus.yml \
                  -v $WORKSPACE/alert_rules.yml:/etc/prometheus/alert_rules.yml \
                  prom/prometheus
                docker run -d --name grafana -p 3000:3000 grafana/grafana
                '''
            }
        }
    }
}
```

Created the pipeline job in Jenkins by going to **New Item → Pipeline → Pipeline script**, pasting the `Jenkinsfile` contents into the **Pipeline Script** box, saving, and running the build.

## 6. Simulated Alerts

Modified `delivery_metrics.py` to push `pending_deliveries` well above the alert threshold:

```python
pending = random.randint(50, 100)  # Increase range to exceed alert threshold
```

Restarted the Python script and Prometheus, then checked the **Alerts** page in the Prometheus UI and the dashboard in Grafana. Since `pending_deliveries > 10` stayed true for longer than the `15s` hold time, the `HighPendingDeliveries` alert moved from *Pending* to *Firing*.

## 7. Expected Outputs

1. **Metrics endpoint:** `http://localhost:8000/metrics` exposing `total_deliveries`, `pending_deliveries`, `on_the_way_deliveries` and `average_delivery_time`.
2. **Prometheus queries:** graphs for `total_deliveries`, `pending_deliveries` and `average_delivery_time`.
3. **Grafana dashboard:** visualizations for the delivery metrics and their trends.
4. **Jenkins logs:** a successful pipeline build with logs for the Docker commands.
5. **Prometheus alerts:** alerts firing for high pending deliveries (and high average delivery time).

## 8. Result

The exercise covered simulating delivery metrics with the Python `prometheus_client` library, scraping them with Prometheus, defining alert rules for high pending deliveries and high average delivery time, visualizing the metrics in a four-panel Grafana dashboard, and automating the setup with a Jenkins pipeline. Raising the simulated pending deliveries above the threshold caused the alert to fire, showing that the pipeline can detect operational problems in near real time.