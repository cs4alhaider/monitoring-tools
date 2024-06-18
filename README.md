
# Monitoring Tools for Dockerized Environments

This document provides an overview of various tools and Docker images to monitor server CPU, memory, storage, and other performance metrics. It includes descriptions, Docker images, metrics monitored, visualization options, setup complexity, and GitHub links.

## Tools Overview

| **Tool**            | **Description**                                                                               | **Docker Image**                       | **Metrics Monitored**                         | **Visualization**          | **Setup Complexity** | **GitHub Link**                                                              |
|---------------------|-----------------------------------------------------------------------------------------------|----------------------------------------|-----------------------------------------------|----------------------------|----------------------|------------------------------------------------------------------------------|
| **Prometheus**      | Time-series database for monitoring and alerting.                                             | `prom/prometheus`                      | CPU, Memory, Disk, Network                    | Integrated with Grafana    | Medium               | [Prometheus GitHub](https://github.com/prometheus/prometheus)               |
| **Grafana**         | Visualization tool that integrates with Prometheus.                                            | `grafana/grafana`                      | N/A (depends on data source)                  | Customizable dashboards    | Medium               | [Grafana GitHub](https://github.com/grafana/grafana)                       |
| **cAdvisor**        | Container Advisor for monitoring container resource usage.                                     | `google/cadvisor`                      | CPU, Memory, Disk, Network (container-level)  | Standalone Web UI          | Low                  | [cAdvisor GitHub](https://github.com/google/cadvisor)                      |
| **Node Exporter**   | Prometheus exporter for hardware and OS metrics.                                               | `prom/node-exporter`                   | CPU, Memory, Disk, Network                    | Integrated with Prometheus | Medium               | [Node Exporter GitHub](https://github.com/prometheus/node_exporter)        |
| **Netdata**         | Real-time performance monitoring and visualization.                                            | `netdata/netdata`                      | CPU, Memory, Disk, Network, Processes         | Standalone Web UI          | Low                  | [Netdata GitHub](https://github.com/netdata/netdata)                      |
| **Elasticsearch**   | Search and analytics engine, part of ELK stack.                                                | `docker.elastic.co/elasticsearch/elasticsearch:7.17.9` | N/A (depends on data ingested) | Kibana                       | High                 | [Elasticsearch GitHub](https://github.com/elastic/elasticsearch)           |
| **Logstash**        | Data processing pipeline for ingesting and transforming data, part of ELK stack.               | `docker.elastic.co/logstash/logstash:7.17.9`            | N/A (depends on data ingested) | N/A (feeds Elasticsearch)   | High                 | [Logstash GitHub](https://github.com/elastic/logstash)                     |
| **Kibana**          | Visualization tool for Elasticsearch, part of ELK stack.                                       | `docker.elastic.co/kibana/kibana:7.17.9`                | N/A (depends on data source)    | Customizable dashboards      | High                 | [Kibana GitHub](https://github.com/elastic/kibana)                         |
| **Metricbeat**      | Lightweight shipper for collecting and shipping system and service metrics to Elasticsearch.   | `docker.elastic.co/beats/metricbeat:7.17.9`             | CPU, Memory, Disk, Network, Processes | Kibana                       | Medium               | [Metricbeat GitHub](https://github.com/elastic/beats/tree/main/metricbeat) |

## Setup Instructions

### Prometheus & Grafana

Prometheus and Grafana are commonly used together to collect and visualize metrics.

**Docker Compose Setup:**

```yaml
version: '3'

services:
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
```

### cAdvisor

cAdvisor is useful for monitoring Docker containers' resource usage.

**Docker Compose Setup:**

```yaml
version: '3'

services:
  cadvisor:
    image: google/cadvisor
    ports:
      - "8080:8080"
    volumes:
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
```

### Node Exporter

Node Exporter works with Prometheus to collect hardware and OS metrics.

**Docker Compose Setup:**

```yaml
version: '3'

services:
  node-exporter:
    image: prom/node-exporter
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
```

### Netdata

Netdata provides real-time monitoring with a built-in web interface.

**Docker Compose Setup:**

```yaml
version: '3'

services:
  netdata:
    image: netdata/netdata
    ports:
      - "19999:19999"
    cap_add:
      - SYS_PTRACE
    security_opt:
      - apparmor=unconfined
    volumes:
      - /etc/passwd:/host/etc/passwd:ro
      - /etc/group:/host/etc/group:ro
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /etc/os-release:/host/etc/os-release:ro
```

### ELK Stack with Metricbeat

The ELK stack combined with Metricbeat provides a comprehensive logging and monitoring solution.

**Docker Compose Setup:**

```yaml
version: '3'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.17.9
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    volumes:
      - esdata:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"

  logstash:
    image: docker.elastic.co/logstash/logstash:7.17.9
    container_name: logstash
    ports:
      - "5044:5044"
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf

  kibana:
    image: docker.elastic.co/kibana/kibana:7.17.9
    container_name: kibana
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

  metricbeat:
    image: docker.elastic.co/beats/metricbeat:7.17.9
    container_name: metricbeat
    user: root
    volumes:
      - /sys/fs/cgroup:/hostfs/sys/fs/cgroup:ro
      - /proc:/hostfs/proc:ro
      - /:/hostfs:ro
      - ./metricbeat.yml:/usr/share/metricbeat/metricbeat.yml:ro
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200

volumes:
  esdata:
```

**Logstash Configuration (logstash.conf):**

```plaintext
input {
  beats {
    port => 5044
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    manage_template => false
    index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
  }
}
```

**Metricbeat Configuration (metricbeat.yml):**

```yaml
metricbeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false

output.elasticsearch:
  hosts: ["http://elasticsearch:9200"]

setup.kibana:
  host: "http://kibana:5601"
```

**Metricbeat System Module Configuration (modules.d/system.yml):**

```yaml
- module: system
  period: 10s
  metricsets:
    - cpu
    - load
    - memory
    - network
    - process
    - process_summary
    - diskio
    - filesystem
    - fsstat
  processors:
    - add_host_metadata: ~
    - add_cloud_metadata: ~
```

## Accessing the Monitoring Dashboard

- **Grafana**: Navigate to `http://localhost:3000` to access Grafana.
- **cAdvisor**: Navigate to `http://localhost:8080` to access cAdvisor.
- **Netdata**: Navigate to `http://localhost:19999` to access Netdata.
- **Kibana**: Navigate to `http://localhost:5601` to access Kibana.

These tools provide robust monitoring solutions for various needs and setups, ensuring you have visibility into your server's performance metrics and application logs.
