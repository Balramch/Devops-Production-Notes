// --- Loki Logs Configuration ---
local.file_match "pm2_files" {
  path_targets = [{
  //your log file path
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
  //endpoint loki url
    url = "http://your_ip:32297/loki/api/v1/push"
  }
}

// --- Prometheus Metrics Configuration ---

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
  disable_collectors = ["ipvs", "btrfs", "infiniband", "xfs", "zfs"]
  enable_collectors  = ["meminfo"]

  filesystem {
    // FIXED: Repaired broken line break split across 'securityffs'
    fs_types_exclude     = "^(autofs|binfmt_misc|bpf|cgroup2?|configfs|debugfs|devpts|devtmpfs|tmpfs|fusectl|hugetlbfs|iso9660|mqueue|nsfs|overlay|proc|procfs|pstore|rpc_pipefs|securityfs|selinuxfs|squashfs|sysfs|tracefs)$"
    // FIXED: Closed the missing double quotes and properly formatted mount_timeout
    mount_points_exclude = "^/(dev|proc|run/credentials/.+|sys|var/lib/docker/.+)($|/)"
    mount_timeout        = "5s"
  }

  netclass {
    ignored_devices = "^(veth.*|cali.*|[a-f0-9]{15})$"
  }

  netdev {
    device_exclude = "^(veth.*|cali.*|[a-f0-9]{15})$"
  }
}

// FIXED: Added missing scraper and remote write configurations for metrics
prometheus.scrape "metrics" {
  scrape_interval = "15s"
  targets         = discovery.relabel.metrics.output
  forward_to      = [prometheus.remote_write.local.receiver]
}

prometheus.remote_write "local" {
  endpoint {
    url = "http://kube-prometheus-stack-prometheus.monitoring:9090/api/v1/write"
  }
}
