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


