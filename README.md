# InfraOps System report with metrics from Promthues in n8n
### Daily Infrastructure Health Report (n8n Workflow)

Automated daily health reporting for infrastructure using **n8n + Prometheus/VictoriaMetrics + optional LLM summarization**.  
The workflow collects host metrics, correlates incidents, predicts capacity risks, and sends a polished HTML email report.

## Screenshots

<div align="center">
  <img src="https://raw.githubusercontent.com/Unknowlars/n8n-prometheus-daily-report/main/screenshots/report_1.png" alt="Home Dashboard" width="80%" />
  <p><em>Executive summary</em></p>
</div>

<div align="center">
  <img src="https://raw.githubusercontent.com/Unknowlars/n8n-prometheus-daily-report/main/screenshots/report_2.png" alt="Container Management" width="80%" />
  <p><em>Site SLA & Actiocs</em></p>
</div>

<div align="center">
  <img src="https://raw.githubusercontent.com/Unknowlars/n8n-prometheus-daily-report/main/screenshots/report_3.png" alt="Container Management" width="80%" />
  <p><em>infrastructure Metrics</em></p>
</div>

---

## What this workflow does

This workflow generates a daily **System** report with:

- ✅ Host availability and uptime overview
- ✅ Site-level SLA compliance (Copenhagen / Helsinki / Remote)
- ✅ CPU, memory, disk, and network health
- ✅ Week-over-week trend comparison
- ✅ Predictive insights:
  - Disk full risk (`disk_full_in_days`)
  - Memory leak risk (`mem_pressure_in_days`)
  - Network saturation risk
  - CPU anomaly detection
- ✅ Priority actions with concrete remediation commands
- ✅ Optional LLM-generated executive summary (with fallback handling)
- ✅ Idempotent delivery (avoid duplicate sends per day)

---

## Example output (from a real run)

**Status:** ATTENTION (Critical)  
**Health score:** 88%  
**Hosts up:** 7/8  
**Active alerts:** 2  

### Key findings

- Copenhagen SLA below target (83.33% vs 99.9%)
- High TCP retransmissions on `truenas` (network instability)
- `cachyos-valby` host down/unreachable
- No immediate disk-full or memory-leak predictions

---

## Workflow architecture

```text
Manual Trigger
  → Configuration
  → Prometheus Queries (host, cpu, mem, fs, disk, net, tcp, trends...)
  → Combine Metrics
  → Alert Correlation 
  → Should Skip LLM?
      ├─ Yes → Healthy Report Bypass
      └─ No  → LLM Report Generator
  → Parse JSON 
  → Build HTML 
  → Idempotency Check
  → Not Sent Yet?
      ├─ Yes → Send Email → Mark as Sent
      └─ No  → Already Sent
  → Optional Slack branch
```

---

## Features

### 1) Multi-site SLA tracking
- Define host groups per site
- Track site SLA target vs actual
- Highlight target misses in report

### 2) Adaptive alerting
- Global thresholds + host-specific overrides
- Business-hours/weekend multipliers
- Maintenance window awareness

### 3) Predictive engine
- Disk growth forecasting
- Memory leak trend detection
- Bandwidth saturation risk estimation
- Pattern-based action recommendations

### 4) LLM optimization
- Skip LLM when everything is healthy (cost-saving mode)
- Robust JSON parse and fallback report if LLM output fails

### 5) Email safety
- Daily idempotency key prevents duplicate sends
- Incident history supports “incidents in last 30 days”

---

## Requirements

- **n8n** (self-hosted or cloud)
- **Prometheus-compatible query endpoint** (Prometheus / VictoriaMetrics)
- `node_exporter`-style host metrics
- n8n credentials:
  - Email (SMTP / Send Email node)
  - Optional OpenAI-compatible credential for LLM summary
  - Optional Slack webhook

---

## Quick start

### 1) Import workflow

- In n8n UI: import the workflow JSON file.
- Recommended filename: `n8n-prometheus-daily-report.json`

### 2) Configure credentials in n8n

Set credentials for:
- **Send Email** node (SMTP)
- **LLM Report Generator** (optional)
- **Slack Alert** (optional)

### 3) Update configuration node (`Configuration`)

Edit these values first:

- `base_url`
- `allowed_instances`
- `sites`
- `site_sla_targets`
- `thresholds` / `overrides`
- `network_capacity`
- `email_recipient`
- `timezone`
- optional: `slack_webhook`, `llm_model`, `llm_skip_on_healthy`

### 4) Test run

- Execute workflow manually
- Verify:
  - Query nodes return data
  - `Combine Metrics v6.0` output looks sane
  - Email HTML renders correctly

### 5) Production scheduling

Replace `Manual Trigger` with a **Schedule Trigger** (for example daily at 11:00 Europe/Copenhagen), then activate workflow.

---

## Configuration reference (example)

```js
const cfg = {
  base_url: "http://<prometheus-or-vm>:8428",
  allowed_instances: "host1,host2,host3",
  node_job_regex: "integrations/node_exporter",

  thresholds: {
    cpu_warn: 85, cpu_crit: 95,
    mem_warn: 80, mem_crit: 90,
    fs_warn: 85, fs_crit: 95,
    tcp_warn: 1000, tcp_crit: 50000,
    disk_latency_warn: 50, disk_latency_crit: 100,
    net_drops_warn: 1000, net_drops_crit: 10000,
    load_warn: 2.0, load_crit: 4.0
  },

  overrides: {
    "truenas": { mem_warn: 98, mem_crit: 99 },
    "aarhus-pi": { cpu_warn: 90 }
  },

  sites: {
    copenhagen: ["pve", "truenas", "proxmox-backup-server", "docker-proxmox", "vm-host-docker", "cachyos-valby"],
    helsinki: ["ubuntu-4gb-hel1-2"],
    remote: ["aarhus-pi"]
  },

  site_sla_targets: {
    copenhagen: 99.9,
    helsinki: 99.5,
    remote: 95.0
  },

  trends: {
    lookback_days: 7,
    detect_anomalies: true,
    predict_disk_full: true,
    predict_memory_leak: true,
    predict_network_saturation: true,
    disk_full_warn_days: 30,
    disk_full_crit_days: 14,
    mem_leak_warn_days: 45,
    mem_leak_crit_days: 21
  },

  email_recipient: "you@example.com",
  timezone: "Europe/Copenhagen",
  llm_skip_on_healthy: true
}
```

---

## Metrics covered

The workflow can pull and correlate metrics across:

- Host status / uptime / reboot count
- CPU usage + 7d trend
- Memory usage + 7d trend + swap
- Filesystem usage / capacity / growth
- Disk busy / read latency / write latency
- Network rx/tx, drops/errors, interface status
- TCP established / retransmissions
- Inventory-style context (e.g., uname, cpu_count, mem_total)

---

## Alerting & status model

- **CRITICAL:** One or more hosts in critical state
- **WARNING:** No critical hosts, but warnings present
- **HEALTHY:** No critical/warn issues

Report includes:
- Executive summary
- Site SLA table
- Week-over-week trends
- Priority actions (host + issue + command)
- Infrastructure metrics table

---

## Idempotency behavior

This workflow stores a daily key in workflow static data and skips duplicate sends for the same date.  
This prevents duplicate reports if the workflow is re-triggered later the same day.

---

## Troubleshooting

### No data in report
- Validate `base_url`
- Test Prometheus query directly:
  - `up{job=~"...",instance=~"..."}`
- Confirm exporter labels match your selectors

### Hosts missing
- Update `allowed_instances`
- Check site mapping (`sites`) includes hostnames exactly as seen in metrics

### Too many false alerts
- Tune `thresholds`, `overrides`, and `threshold_multipliers`
- Add/adjust maintenance windows

### Duplicate emails
- Confirm Idempotency node is connected before `Send Email`
- Ensure workflow static data persists in your n8n deployment

### LLM output parse failures
- `Parse JSON` fallback should still produce a safe summary
- Optionally tighten prompt and temperature settings

---

## Security notes

- Do not commit real credentials, API keys, or personal recipient addresses.
- Use n8n credential store and environment variables where possible.
- Restrict outbound SMTP/API access to trusted destinations only.

---

## Repository structure (suggested)

```text
.
├─ workflow/
│  └─ n8n-prometheus-daily-report.json
├─ docs/
│  ├─ SAMPLE_REPORT.md
│  ├─ METRIC_DICTIONARY.md
│  └─ TROUBLESHOOTING.md
└─ README.md
```

---

## Credits

Built with:
- n8n
- Prometheus-compatible metrics backend
- Optional LLM summarization for executive reporting
