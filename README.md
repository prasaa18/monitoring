# Monitoring Lab — Architecture & Quick Reference

Neat, single-page reference for the 4-instance monitoring lab: Prometheus, Grafana, Loki and the Web Server (Titan + Alloy + Node Exporter).

## Architecture (4 AWS instances)
- Prometheus (A) — 9090 (web UI & scrape API)
- Grafana (B) — 3000 (UI, dashboards, alerts)
- Loki (C) — 3100 (log ingestion/query)
- Web Server (D) — Titan Flask app (5000), Node Exporter (9100), Alloy (default 12345), load generators

## Quick diagram
Prometheus(A:9090)
  ├─ scrapes Node Exporter (D:9100)
  └─ scrapes App metrics (D:5000)
Grafana(B:3000) → reads Prometheus(A:9090) & Loki(C:3100)
Loki(C:3100) ← Alloy on WebServer(D) pushes logs

## Security Group (SG) rules (exact)
Replace `YOUR_IP` with your public IP/CIDR.

- Prometheus-SG (A)
  - TCP 9090 from Grafana-SG and YOUR_IP
  - TCP 22 from YOUR_IP

- Grafana-SG (B)
  - TCP 3000 from YOUR_IP
  - TCP 22 from YOUR_IP

- Loki-SG (C)
  - TCP 3100 from 0.0.0.0/0
  - TCP 22 from YOUR_IP

- WebServer-SG (D)
  - TCP 9100 from Prometheus-SG and YOUR_IP
  - TCP 5000 from Prometheus-SG and YOUR_IP
  - TCP 12345 (Alloy) from Prometheus-SG and YOUR_IP
  - TCP 22 from YOUR_IP

## Key files in this repo
- `webnode_setup.sh` — installs node_exporter, Titan app, Alloy and load scripts
- `lokisetup.sh` — Loki installer + systemd service
- `load.sh`, `generate_multi_logs.sh` — load & log generators
- `PortNumbers.png` — port diagram reference

## How components connect
- Alloy on WebServer reads `/var/log/titan/*.log` and forwards to Loki (http://LOKI_IP:3100/loki/api/v1/push) and optionally remote-writes metrics to Prometheus.
- Prometheus scrapes web app metrics at `http://WEB_SERVER_IP:5000/metrics` and node_exporter at `http://WEB_SERVER_IP:9100/metrics`.
- Grafana uses Prometheus (A:9090) and Loki (C:3100) as datasources.

## Prometheus scrape example (prometheus.yml)
job_name: 'webserver'
static_configs:
  - targets: ['WEB_SERVER_IP:5000','WEB_SERVER_IP:9100']

## Run instructions (short)
Copy scripts to each instance and run with sudo:

```bash
# On Loki instance
chmod +x lokisetup.sh
sudo ./lokisetup.sh

# On Web server
chmod +x webnode_setup.sh
sudo ./webnode_setup.sh
```

## Verification quick checks
- Prometheus UI: http://PROMETHEUS_IP:9090
- Grafana UI: http://GRAFANA_IP:3000
- Loki ready: `curl http://LOKI_IP:3100/ready`
- App: http://WEB_SERVER_IP:5000/
- Node Exporter: http://WEB_SERVER_IP:9100/metrics

## Notes
- Replace placeholders (PROMETHEUS_IP, GRAFANA_IP, LOKI_IP, WEB_SERVER_IP, YOUR_IP).
- Ensure SG rules allow Prometheus to reach WebServer ports (9100, 5000) and Grafana to reach Prometheus (9090).
- Alloy default listen port in scripts: 12345 — open in SG if used.

## Interview cheat-sheet — quick questions
1. Which ports are used by Prometheus, Grafana, Loki, Node Exporter and the app?
2. How does Prometheus scrape the web app and node_exporter?
3. Why restrict node_exporter access to Prometheus?
4. What role does Alloy play in this pipeline?
5. How are logs pushed to Loki?
6. How to configure Grafana to show metrics+logs together?
7. Basic health checks for Loki and Prometheus?
8. How to send Grafana alerts to Slack?
9. Risks of exposing Loki to the public internet?
10. Quick troubleshooting steps if Prometheus can't scrape the app.

---

File created by automation. Update placeholders before use.
