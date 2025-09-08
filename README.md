# 📊 Monitoring Setup - ReNoteAI

This repository contains the monitoring stack configuration for **ReNoteAI**.  
We are using **Prometheus** for metrics collection and **Loki + Promtail** for log aggregation.  
Together, they provide a complete **observability solution** when integrated with Grafana.

---

## 📂 Folder Structure

```plaintext
Monitoring/
├── promstack/
│   └── prometheus.yml
│
└── lokistack/
    ├── loki-config.yaml
    └── promtail-config.yaml
```
## 📑 promstack(Prometheus)

``` plaintext
⚙️ Prometheus Configuration

File: Monitoring/promstack/prometheus.yml

global:
  scrape_interval: 15s  # how often to scrape targets

scrape_configs:
  # Monolithic service names as the fastapi_app job 
  - job_name: 'fastapi_app'
    metrics_path: /metrics
    static_configs:
      - targets: ['10.7.1.4:5000']   # Monolithic application running on port 5000

  # GENAI job
  - job_name: 'genai_app'
    metrics_path: /metrics
    static_configs:
      - targets: ['10.7.1.4:8000']   # GenAI application running on port 8000
 
  # Node Exporter (VM metrics: CPU, Memory)
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['10.7.1.4:9100']


# 🔎 Explanation of Jobs

fastapi_app

Target: 10.7.1.4:5000

Monitors the monolithic FastAPI application.

Collects API-level metrics such as request counts, latencies, and errors.

genai_app

Target: 10.7.1.4:8000

Monitors the GenAI service.

Tracks performance and usage of AI-related features.

node_exporter

Target: 10.7.1.4:9100

Monitors VM infrastructure metrics via Node Exporter:

CPU usage

Memory usage

Disk I/O

Network statistics


        +--------------------+
        |  FastAPI App       |
        |  (10.7.1.4:5000)   |
        +--------------------+
                 |
                 |
        +--------------------+
        |  GenAI App         |
        |  (10.7.1.4:8000)   |
        +--------------------+
                 |
                 |
        +--------------------+
        |  Node Exporter     |
        |  (10.7.1.4:9100)   |
        +--------------------+
                 |
                 v
        +--------------------+
        |   Prometheus       |
        |   (promstack)      |
        +--------------------+
```

# 📜 Loki Stack Documentation

This folder contains the **Loki** and **Promtail** setup for centralized log collection, storage, and querying.

---

## 📂 Folder Structure
lokistack/
├── loki-config.yaml # Loki configuration file
└── promtail-config.yaml # Promtail configuration file


---

## 🛠️ Loki Configuration (`loki-config.yaml`)

Loki stores and indexes logs for querying.

### Key Settings:
- **Server**  
  - `http_listen_port: 3100` → Loki API endpoint  
  - `grpc_listen_port: 9096` → gRPC endpoint  
  - `log_level: debug` → Verbose logs for debugging  

- **Storage**  
  - `path_prefix: /var/lib/loki` → Data storage location  
  - `chunks_directory` → Stores log data chunks  
  - `rules_directory` → Stores alert rules  

- **Retention**  
  - `retention_period: 720h` → Keep logs for 30 days  

- **Schema**  
  - `v13` schema with daily index partitioning  

- **Compactor**  
  - Cleans up old log data automatically  

---

## 📜 Promtail Configuration (`promtail-config.yaml`)

Promtail collects logs from files and pushes them to Loki.

### Key Settings:
- **Server**
  - `http_listen_port: 9080` → Promtail web server  
  - `grpc_listen_port: 0` → gRPC disabled  

- **Positions**  
  - Tracks last read position to prevent duplicate logs  

- **Clients**  
  - Sends logs to `http://loki:3100/loki/api/v1/push`  

- **Scrape Jobs**  
  | Job Name                  | Path                        | Labels Used               |
  |---------------------------|-----------------------------|----------------------------|
  | `system`                  | `/var/log/*log`              | `job: varlogs`             |
  | `renote_monolithic_logs`   | `/mnt/logs/renote/*.log, *.txt` | `job: renote_monolithic_logs` |
  | `genai_logs`               | `/mnt/logs/genai/*.log, *.txt`  | `job: genai_logs`          |

---

## Create the required directories and give permissions 

```plaintext

# For Prometheus data
sudo mkdir -p /var/lib/prometheus

# For Loki data
sudo mkdir -p /var/lib/loki

# For Logs (Promtail reads from these)
sudo mkdir -p /mnt/logs/renote
sudo mkdir -p /mnt/logs/genai

## Permissions

sudo chown -R 65534:65534 /var/lib/prometheus
sudo chown -R 10001:10001 /var/lib/loki
```


## Running the all services 

# 1️⃣ Run Prometheus
```plaintext
docker run -d --name prometheus --restart=always \
  --network milvus \
  -p 9090:9090 \
  -v /home/azureuser/Monitoring/promstack/prometheus.yml:/etc/prometheus/prometheus.yml:ro \
  -v /var/lib/prometheus:/prometheus \
  prom/prometheus

Explanation:

docker run → Start a new container

-d → Run in background (detached mode)

--name prometheus → Container name = prometheus

--restart=always → Auto-restart on reboot or failure

--network milvus → Connect to milvus Docker network for communication with other containers

-p 9090:9090 → Map host port 9090 → container port 9090 (Prometheus web UI)

-v /home/azureuser/Monitoring/promstack/prometheus.yml:/etc/prometheus/prometheus.yml:ro
→ Mount local config file into container (read-only)

-v /var/lib/prometheus:/prometheus → Persistent data directory for metrics storage

prom/prometheus → Official Prometheus image

```

# 2️⃣ Run Loki

```plaintext

Command:
docker run -d --name loki --restart=always \
  --network milvus \
  -p 3100:3100 \
  -v /home/azureuser/Monitoring/lokistack/loki-config.yaml:/etc/loki/config.yml:ro \
  -v /var/lib/loki:/var/lib/loki \
  grafana/loki:3.4.1 \
  -config.file=/etc/loki/config.yml


Explanation:

--name loki → Container name = loki

--restart=always → Auto-restart on reboot/failure

--network milvus → Same network so Promtail can talk to Loki

-p 3100:3100 → Map host port 3100 → container port 3100 (Loki API/UI)

-v /home/azureuser/Monitoring/lokistack/loki-config.yaml:/etc/loki/config.yml:ro
→ Mount Loki config file into container (read-only)

-v /var/lib/loki:/var/lib/loki → Persistent data directory for log storage

grafana/loki:3.4.1 → Official Loki image, version 3.4.1

-config.file=/etc/loki/config.yml → Tell Loki where the config file is

```

# 3️⃣ Run Promtail

```plaintext

Command:

docker run -d --name promtail \
  --network milvus \
  -v /home/azureuser/Monitoring/lokistack:/mnt/config:ro \
  -v /mnt/logs/renote:/mnt/logs/renote:ro \
  -v /mnt/logs/genai:/mnt/logs/genai:ro \
  grafana/promtail:3.4.1 \
  -config.file=/mnt/config/promtail-config.yaml


Explanation:

--name promtail → Container name = promtail

--network milvus → So Promtail can push logs to Loki inside same network

-v /home/azureuser/Monitoring/lokistack:/mnt/config:ro → Mount config folder (read-only)

-v /mnt/logs/renote:/mnt/logs/renote:ro → Mount logs from renote app (read-only)

-v /mnt/logs/genai:/mnt/logs/genai:ro → Mount logs from genai app (read-only)

grafana/promtail:3.4.1 → Official Promtail image, version 3.4.1

-config.file=/mnt/config/promtail-config.yaml → Tell Promtail which config file to use

```


