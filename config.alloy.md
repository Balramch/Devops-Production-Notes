# Grafana Alloy Configuration for PM2 Logs and Prometheus Metrics

This guide explains how to configure **Grafana Alloy** to:

* Collect PM2 application logs
* Send logs to Loki
* Collect system metrics using Node Exporter
* Send metrics to Prometheus via Remote Write

---

## Prerequisites

* Grafana Alloy installed
* PM2 applications running
* Loki endpoint accessible
* Prometheus Remote Write Receiver enabled
* Grafana configured with Loki and Prometheus datasources

---

## Alloy Configuration

### Loki Log Collection

```alloy
local.file_match "pm2_files" {
  path_targets = [{
    // PM2 log file path
    __path__ = "/home/ubuntu/.pm2/logs/*.log",

    job      = "pm2",
    instance = constants.hostname,
  }]
}

loki.source.file "pm2_files" {
  targets    = local.file_match.pm2_files.targets
  forward_to = [loki.write.local.receiver]
}

loki.write "local" {
  endpoint {
    // Loki endpoint
    url = "http://your_ip:32297/loki/api/v1/push"
  }
}
```

---

### Prometheus Metrics Collection

```alloy
discovery.relabel "metrics" {
  targets = prometheus.exporter.unix.metrics.targets

  rule {
    target_label = "instance"
    replacement  = constants.hostname
  }

  rule {
    target_label = "job"
    replacement  = string.format("%s-metrics", constants.hostname)
  }
}

prometheus.exporter.unix "metrics" {
  disable_collectors = [
    "ipvs",
    "btrfs",
    "infiniband",
    "xfs",
    "zfs"
  ]

  enable_collectors = ["meminfo"]

  filesystem {
    fs_types_exclude = "^(autofs|binfmt_misc|bpf|cgroup2?|configfs|debugfs|devpts|devtmpfs|tmpfs|fusectl|hugetlbfs|iso9660|mqueue|nsfs|overlay|proc|procfs|pstore|rpc_pipefs|securityfs|selinuxfs|squashfs|sysfs|tracefs)$"

    mount_points_exclude = "^/(dev|proc|run/credentials/.+|sys|var/lib/docker/.+)($|/)"

    mount_timeout = "5s"
  }

  netclass {
    ignored_devices = "^(veth.*|cali.*|[a-f0-9]{15})$"
  }

  netdev {
    device_exclude = "^(veth.*|cali.*|[a-f0-9]{15})$"
  }
}

prometheus.scrape "metrics" {
  scrape_interval = "15s"

  targets    = discovery.relabel.metrics.output
  forward_to = [prometheus.remote_write.local.receiver]
}

prometheus.remote_write "local" {
  endpoint {
    url = "http://kube-prometheus-stack-prometheus.monitoring:9090/api/v1/write"
  }
}
```

---

## Restart Alloy

```bash
sudo systemctl restart alloy
```

Verify status:

```bash
sudo systemctl status alloy
```

Follow logs:

```bash
sudo journalctl -u alloy -f
```

---

## Verify Loki Log Shipping

Check Alloy metrics:

```bash
curl -s http://localhost:12345/metrics | grep sent_entries_total
```

Expected output:

```text
loki_write_sent_entries_total 150
```

The number should continuously increase as new logs are generated.

---

## Verify Prometheus Remote Write

Check Alloy logs:

```bash
sudo journalctl -u alloy -f | grep remote_write
```

You should NOT see:

```text
Failed to send batch
```

or

```text
lookup prometheus
```

---

## Test PM2 Log Collection

Generate test logs:

```bash
for i in {1..10}; do
  echo "TEST_LOG_$i $(date)" >> /home/ubuntu/.pm2/logs/app-out.log
done
```

Query in Grafana Explore:

```logql
{job="pm2"}
```

---

## Useful Loki Queries

### All PM2 Logs

```logql
{job="pm2"}
```

### Errors

```logql
{job="pm2"} |= "error"
```

### Warnings

```logql
{job="pm2"} |= "warn"
```

### Exceptions

```logql
{job="pm2"} |= "Exception"
```

### Last 15 Minutes Errors

```logql
{job="pm2"} |= "error"
```

Set Grafana time range to:

```text
Last 15 minutes
```

---

## Useful Prometheus Queries

### CPU Usage

```promql
100 - (avg(irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

### Memory Available

```promql
node_memory_MemAvailable_bytes
```

### Memory Usage %

```promql
(
  1 -
  (
    node_memory_MemAvailable_bytes
    /
    node_memory_MemTotal_bytes
  )
) * 100
```

### Disk Usage %

```promql
100 -
(
  node_filesystem_avail_bytes
  /
  node_filesystem_size_bytes
) * 100
```

---

## Troubleshooting

### Logs Not Appearing

Check Alloy is reading files:

```bash
sudo journalctl -u alloy | grep "start tailing file"
```

### Only New Logs Appear

Alloy stores read positions in:

```bash
/var/lib/alloy/data/loki.source.file.pm2_files/positions.yml
```

To re-import old logs:

```bash
sudo systemctl stop alloy

sudo rm -rf \
/var/lib/alloy/data/loki.source.file.pm2_files

sudo systemctl start alloy
```

### Prometheus Remote Write Failing

Verify receiver is enabled:

```bash
kubectl get prometheus -n monitoring -o yaml | grep enableRemoteWriteReceiver
```

Expected:

```yaml
enableRemoteWriteReceiver: true
```

---

## Architecture

```text
PM2 Logs
   │
   ▼
Grafana Alloy
   │
   ├── Loki
   │      │
   │      ▼
   │   Grafana
   │
   └── Prometheus Remote Write
            │
            ▼
       Prometheus
            │
            ▼
         Grafana
```

