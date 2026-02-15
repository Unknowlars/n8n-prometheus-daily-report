# Metric & Query Dictionary for `n8n-prometheus-infra-health-report`

This document defines each query node, the PromQL used, units, and how values are interpreted in the report pipeline.

---

## 1) Selector variables from `Configuration v3.0`

The workflow builds selector fragments once and reuses them across queries.

| Selector | Purpose |
|---|---|
| `node_selector` | Core host selector (`job` + allowed instance regex) |
| `disk_selector` | `node_selector` + disk device exclusions |
| `net_selector` | `node_selector` + network interface exclusions |
| `fs_selector` | `node_selector` + filesystem/mountpoint exclusions |

### Filtering defaults used in this workflow

- **Network device exclude regex:** `^(lo|veth.*|docker.*|br-.*|cni.*|flannel.*|tun.*|tap.*|wg.*|cali.*|tailscale.*)$`
- **Disk device exclude regex:** `^(loop.*|dm-.*|md.*|zram.*|sr[0-9]+)$`
- **Filesystem exclusions:** tmpfs/overlay/squashfs/proc/sysfs/devtmpfs/autofs, `rootfs`, runtime/docker overlay mountpoints

---

## 2) Threshold defaults

| Signal | Warn | Critical | Unit |
|---|---:|---:|---|
| CPU usage | 85 | 95 | % |
| Memory usage | 80 | 90 | % |
| Filesystem usage | 85 | 95 | % |
| TCP retransmissions (24h) | 1,000 | 50,000 | count |
| Disk latency | 50 | 100 | ms |
| Network drops (24h) | 1,000 | 10,000 | count |
| Load per CPU | 2.0 | 4.0 | ratio |

Host overrides (from config):
- `truenas`: `mem_warn=98`, `mem_crit=99`
- `aarhus-pi`: `cpu_warn=90`

---

## 3) Query catalog (raw nodes)

> Legend:  
> `Q` = instant query (`/api/v1/query`)  
> `QR` = range query (`/api/v1/query_range`)

| Node | Type | PromQL (template form) | Unit | Window / Step | Primary use |
|---|---|---|---|---|---|
| `Query: up` | Q | `up{<node_selector>}` | 0/1 | now | Host reachability |
| `Query: avail_24h` | Q | `avg_over_time(up{<node_selector>}[24h]) * 100` | % | 24h | SLA/availability |
| `Query: cpu_usage` | Q | `100 - (avg by (instance) (rate(node_cpu_seconds_total{<node_selector>,mode="idle"}[5m])) * 100)` | % | 5m rate | CPU pressure |
| `Query: cpu_trend_7d` | QR | Same as `cpu_usage` | % | 7d, 1h step | Trend graph + WoW |
| `Query: load_per_cpu` | Q | `node_load1{<node_selector>} / clamp_min(count by (instance) (node_cpu_seconds_total{<node_selector>,mode="idle"}), 1)` | ratio | now | Saturation relative to cores |
| `Query: mem_usage` | Q | `(1 - (node_memory_MemAvailable_bytes{<node_selector>} / node_memory_MemTotal_bytes{<node_selector>})) * 100` | % | now | Memory pressure |
| `Query: mem_trend_7d` | QR | Same as `mem_usage` | % | 7d, 1h step | Trend graph + WoW |
| `Query: swap_usage` | Q | `(1 - (node_memory_SwapFree_bytes{<node_selector>} / clamp_min(node_memory_SwapTotal_bytes{<node_selector>}, 1))) * 100` | % | now | Swap stress indicator |
| `Query: fs_size` | Q | `node_filesystem_size_bytes{<fs_selector>}` | bytes | now | Filesystem capacity |
| `Query: fs_avail` | Q | `node_filesystem_avail_bytes{<fs_selector>}` | bytes | now | Free filesystem space |
| `Query: disk_growth_7d` | Q | `(1 - (node_filesystem_avail_bytes{<fs_selector>} / node_filesystem_size_bytes{<fs_selector>})) * 100` | % | now | Used% baseline for prediction |
| `Query: disk_busy` | Q | `max by (instance) (rate(node_disk_io_time_seconds_total{<disk_selector>}[5m]) * 100)` | % | 5m rate | Disk utilization |
| `Query: disk_read_lat` | Q | `sum by (instance) (rate(node_disk_read_time_seconds_total{<disk_selector>}[5m]) / clamp_min(rate(node_disk_reads_completed_total{<disk_selector>}[5m]), 0.0001)) * 1000` | ms | 5m rate | Read latency |
| `Query: disk_write_lat` | Q | `sum by (instance) (rate(node_disk_write_time_seconds_total{<disk_selector>}[5m]) / clamp_min(rate(node_disk_writes_completed_total{<disk_selector>}[5m]), 0.0001)) * 1000` | ms | 5m rate | Write latency |
| `Query: net_rx` | Q | `sum by (instance) (rate(node_network_receive_bytes_total{<net_selector>}[5m])) * 8` | bps | 5m rate | Ingress bandwidth |
| `Query: net_tx` | Q | `sum by (instance) (rate(node_network_transmit_bytes_total{<net_selector>}[5m])) * 8` | bps | 5m rate | Egress bandwidth |
| `Query: net_drop_rx` | Q | `sum by (instance) (increase(node_network_receive_drop_total{<net_selector>}[24h]))` | count | 24h | Receive packet drops |
| `Query: net_drop_tx` | Q | `sum by (instance) (increase(node_network_transmit_drop_total{<net_selector>}[24h]))` | count | 24h | Transmit packet drops |
| `Query: net_err_rx` | Q | `sum by (instance) (increase(node_network_receive_errs_total{<net_selector>}[24h]))` | count | 24h | Receive errors |
| `Query: net_err_tx` | Q | `sum by (instance) (increase(node_network_transmit_errs_total{<net_selector>}[24h]))` | count | 24h | Transmit errors |
| `Query: net_interface_status` | Q | `node_network_up{<net_selector>}` | 0/1 | now | Interface state |
| `Query: tcp_estab` | Q | `sum by (instance) (node_netstat_Tcp_CurrEstab{<node_selector>})` | count | now | Active TCP sessions |
| `Query: tcp_retrans` | Q | `sum by (instance) (increase(node_netstat_Tcp_RetransSegs{<node_selector>}[24h]))` | count | 24h | Packet-loss symptom |
| `Query: uptime` | Q | `node_time_seconds{<node_selector>} - node_boot_time_seconds{<node_selector>}` | seconds | now | Uptime |
| `Query: reboots` | Q | `changes(node_boot_time_seconds{<node_selector>}[7d])` | count | 7d | Reboot activity |
| `Query: uname` | Q | `node_uname_info{<node_selector>}` | info | now | Host metadata |
| `Query: cpu_count` | Q | `count by (instance) (count by (instance, cpu) (node_cpu_seconds_total{<node_selector>,mode="idle"}))` | cores | now | Capacity normalization |
| `Query: mem_total` | Q | `node_memory_MemTotal_bytes{<node_selector>}` | bytes | now | Capacity baseline |

---

## 4) Derived fields generated in code nodes

### `Combine Metrics v6.0`
Combines all query results into per-host records and computes composite indicators, including:

- host health flags (`up`, warning/critical status)
- site and global availability percentages
- week-over-week deltas (CPU / memory)
- disk and memory pressure indicators
- network anomaly indicators (drops/errors/retrans)
- report sections and summary objects consumed by HTML renderer

### `Alert Correlation v8.0`
Builds incident list and priority actions by correlating:

- host-level threshold breaches
- service availability state
- trend-based risk flags
- retrans/drop/error spikes

### Predictive outputs (from trends logic)
- `disk_full_in_days` (or null)
- `mem_pressure_in_days` (or null)
- network saturation risk (host-level)
- anomaly flags for CPU/memory variance

---

## 5) Site/SLA mapping used by this workflow

| Site | Hosts | Target SLA |
|---|---|---:|
| copenhagen | pve, truenas, proxmox-backup-server, docker-proxmox, vm-host-docker, cachyos-valby | 99.9% |
| helsinki | ubuntu-4gb-hel1-2 | 99.5% |
| remote | aarhus-pi | 95.0% |

---

## 6) Optional custom exporter metrics (recommended additions)

If you later export workflow-level metrics back into Prometheus, consider adding:

| Metric name | Type | Labels | Description |
|---|---|---|---|
| `infraops_report_health_score_percent` | gauge | `site` | Final computed health score |
| `infraops_report_hosts_up_ratio` | gauge | `site` | Hosts up / total hosts |
| `infraops_report_alerts_active` | gauge | `severity` | Number of active incidents |
| `infraops_report_generation_duration_seconds` | gauge | `workflow` | Runtime of report pipeline |
| `infraops_report_send_success_total` | counter | `channel` | Successful report deliveries |
| `infraops_report_send_failures_total` | counter | `channel,error_class` | Delivery failures |
| `infraops_report_idempotent_skips_total` | counter | `workflow` | Duplicate-send prevented count |

---

## 7) Data quality notes

1. **Label consistency matters:** ensure `instance` labels are stable (hostname vs hostname:port).  
2. **Query windows influence behavior:** 5m rates smooth spikes; 24h increase captures incident volume.  
3. **Interface filters are intentional:** virtual/noise interfaces are excluded to reduce false positives.  
4. **Per-host overrides are expected:** especially for memory-heavy appliances like TrueNAS.  

---

## 8) Validation checklist

- [ ] `up` returns one series per expected host
- [ ] `instance` label values match `allowed_instances`
- [ ] `cpu_count` > 0 for all hosts
- [ ] `mem_total` present for all hosts
- [ ] `fs_size` and `fs_avail` available for real mountpoints
- [ ] `tcp_retrans` not missing (node_exporter netstat collector enabled)