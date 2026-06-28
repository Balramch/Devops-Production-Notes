
# Grafana Alloy Monitoring Setup

## Overview

This guide explains how to:

* Collect PM2 application logs from an EC2 instance using Grafana Alloy
* Send logs to Loki
* Visualize logs in Grafana
* Collect system metrics using Node Exporter integration inside Alloy
* Send metrics to Prometheus
* Troubleshoot common issues

---

# Architecture

```text
PM2 Logs
   │
   ▼
Grafana Alloy
   │
   ├── Logs ─────► Loki
   │                  │
   │                  ▼
   │              Grafana
   │
   └── Metrics ───► Prometheus
                      │
                      ▼
                  Grafana
```

---

# Prerequisites

* EC2 Instance
* PM2 running applications
* Kubernetes Cluster
* Loki deployed
* Prometheus deployed
* Grafana deployed
* Grafana Alloy installed

---

# Verify Loki Endpoint

Find Loki service:

```bash
kubectl get svc -n monitoring
```

Example:

```text
loki-stack NodePort your-ip 3100:32297/TCP
```

NodePort:

```text
32297
```

Worker Node:

```bash
kubectl get nodes -o wide
```

Example:

```text
10.200.3.344
```

Loki endpoint:

```text
http://your_ip:32297/loki/api/v1/push
```

---

# Verify Loki Connectivity

Test from EC2:

```bash
curl http://your_ip:32297/ready
```

Expected:

```text
ready
```

---

# PM2 Log Collection Configuration

## Alloy Configuration

```hcl
local.file_match "pm2_files" {
  path_targets = [{
    __address__ = "localhost",
    __path__    = "/home/ubuntu/.pm2/logs/*.log",
    instance    = constants.hostname,
    job         = "pm2",
  }]
}

loki.source.file "pm2_files" {
  targets    = local.file_match.pm2_files.targets
  forward_to = [loki.write.local.receiver]
}

loki.write "local" {
  endpoint {
    url = "http://your_ip:32297/loki/api/v1/push"
  }
}
```

---

# Restart Alloy

```bash
sudo systemctl restart alloy
```

Check:

```bash
sudo systemctl status alloy
```

---

# Verify Alloy is Reading Logs

```bash
sudo journalctl -u alloy -n 100 --no-pager
```

Expected:

```text
start tailing file
```

Example:

```text
start tailing file path=/home/ubuntu/.pm2/logs/slicing-out.log
```

---

# Verify Alloy Sends Logs

```bash
curl -s http://localhost:12345/metrics | grep sent_entries_total
```

Example:

```text
loki_write_sent_entries_total 151
```

Counter should increase.

---

# Loki Queries

## All Logs

```logql
{job="pm2"}
```

---

## Specific File

```logql
{filename="/home/ubuntu/.pm2/logs/slicing-out.log"}
```

---

## Errors

```logql
{job="pm2"} |= "error"
```

---

## Last 15 Minutes Errors

```logql
{job="pm2"} |= "error"
```

Time Range:

```text
Last 15 minutes
```

---

## Exceptions

```logql
{job="pm2"} |= "Exception"
```

---

## Warnings

```logql
{job="pm2"} |= "warn"
```

---

# Debugging Loki

## Query Labels

```bash
curl http://your_ip:32297/loki/api/v1/labels
```

---

## Query Streams

```bash
curl -G \
"http://your_ip:32297/loki/api/v1/query_range" \
--data-urlencode 'query={job="pm2"}'
```

---

# Common Issue: Only Latest Logs Visible

Symptoms:

```text
Only 1 line appears
```

Cause:

```text
Alloy positions file already points to end of file.
```

Check:

```bash
cat /var/lib/alloy/data/loki.source.file.pm2_files/positions.yml
```

Example:

```yaml
positions:
  /home/ubuntu/.pm2/logs/slicing-out.log: "123456"
```

---

# Re-Ingest Historical Logs

Stop Alloy:

```bash
sudo systemctl stop alloy
```

Delete positions:

```bash
sudo rm -rf /var/lib/alloy/data/loki.source.file.pm2_files
```

Start Alloy:

```bash
sudo systemctl start alloy
```

Alloy will reread files from beginning.

---

# Why Logs Disappear Again

Reason:

```text
Positions file updates automatically.
```

Alloy only tails new log lines after initial ingestion.

This is expected behavior.

---

# Prometheus Metrics Collection

## Node Exporter Integration

```hcl
prometheus.exporter.unix "metrics" {
  enable_collectors = ["meminfo"]
}
```

---

## Relabeling

```hcl
discovery.relabel "metrics" {
  targets = prometheus.exporter.unix.metrics.targets

  rule {
    target_label = "instance"
    replacement = constants.hostname
  }

  rule {
    target_label = "job"
    replacement = "ec2-metrics"
  }
}
```

---

## Scraping

```hcl
prometheus.scrape "metrics" {
  scrape_interval = "15s"

  targets = discovery.relabel.metrics.output

  forward_to = [prometheus.remote_write.local.receiver]
}
```

---

# Prometheus Remote Write

```hcl
prometheus.remote_write "local" {
  endpoint {
    url = "http://PROMETHEUS_ENDPOINT/api/v1/write"
  }
}
```

Important:

```text
Prometheus must have remote write receiver enabled.
```

Verify:

```bash
kubectl get prometheus -n monitoring -o yaml | grep enableRemoteWriteReceiver
```

Expected:

```yaml
enableRemoteWriteReceiver: true
```

---

# Useful Grafana Dashboards

## Logs

```logql
{job="pm2"}
```

---

## Error Rate

```logql
count_over_time({job="pm2"} |= "error" [5m])
```

---

## Memory Usage

```promql
node_memory_MemAvailable_bytes
```

---

## CPU Usage

```promql
100 - (avg(irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

---

# Production Recommendations

1. Use DNS instead of Node IPs.
2. Secure Loki with authentication.
3. Enable retention policies.
4. Backup Grafana dashboards.
5. Use Alloy as a systemd service.
6. Monitor Alloy health metrics.
7. Use labels consistently.
8. Avoid using Pod IPs in production.
9. Use Kubernetes Services whenever possible.
10. Store Alloy configuration in Git.

---

# Useful Commands

```bash
sudo systemctl restart alloy
sudo systemctl status alloy
sudo journalctl -u alloy -f

curl http://localhost:12345/metrics

kubectl get pods -n monitoring
kubectl get svc -n monitoring
kubectl get endpoints -n monitoring

curl http://your_ip:32297/ready
```

---

# Troubleshooting Checklist

* Alloy service running
* Loki reachable
* Prometheus reachable
* PM2 logs readable
* Correct file path
* Positions file healthy
* NodePort reachable
* Grafana datasource healthy
* Labels available in Loki
* No Alloy errors in journal logs

Following this checklist resolves the majority of Loki + Alloy + Prometheus integration issues.
