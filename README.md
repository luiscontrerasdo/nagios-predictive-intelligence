# Nagios: From Reactive Alerts to Predictive Intelligence
**Nagios World Conference 2026 — Live Demo**

A 30–45 minute conference session that transforms Nagios Core into a predictive monitoring platform using machine learning — with no changes to Nagios Core itself.

---

## What This Demo Shows

| What you see | How it works |
|---|---|
| Dashed cyan forecast line on each chart | Prophet (Meta) fits a trend model on the last 20 min of data |
| "PREDICTED CRITICAL in Xmin" badge | Linear regression on last 10 data points detects slope |
| Red dots on the chart | Isolation Forest flags statistically anomalous readings |
| Health Score gauge (0–100) | Weighted average: CPU×0.40 + Disk×0.30 + Network×0.30 |
| Blinking ⚠ badge in services panel | Linear regression breach prediction per Nagios service |
| Predictive timeline strip | Canvas: 45 min history + NOW marker + 30 min forecast zone |

The ML prediction appears **~2 minutes** after clicking `[ START DEGRADATION ]`, before Nagios fires a CRITICAL alert.

---

## Architecture

```
MacBook Pro (VMware Fusion)
├── nagios-server  172.16.242.124  (Debian 12 ARM64)
│   ├── Nagios Core 4.4.14
│   ├── Apache2 → http://nagios-server/nagios
│   └── FastAPI ML Sidecar :8000
│       ├── In-memory store  (deque, 720 pts × 5s = 60 min)
│       ├── Prophet          → forecast chart line + confidence band
│       ├── Linear Regression → breach_minutes (fast, Prophet-independent)
│       ├── Isolation Forest → anomaly dot markers
│       ├── Health Score     → unified 0–100 gauge
│       ├── WebSocket push   → browser dashboard every 5s
│       └── Passive check injection → Nagios external command pipe every 5s
│           (PROCESS_SERVICE_CHECK_RESULT → /usr/local/nagios/var/rw/nagios.cmd)
└── monitored-host  172.16.242.125  (Debian 12 ARM64)
    └── NRPE 4.1.0  (passive-only services — sidecar is sole status source)
```

The sidecar writes a `PROCESS_SERVICE_CHECK_RESULT` command to Nagios every 5 seconds so the **Nagios Web UI always reflects the same simulated values** as the ML dashboard. Nagios services are configured with `active_checks_enabled 0` / `passive_checks_enabled 1` — NRPE is never called during the demo.

---

## Prerequisites

- **MacBook** with VMware Fusion installed
- **Two Debian 12 VMs** already created and running (see VM specs below)
- **SSH access** as `root` to both VMs
- On your Mac: `brew`, `python3`, `pip3`

### VM Specs (minimum)

| VM | RAM | Disk | IP |
|---|---|---|---|
| `nagios-server` | 2 GB | 20 GB | `172.16.242.124` |
| `monitored-host` | 1 GB | 10 GB | `172.16.242.125` |

Both VMs on the same VMware host-only network. `root` password: `laboratorio` on both.

---

## Quick Start — Full Install (~15–20 min)

```bash
# 1. Verify both VMs are reachable
ping -c1 172.16.242.124
ping -c1 172.16.242.125

# 2. Clone or download this repo to your Mac
cd /path/to/Nagios_2026_DEMO

# 3. Run the provisioner (installs everything on both VMs)
bash ansible/setup.sh
```

`setup.sh` will:
- Install `sshpass` and `ansible` if missing (via Homebrew / pip)
- Install `ansible.posix` collection if missing
- Run `ansible/playbook.yml` which provisions:
  - **nagios-server**: Nagios Core 4.4.14, Nagios Plugins 2.4.6, NRPE 4.1.0 client, Python 3.11 venv, Prophet, CmdStan, scikit-learn, FastAPI sidecar as a systemd service
  - **monitored-host**: Nagios Plugins 2.4.6, NRPE 4.1.0 daemon

### Verify Install

```bash
# Health check
curl http://172.16.242.124:8000/api/health
# Expected: {"status":"ok"}

# Open dashboard in browser
open http://172.16.242.124:8000

# Open Nagios Web UI  (login: nagiosadmin / nagios2026)
open http://172.16.242.124/nagios
```

---

## Running the Demo

### Option A — Dashboard button (recommended for live presentation)
1. Open `http://172.16.242.124:8000` in your browser
2. Click **`[ START DEGRADATION ]`**
3. Watch metrics rise every 5 seconds
4. In ~2 minutes: blinking ⚠ badges appear in the services panel
5. Click **`[ RESET TO NORMAL ]`** between runs

### Option B — Terminal (useful for monitoring alongside the browser)
```bash
# Start degradation + live terminal status output
python3 ansible/demo_scenario.py start

# Reset between runs
python3 ansible/demo_scenario.py reset
```

Both options call the same API endpoint (`POST /api/scenario/start`).

### Demo Timing

| Time | Event |
|---|---|
| 0:00 | Click START DEGRADATION |
| ~1:00 | Charts start rising, health score moves to YELLOW |
| ~2:00 | "PREDICTED CRITICAL in Xmin" appears — Nagios has NOT fired yet |
| ~5:00 | Network breach (~75%) — first Nagios CRITICAL |
| ~6:00 | CPU breach (~80%) |
| ~9:00 | Disk breach (~85%) |

---

## Updating Code (without reinstalling)

When you change Python, JS, CSS, or HTML files, use the fast update playbook (~45 seconds). This also re-applies the Nagios passive check configuration:

```bash
cd ansible
ansible-playbook -i inventory.ini update_sidecar.yml \
  --extra-vars "project_dir=$(cd .. && pwd)"
```

Or manually via rsync:
```bash
rsync -av --exclude='.venv' --exclude='__pycache__' --exclude='*.pyc' \
  nagios_ml_sidecar/ \
  root@172.16.242.124:/opt/nagios_ml_sidecar/ \
  --rsh="sshpass -p laboratorio ssh -o StrictHostKeyChecking=no"

sshpass -p laboratorio ssh root@172.16.242.124 \
  "systemctl restart nagios-ml-sidecar"
```

---

## Reset / Reinstall from Scratch

```bash
# Remove everything from both VMs
ansible-playbook -i ansible/inventory.ini ansible/reset.yml

# Reinstall
bash ansible/setup.sh
```

---

## Project Structure

```
Nagios_2026_DEMO/
├── README.md
├── ansible/
│   ├── setup.sh              # Mac → runs full provisioning playbook
│   ├── playbook.yml          # Full install: Nagios Core + NRPE + ML sidecar
│   ├── update_sidecar.yml    # Fast code update: rsync + restart (~30s)
│   ├── reset.yml             # Full uninstall of both VMs
│   ├── demo_scenario.py      # CLI controller: start / reset degradation
│   ├── inventory.ini         # VM IP addresses
│   └── ansible.cfg           # SSH settings (sshpass, no host key check)
├── nagios_ml_sidecar/        # FastAPI ML sidecar (deployed to /opt/nagios_ml_sidecar/)
│   ├── main.py               # App entrypoint, routes, WebSocket, broadcast loop
│   ├── models/
│   │   ├── forecaster.py     # Prophet (chart) + linear regression (breach_minutes)
│   │   ├── anomaly.py        # Isolation Forest anomaly dot markers
│   │   └── health_score.py   # Weighted 0–100 health gauge
│   ├── data/
│   │   └── simulator.py      # Baseline seed data + degradation value generator
│   ├── templates/
│   │   └── dashboard.html    # Dashboard UI (Jinja2 + Chart.js + WebSocket)
│   ├── static/
│   │   ├── dashboard.js      # Live chart updates, timeline canvas, services panel
│   │   └── style.css         # Dark terminal theme
│   ├── tests/                # pytest suite (~30 tests, runs in ~3s)
│   └── requirements.txt
├── diagrams/
│   ├── architecture.png      # System diagram (VM layout)
│   └── ml_pipeline.png       # ML data flow diagram
├── scripts/
│   └── generate_presentation.py  # Generates the 19-slide PPTX from code
├── presentation/
│   └── Nagios_Predictive_2026.pptx
└── docs/
    └── superpowers/specs/
        └── 2026-04-21-nagios-predictive-demo-design.html
```

---

## ML Models — Key Design Decisions

### Why two models for breach detection?

| Model | Role | Why |
|---|---|---|
| **Prophet** | Forecast chart visualization (30-min line + confidence band) | Needs historical context; handles seasonality for the visual story |
| **Linear regression** (`numpy.polyfit`) | `breach_minutes` countdown | Detects rising slope in 1–2 min; runs independently of Prophet |

Prophet's Stan backend can time out on short data windows on ARM64 VMs. The linear regression runs **outside** the `try/except` that wraps Prophet so `breach_minutes` always appears even when Prophet fails.

### Linear regression slope filter

```python
if slope < 0.5:   # %/min — filters noise, not real trends
    return None
```

Normal operation noise produces slopes below 0.1 %/min. The demo's degradation rate (0.12–0.18 %/s = 7–11 %/min) is detected within the first 1–2 minutes.

### Data resampling strategy

Raw data is at 5-second resolution (720 points). Before ML fitting:
- Resampled to 1-minute means (`df.resample("1min").mean()`)
- Limited to **last 20 minutes** (`tail(20)`) for Prophet input

Without the 20-minute cap, 60 minutes of flat baseline drowns out the degradation trend — Prophet predicts a flat line even while metrics actively climb.

---

## Credentials

| Resource | URL / Value |
|---|---|
| ML Dashboard | `http://172.16.242.124:8000` |
| Nagios Web UI | `http://172.16.242.124/nagios` |
| Nagios login | `nagiosadmin` / `nagios2026` |
| SSH (both VMs) | `root` / `laboratorio` |

---

## Troubleshooting

**Nagios services always show OK even during degradation**

Passive check injection requires two conditions on the server:
1. `check_external_commands=1` in `/usr/local/nagios/etc/nagios.cfg`
2. Services configured with `active_checks_enabled 0` / `passive_checks_enabled 1`

The `update_sidecar.yml` playbook applies both. If you skipped it, run:
```bash
cd ansible
ansible-playbook -i inventory.ini update_sidecar.yml \
  --extra-vars "project_dir=$(cd .. && pwd)"
```

**Prediction log shows "Invalid Date" entries**

This is a display bug fixed in the current version of `dashboard.js`. The prediction log timestamps are already formatted as `HH:MM:SS` strings by the backend — they should not be passed through `new Date()`. Redeploy with `update_sidecar.yml` to pick up the fix.

**ML Prediction column shows `—` after starting degradation**
```bash
# Check sidecar status
sshpass -p laboratorio ssh root@172.16.242.124 systemctl status nagios-ml-sidecar

# Check logs for Prophet errors
sshpass -p laboratorio ssh root@172.16.242.124 journalctl -u nagios-ml-sidecar -n 50

# Verify breach_minutes in API response
curl -s http://172.16.242.124:8000/api/metrics | python3 -m json.tool | grep breach_minutes
```

**Dashboard does not load / connection refused**
```bash
sshpass -p laboratorio ssh root@172.16.242.124 \
  "systemctl restart nagios-ml-sidecar && systemctl status nagios-ml-sidecar --no-pager"
```

**Full reinstall**
```bash
ansible-playbook -i ansible/inventory.ini ansible/reset.yml
bash ansible/setup.sh
```

**Regenerate the PowerPoint (19 slides)**
```bash
cd /path/to/Nagios_2026_DEMO
python3 scripts/generate_presentation.py
# Output: presentation/Nagios_Predictive_2026.pptx  (expected 19 slides)
```

---

## Running Tests

```bash
cd nagios_ml_sidecar
source .venv/bin/activate

# Fast suite — excludes slow Prophet fitting tests (~3s)
pytest tests/ -v --ignore=tests/test_forecaster.py

# Full suite including Prophet (~30s)
pytest tests/ -v
```
