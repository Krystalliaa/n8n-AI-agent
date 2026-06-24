API Observability & Monitoring Homelab
📖 Overview
This project is a simple, containerized homelab designed to demonstrate core competencies in Cloud Environments, API Integrations, and System Metrics/Monitoring.

It simulates a production-like monitoring stack used by Technical Support Engineers to debug system behavior, track API health, and read metrics—skills critical for modern distributed systems.

🛠️ Tech Stack
Docker & Docker Compose: Cloud-native environment simulation (stepping stone to Kubernetes).
Prometheus: System and API metric collection.
Grafana: Data visualization and log/metric reading.
Blackbox Exporter: Probing endpoints (APIs) over HTTP/HTTPS.
⚙️ Architecture / System Integration
The system integrates three main components:

An external public API (simulating a customer or internal microservice).
Prometheus (with Blackbox Exporter) that constantly pings the API to check latency, HTTP status codes, and uptime.
Grafana, which connects to Prometheus via API to visualize the system's behavior in real-time.
🚀 Step-by-Step Tutorial: How to Run This Lab
Prerequisite
Install Docker Desktop on your machine.
Step 1: Create the Configuration Files
Clone this repo or create the following two files in an empty directory.

1. docker-compose.yml

version: '3.8'
services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin

  blackbox_exporter:
    image: prom/blackbox-exporter:latest
    ports:
      - "9115:9115"
2. prometheus.yml (Configures the integration to monitor an API)

global:
  scrape_interval: 10s

scrape_configs:
  - job_name: 'api_monitor'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
        - [https://api.coindesk.com/v1/bpi/currentprice.json](https://api.coindesk.com/v1/bpi/currentprice.json) # Simulating a FinTech/Financial API
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox_exporter:9115
Step 2: Deploy the Environment
Open your terminal in the directory where the files are and run:

docker-compose up -d
Step 3: Verify & Investigate the Environment
Once the containers are deployed, a Support Engineer's first step is to verify system health by checking the logs and application targets.

Check Container Logs We need to ensure Prometheus initialized correctly and loaded our configuration without syntax errors. Run the following command:
docker-compose logs prometheus
Expected Output: You should see logs indicating that the server is ready and the Time Series Database (TSDB) has started successfully.

Prometheus Logs

Verify API Targets in Prometheus Next, we verify that Prometheus is actively communicating with the Blackbox Exporter to probe our FinTech API.
Open your browser and navigate to http://localhost:9090/targets.

Expand the api_monitor job.

What you are looking at:

State: Should be UP.
Last Scrape: Shows the exact time since the API was last pinged (e.g., 7m 57.204s ago).
Scrape Duration: The time it took to complete the request (e.g., 273ms latency).
Interval: Confirms our configuration is running every 10s.
Prometheus API

📊 Step 4: Configure the Grafana Dashboard
With Prometheus actively scraping data, we need to connect it to Grafana to extract and visualize our system metrics.

Navigate to http://localhost:3000 in your browser.
Log in using the default credentials (Username: admin / Password: admin).
Go to Connections > Data Sources > Click Add data source > Select Prometheus.
Grafana Login

Grafana dashboard

Grafana Prometheus

Exact Configuration Settings:
Based on the Grafana UI, configure only the following fields. Leave everything else as default:

Connection > Prometheus server URL: http://prometheus:9090
Note: We use http://prometheus:9090 instead of localhost, because Grafana and Prometheus are running inside the same Docker network. They communicate using their container names as internal DNS records.

Authentication > Authentication method: No Authentication
TLS settings: Leave all unchecked (TLS is disabled for this local lab).
Other > HTTP method: POST (Default)
Grafana Prometheus Settings1

Grafana Prometheus Settings2

Scroll to the very bottom and click Save & test.
You should receive a green notification stating: "Data source is working".
Grafana Prometheus Success

🛠️ Troubleshooting & Knowledge Base
As a Technical Support Engineer, resolving issues efficiently and documenting the root cause is critical to reducing engineering escalation and enabling self-service. Below is the knowledge base documenting the common edge cases encountered during the build of this monitoring stack.

Issue 1: "Site can't be reached" when clicking the Blackbox Exporter target link
Symptom: In the Prometheus Targets UI (http://localhost:9090/targets), clicking the endpoint link results in a browser error. The address bar shows a URL starting with: http://blackbox_exporter:9115/probe?module=http_2xx&target=...
Root Cause: This is a byproduct of Docker Internal Isolation. The hostname blackbox_exporter is an internal Docker DNS alias. Prometheus (running inside the Docker network) resolves this perfectly. However, your host machine's web browser (running outside the Docker network) has no routing path for blackbox_exporter, causing the DNS lookup to fail.
Resolution: To manually test and view what the Blackbox exporter is seeing from your browser, swap out the internal Docker hostname for your local loopback address:
Fails: http://blackbox_exporter:9115/probe?...
Succeeds: http://localhost:9115/probe?...
Blackbox Error

Issue 2: Prometheus Container Fails to Start (YAML Parsing Error)
Symptom: Running docker-compose logs prometheus reveals a fatal initialization error:
yaml: line X: found character that cannot start any token
Root Cause: 1. Markdown Artifacts: Copy-pasting API URLs from documentation can inadvertently introduce hidden Markdown hyperlink formatting (e.g., [text](url) syntax), which breaks the Prometheus config parser. 2. Tab Characters: The YAML standard strictly prohibits the use of the Tab character for structural alignment. Only spaces are allowed.
Resolution: Clean the target URL string, wrap it in single quotes ('https://...') to safely escape special characters like ? and &, and ensure your text editor is configured to use spaces for indentation. Apply the changes by restarting the stack:
docker-compose down
docker-compose up -d
Issue 3: Data Source Connection Error in Grafana
Symptom: Clicking Save & test in Grafana yields a connection error or timeout.

Root Cause: Attempting to use http://localhost:9090 as the Prometheus server URL. Inside the containerized environment, localhost refers to the Grafana container itself, not the host machine or the Prometheus container.

Resolution: Change the Prometheus server URL to http://prometheus:9090. This instructs Grafana to utilize Docker's internal DNS to route traffic directly to the Prometheus service container.