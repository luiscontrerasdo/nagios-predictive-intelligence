# Nagios: From Reactive Alerts to Predictive Intelligence
**Nagios World Conference 2026 — Live Demo**

A conference session that transforms Nagios Core into a predictive monitoring platform using machine learning — with no changes to Nagios Core itself.

---

## What This Demo Shows

| What you see | How it works |
|---|---|
| Dashed cyan forecast line on each chart | Prophet (Meta) fits a trend model on the last 20 min of data |
| "PREDICTED CRITICAL in Xmin" badge | Linear regression on last 10 data points detects slope |
| Red dots on the chart | Isolation Forest flags statistically anomalous readings |
| Health Score gauge (0–100) | Weighted average: CPU×0.40 + Disk×0.30 + Network×0.30 |
| Nagios UI reflecting live status | Passive check injection via external command pipe every 5s |
| Prediction log | Audit trail of first-detection events per metric |

The ML prediction appears **~2 minutes** after clicking `[ START DEGRADATION ]`, before Nagios fires a CRITICAL alert.

---

## Architecture

```
┌──────────────────────────────────────────────────────┐
│  nagios-server                                       │
│  ├── Nagios Core 4.4.14                              │
│  ├── Apache2  →  /nagios  (Web UI)                   │
│  └── FastAPI ML Sidecar  :8000                       │
│      ├── In-memory store  (deque, 720 pts × 5s)      │
│      ├── Prophet          → forecast chart + band    │
│      ├── Linear Regression → breach_minutes          │
│      ├── Isolation Forest → anomaly markers          │
│      ├── Health Score     → unified 0–100 gauge      │
│      ├── WebSocket push   → browser every 5s         │
│      └── Passive check injection every 5s            │
│          PROCESS_SERVICE_CHECK_RESULT                │
│          → /usr/local/nagios/var/rw/nagios.cmd       │
└──────────────────────────────────────────────────────┘
```

The sidecar writes a `PROCESS_SERVICE_CHECK_RESULT` command to the Nagios external command pipe every 5 seconds so the **Nagios Web UI always reflects the same simulated values** as the ML dashboard. Nagios services run with `active_checks_enabled 0` / `passive_checks_enabled 1` — the sidecar is the sole status source during the demo.

---

## Project Structure

```
Nagios_2026_DEMO/
├── ansible/
│   ├── playbook.yml          # Full provisioning: Nagios Core + NRPE + ML sidecar
│   ├── update_sidecar.yml    # Fast code update + Nagios config sync (~45s)
│   └── reset.yml             # Full uninstall
├── nagios_ml_sidecar/        # FastAPI ML sidecar
│   ├── main.py               # App entrypoint, routes, WebSocket, broadcast loop
│   ├── models/
│   │   ├── forecaster.py     # Prophet (chart) + linear regression (breach_minutes)
│   │   ├── anomaly.py        # Isolation Forest anomaly markers
│   │   └── health_score.py   # Weighted 0–100 health gauge
│   ├── data/
│   │   └── simulator.py      # Baseline seed data + degradation value generator
│   ├── templates/
│   │   └── dashboard.html    # Dashboard UI (Jinja2 + Chart.js + WebSocket)
│   ├── static/
│   │   ├── dashboard.js      # Live chart updates, timeline canvas, services panel
│   │   └── style.css         # Dark terminal theme
│   └── requirements.txt
├── diagrams/
│   ├── architecture.png      # System diagram
│   └── ml_pipeline.png       # ML data flow diagram
├── scripts/
│   └── generate_presentation.py  # Generates the 19-slide PPTX
└── presentation/
    └── Nagios_Predictive_2026.pptx
```

---

## ML Models — Key Design Decisions

### Why two models for breach detection?

| Model | Role | Why |
|---|---|---|
| **Prophet** | Forecast chart visualization (30-min line + confidence band) | Handles seasonality; provides the visual story |
| **Linear regression** (`numpy.polyfit`) | `breach_minutes` countdown | Detects rising slope in 1–2 min; runs independently of Prophet |

Prophet's Stan backend can time out on short data windows on ARM64. The linear regression runs **outside** the `try/except` that wraps Prophet so `breach_minutes` always appears even when Prophet fails.

### Linear regression slope filter

```python
if slope < 0.5:   # %/min — filters noise, not real trends
    return None
```

Normal operation noise produces slopes below 0.1 %/min. The demo's degradation rate (7–11 %/min) is detected within the first 1–2 minutes.

### Data resampling strategy

Raw data is at 5-second resolution (720 points). Before ML fitting:
- Resampled to 1-minute means (`df.resample("1min").mean()`)
- Limited to **last 20 minutes** (`tail(20)`) for Prophet input

Without the 20-minute cap, 60 minutes of flat baseline drowns out the degradation trend.

### Passive check injection

Every 5 seconds the sidecar evaluates each metric against warning/critical thresholds and writes the result to the Nagios external command pipe:

```
[timestamp] PROCESS_SERVICE_CHECK_RESULT;monitored-host;CPU Usage;2;CRITICAL - CPU at 83.4% | cpu=83.4%;70;80;0;100
```

This keeps the Nagios UI synchronized with the ML dashboard without modifying Nagios Core.

---

## Demo Timing

| Time | Event |
|---|---|
| 0:00 | Click START DEGRADATION |
| ~1:00 | Charts start rising, health score moves to YELLOW |
| ~2:00 | "PREDICTED CRITICAL in Xmin" appears — Nagios has NOT fired yet |
| ~5:00 | Network breach — first Nagios CRITICAL |
| ~6:00 | CPU breach |
| ~9:00 | Disk breach |

---

## Tech Stack

| Component | Technology |
|---|---|
| ML Sidecar | Python 3.11, FastAPI, Uvicorn |
| Forecasting | Prophet (Meta) |
| Breach detection | NumPy `polyfit` (linear regression) |
| Anomaly detection | scikit-learn Isolation Forest |
| Real-time push | WebSocket (native FastAPI) |
| Charts | Chart.js |
| Nagios integration | External command pipe (passive checks) |
| Provisioning | Ansible |

---

*Proof of Concept — Nagios World Conference 2026*
