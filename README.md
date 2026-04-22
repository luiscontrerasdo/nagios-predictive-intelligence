# Nagios: From Reactive Alerts to Predictive Intelligence
**Nagios World Conference 2026 — Live Demo**

A proof of concept that extends Nagios Core with a machine learning sidecar for predictive monitoring — no changes to Nagios Core itself.

---

## What This Demo Shows

| What you see | How it works |
|---|---|
| Dashed cyan forecast line on each chart | Prophet (Meta) fits a trend model on the last 20 min of data |
| "PREDICTED CRITICAL in Xmin" badge | Linear regression detects rising slope before threshold breach |
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

The sidecar writes a `PROCESS_SERVICE_CHECK_RESULT` command to the Nagios external command pipe every 5 seconds so the **Nagios Web UI always reflects the same simulated values** as the ML dashboard. Services run with `active_checks_enabled 0` / `passive_checks_enabled 1` — the sidecar is the sole status source during the demo.

---

## Prerequisites

### Host machine
- macOS with **VMware Fusion** (or any hypervisor that supports host-only networking)
- `brew`, `python3`, `ansible` installed on the host
- `ansible.posix` collection: `ansible-galaxy collection install ansible.posix`

### Virtual Machines (2)

| VM | Role | Min RAM | Min Disk |
|---|---|---|---|
| `nagios-server` | Nagios Core + ML Sidecar | 2 GB | 20 GB |
| `monitored-host` | NRPE agent (simulated target) | 1 GB | 10 GB |

- OS: **Debian 12 (Bookworm)** — ARM64 or x86_64
- Both VMs on the same host-only network, reachable from the host
- SSH access as `root` to both VMs

### Inventory configuration

Edit `ansible/inventory.ini` with your VM IPs:

```ini
[nagios_server]
nagios-server ansible_host=<NAGIOS_SERVER_IP>

[monitored_host]
monitored-host ansible_host=<MONITORED_HOST_IP>

[all:vars]
ansible_user=root
ansible_ssh_pass=<YOUR_ROOT_PASSWORD>
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

---

## Deploy the Lab (~15–20 min)

```bash
# 1. Clone the repository
git clone https://github.com/luiscontrerasdo/nagios-predictive-intelligence.git
cd nagios-predictive-intelligence

# 2. Verify both VMs are reachable
ping -c1 <NAGIOS_SERVER_IP>
ping -c1 <MONITORED_HOST_IP>

# 3. Run the provisioner
bash ansible/setup.sh
```

The provisioner installs on `nagios-server`:
- Nagios Core 4.4.14 + Nagios Plugins 2.4.6
- NRPE 4.1.0 (client)
- Python 3.11 virtual environment
- Prophet, CmdStan, scikit-learn, FastAPI
- ML Sidecar as a systemd service (`nagios-ml-sidecar`)
- Passive check configuration (external command pipe enabled)

And on `monitored-host`:
- Nagios Plugins 2.4.6
- NRPE 4.1.0 daemon

### Verify the install

```bash
# Health check
curl http://<NAGIOS_SERVER_IP>:8000/api/health
# Expected: {"status":"ok"}

# Open the ML dashboard
open http://<NAGIOS_SERVER_IP>:8000

# Open the Nagios Web UI
open http://<NAGIOS_SERVER_IP>/nagios
```

---

## Running the Demo

1. Open both tabs side by side: **ML dashboard** and **Nagios Web UI**
2. Click **`[ START DEGRADATION ]`** on the dashboard
3. Watch metrics rise every 5 seconds
4. In ~2 minutes: blinking ⚠ badges appear — Nagios has not fired yet
5. Nagios services transition WARNING → CRITICAL as thresholds are crossed
6. Click **`[ RESET TO NORMAL ]`** to restore baseline

### Demo Timing

| Time | Event |
|---|---|
| 0:00 | Click START DEGRADATION |
| ~1:00 | Charts rising, health score moves to YELLOW |
| ~2:00 | "PREDICTED CRITICAL in Xmin" — Nagios still OK |
| ~5:00 | Network breach — first Nagios CRITICAL |
| ~6:00 | CPU breach |
| ~9:00 | Disk breach |

### Update code without reinstalling

```bash
cd ansible
ansible-playbook -i inventory.ini update_sidecar.yml \
  --extra-vars "project_dir=$(cd .. && pwd)"
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
