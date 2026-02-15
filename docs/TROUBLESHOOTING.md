# n8n + Prometheus/VictoriaMetrics Workflow Troubleshooting Guide

Use this playbook to diagnose failed runs, missing data, false alerts, and delivery issues.

---

## 1) Fast triage matrix

| Symptom | Likely cause | First check | Fix |
|---|---|---|---|
| Report says all hosts down | Wrong `base_url`, no connectivity, wrong selector | Test `/api/v1/query` manually | Fix URL/selectors/network |
| Some hosts missing | `allowed_instances` mismatch with actual `instance` labels | Query raw `up` | Align hostnames/regex |
| Duplicate email reports | Idempotency key not persisted | Check `Idempotency Check` + workflow active state | Keep workflow active, static data path valid |
| Schedule runs wrong time | n8n timezone mismatch | Check instance timezone and workflow trigger | Set `timezone` / `GENERIC_TIMEZONE` |
| LLM summary broken JSON | Model output not strict JSON | Inspect `Parse JSON v4.0` input | Harden prompt/fallback parser |
| Many network warnings | Interface filtering too broad/narrow | Check `net_device_filter` and raw interface metrics | Update regex, ignore known noisy interfaces |
| Host marked critical but healthy | Threshold too strict for that host | Compare thresholds vs real baseline | Add host overrides |
| No email sent | Credential or provider failure | Check Send Email node execution details | Re-auth credentials / provider policy |

---

## 2) Verify data source connectivity

### 2.1 Test API endpoint health

```bash
curl -s "http://<prom-or-vm>:8428/prometheus/api/v1/query?query=up" | jq .
# or for Prometheus native:
curl -s "http://<prometheus>:9090/api/v1/query?query=up" | jq .
```

Expected: JSON with `"status":"success"` and a non-empty `data.result`.

### 2.2 Validate selector from workflow config

```bash
curl -G "http://<prom-or-vm>:8428/prometheus/api/v1/query"   --data-urlencode 'query=up{job=~"integrations/node_exporter"}'
```

If this returns data but workflow does not, the issue is usually `allowed_instances` regex.

---

## 3) Validate each critical signal quickly

### Host up/down
```bash
up{job=~"integrations/node_exporter",instance=~"^(host1|host2)(:.*)?$"}
```

### CPU %
```bash
100 - (avg by (instance) (rate(node_cpu_seconds_total{job=~"integrations/node_exporter",mode="idle"}[5m])) * 100)
```

### Memory %
```bash
(1 - (node_memory_MemAvailable_bytes{job=~"integrations/node_exporter"} / node_memory_MemTotal_bytes{job=~"integrations/node_exporter"})) * 100
```

### Disk used %
```bash
(1 - (node_filesystem_avail_bytes{fstype!~"tmpfs|overlay|squashfs"} / node_filesystem_size_bytes{fstype!~"tmpfs|overlay|squashfs"})) * 100
```

### TCP retransmissions (24h)
```bash
sum by (instance) (increase(node_netstat_Tcp_RetransSegs{job=~"integrations/node_exporter"}[24h]))
```

---

## 4) n8n execution troubleshooting

## 4.1 Import and activation
- Ensure the workflow JSON imported correctly.
- Ensure credentials are mapped after import.
- Ensure workflow is **active** in production mode.

## 4.2 Schedule trigger not firing
- Confirm trigger node is `Schedule Trigger` in active workflow.
- Confirm n8n instance timezone.
- Confirm no paused/disabled workflow versions.

## 4.3 Code node return-shape errors
If you see errors like:
- “Code doesn't return items properly”
- “A 'json' property isn't an object”

Then make sure each Code node returns:
```js
return [{ json: { ... } }];
```

## 4.4 Static data/idempotency gotchas
- Static data persistence requires active workflow execution from trigger/webhook context.
- Manual test runs can behave differently from active scheduled runs.
- High-frequency concurrent runs may reduce static-data reliability; keep daily cadence single-threaded where possible.

---

## 5) Idempotency debugging checklist

1. Open execution data at `Idempotency Check`.
2. Verify generated key format (e.g., `report:YYYY-MM-DD`).
3. Confirm key read/write happens in same static-data scope (`global` vs node).
4. Trigger workflow twice same day:
   - First run should send.
   - Second run should route to `Already Sent`.

---

## 6) LLM branch troubleshooting

### Symptom: JSON parse failure
- Cause: model response includes prose or markdown.
- Fixes:
  - Force strict schema in prompt.
  - Add fallback parser and default summary.
  - Keep `Should Skip LLM?` enabled for healthy runs.

### Symptom: slow/expensive runs
- Keep `llm_skip_on_healthy = true`.
- Reduce prompt size (trim host-level verbose payload).

---

## 7) Alert quality tuning

## 7.1 Reduce false positives
- Increase warn thresholds gradually (5–10% at a time).
- Add host overrides for known high-baseline systems.
- Exclude noisy interfaces/devices via regex.

## 7.2 Catch missed incidents earlier
- Lower warn thresholds for critical hosts only.
- Add severity weighting in `Alert Correlation`.
- Incorporate trend acceleration (not just absolute values).

---

## 8) Network instability playbook (TCP retrans spikes)

When `tcp_retrans` spikes:

1. Confirm bandwidth and drops:
   ```bash
   ethtool -S <iface> | egrep -i "drop|err|overrun|miss"
   ip -s link show <iface>
   ```
2. Verify duplex/speed mismatches:
   ```bash
   ethtool <iface>
   ```
3. Check host and switch logs for flaps/resets.
4. Correlate with backup windows / large transfers.
5. If only one host is affected, inspect NIC/driver/firmware on that host first.

---

## 9) Host DOWN playbook

1. DNS resolution:
   ```bash
   getent hosts <hostname>
   ```
2. Reachability:
   ```bash
   ping -c 5 <hostname>
   ```
3. Port-level check (SSH/node_exporter):
   ```bash
   nc -zv <hostname> 22
   nc -zv <hostname> 9100
   ```
4. If unreachable:
   - check power/console/KVM
   - check switch port/VLAN/cabling
   - check firewall/security groups

---

## 10) Gmail/SMTP delivery issues

- Reauthorize credentials in n8n.
- Check provider anti-spam/rate limits.
- Validate recipient/domain policies (SPF/DKIM/DMARC).
- Add plain-text fallback content if HTML-only mails are blocked.

---

## 11) Recommended observability for the workflow itself

Add a lightweight “meta-monitoring” job:

- workflow execution duration
- execution success/failure count
- per-node error count
- number of incidents generated per report
- skipped-send (idempotent) count

This makes the report pipeline itself monitorable in Grafana.

---

## 12) Safe change process (before editing production workflow)

1. Export current workflow JSON backup.
2. Clone workflow and test in staging with Manual Trigger.
3. Compare report output against previous day baseline.
4. Activate staged workflow.
5. Disable old workflow after successful next scheduled run.

---

## 13) References

- n8n export/import workflow JSON
- n8n Schedule Trigger behavior and timezone
- n8n Code node return-shape expectations
- n8n workflow static data behavior for persistence/idempotency
- Prometheus HTTP API `/api/v1/query` and `/api/v1/query_range`
- VictoriaMetrics Prometheus-compatible API paths