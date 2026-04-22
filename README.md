# Nagios: From Reactive Alerts to Predictive Intelligence
**Nagios World Conference 2026 — Live Demo**

This project is a proof of concept built for the Nagios World Conference 2026. The idea is simple: add machine learning to Nagios Core without touching Nagios itself. A Python sidecar runs alongside Nagios, analyzes metrics in real time, predicts failures before they happen, and feeds results back into the Nagios UI through passive checks.

No Nagios patches. No plugins compiled into the core. Just a sidecar that speaks the language Nagios already understands.

---

## What the demo shows

When you click **Start Degradation**, three metrics (CPU, Disk, Network) begin to climb at a controlled rate. The ML models pick up the trend within the first couple of minutes and display a countdown to threshold breach. Nagios does not fire until the metric actually crosses the critical line, which happens several minutes later. That gap between prediction and alert is the point of the whole demo.

| Element | What's happening |
|---|---|
| Cyan dashed line on the chart | Prophet fitting a trend on the last 20 minutes of data |
| "PREDICTED CRITICAL in Xmin" badge | Linear regression detecting a rising slope |
| Red dots scattered on the chart | Isolation Forest marking statistically unusual readings |
| Health Score gauge (0 to 100) | Weighted average: CPU 40%, Disk 30%, Network 30% |
| Nagios Web UI changing color | Passive check injection pushing results every 5 seconds |
| Prediction log at the bottom | First-detection log: when the model first called each breach |

---

## Architecture

The sidecar runs on the same VM as Nagios Core and communicates with the browser over WebSocket. Every 5 seconds it ticks, computes new metric values, runs the models, pushes a JSON payload to all connected browsers, and writes a passive check result to the Nagios external command pipe.

```
nagios-server VM
├── Nagios Core 4.4.14  (Apache2, port 80)
└── FastAPI ML Sidecar  (Uvicorn, port 8000)
    ├── In-memory metric store   720 points x 5s = 60 min of history
    ├── Prophet                  forecast line + confidence band on chart
    ├── Linear regression        breach_minutes countdown (always runs)
    ├── Isolation Forest         anomaly dot markers
    ├── Health Score             unified gauge
    ├── WebSocket                live push to browser every 5s
    └── Passive check writer     PROCESS_SERVICE_CHECK_RESULT
                                 to /usr/local/nagios/var/rw/nagios.cmd
```

Nagios services are configured with `active_checks_enabled 0` and `passive_checks_enabled 1`, so NRPE never runs during the demo. The sidecar is the only source of truth for service status.

---

## Prerequisites

### On the host machine

- macOS with **VMware Fusion** (or any hypervisor with host-only networking)
- Homebrew, Python 3, and Ansible installed
- The `ansible.posix` collection:

```bash
ansible-galaxy collection install ansible.posix
```

### Virtual machines

You need two Debian 12 (Bookworm) VMs on the same host-only network, both reachable via SSH as root from the host machine.

| VM | Role | RAM | Disk |
|---|---|---|---|
| nagios-server | Nagios Core + ML Sidecar | 2 GB | 20 GB |
| monitored-host | NRPE agent | 1 GB | 10 GB |

ARM64 and x86_64 are both supported. If you are on Apple Silicon, use the ARM64 Debian image to avoid emulation overhead.

### Inventory setup

Edit `ansible/inventory.ini` before running anything:

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

## Deploying the lab

The full provisioning takes about 15 to 20 minutes, mostly because CmdStan (required by Prophet) compiles from source on the VM.

```bash
git clone https://github.com/luiscontrerasdo/nagios-predictive-intelligence.git
cd nagios-predictive-intelligence

# Confirm both VMs respond before starting
ping -c1 <NAGIOS_SERVER_IP>
ping -c1 <MONITORED_HOST_IP>

bash ansible/setup.sh
```

The setup script installs everything on both VMs:
- **nagios-server**: Nagios Core 4.4.14, Nagios Plugins, NRPE client, Python 3.11 venv, Prophet + CmdStan, scikit-learn, FastAPI sidecar registered as a systemd service, and the passive check configuration applied to Nagios.
- **monitored-host**: Nagios Plugins and NRPE daemon.

After it finishes, verify with:

```bash
curl http://<NAGIOS_SERVER_IP>:8000/api/health
# Should return: {"status":"ok"}
```

Then open both tabs you will need for the demo:
- ML Dashboard: `http://<NAGIOS_SERVER_IP>:8000`
- Nagios Web UI: `http://<NAGIOS_SERVER_IP>/nagios`

---

## Email notifications

The sidecar can send a predictive alert by email the moment the ML model first detects an upcoming breach, before Nagios fires. Each metric sends one email per scenario run, triggered by the first detection only.

The email includes the metric name, current value, predicted breach time in minutes, critical threshold, and the exact time of detection.

### What you need

- An SMTP account (any provider works: Gmail, iPage, Outlook, etc.)
- The SMTP server hostname and port (typically 587 with STARTTLS)
- SMTP username and password
- A sender address and a recipient address

### Setup on the server

Edit the `.env` file on `nagios-server` with your SMTP credentials:

```bash
nano /opt/nagios_ml_sidecar/.env
```

```ini
SMTP_HOST=your.smtp.server.com
SMTP_PORT=587
SMTP_USER=sender@yourdomain.com
SMTP_PASS=your-smtp-password
EMAIL_FROM=sender@yourdomain.com
EMAIL_TO=recipient@yourdomain.com
```

Then restart the sidecar to apply:

```bash
systemctl restart nagios-ml-sidecar
```

If `EMAIL_TO` is left empty, the sidecar skips notifications silently and everything else keeps working normally.

> **Gmail users:** standard account passwords do not work with SMTP. You need to enable two-step verification and generate an App Password at `myaccount.google.com/apppasswords`. Use that as `SMTP_PASS`.

---

## Running the demo

Open the ML dashboard and the Nagios Web UI side by side so the audience can see both react in real time.

1. Click **Start Degradation** on the dashboard.
2. Watch the metrics climb every 5 seconds.
3. Around the 2-minute mark, the prediction badge appears. Nagios is still green.
4. Nagios services begin transitioning to WARNING, then CRITICAL as thresholds are crossed.
5. Click **Reset to Normal** to restore baseline and clear the prediction log.

### Timing reference

| Time | What happens |
|---|---|
| 0:00 | Degradation starts |
| ~1:00 | Metrics rising, Health Score turns yellow |
| ~2:00 | Prediction badge appears. Nagios still OK. |
| ~5:00 | Network crosses critical threshold. Nagios fires. |
| ~6:00 | CPU breach |
| ~9:00 | Disk breach |

### Updating code without a full reinstall

When you change any Python, JS, HTML, or CSS file, push the changes to the VM in about 45 seconds:

```bash
cd ansible
ansible-playbook -i inventory.ini update_sidecar.yml \
  --extra-vars "project_dir=$(cd .. && pwd)"
```

---

## API reference

The sidecar exposes a small REST API you can query directly during or after the demo.

| Method | Path | Description |
|---|---|---|
| GET | `/api/health` | Service health check |
| GET | `/api/metrics` | Current metric snapshot (same payload as WebSocket) |
| GET | `/api/predictions/history` | Full prediction log, up to 50 entries |
| POST | `/api/scenario/start` | Begin degradation scenario |
| POST | `/api/scenario/reset` | Return to normal baseline and clear log |
| WS | `/ws` | Live push every 5 seconds |

---

## How the ML models work

### Two models for one job

There are two models involved in breach detection and they serve different purposes.

**Prophet** produces the forecast chart line and confidence band. It fits a trend model on the last 20 minutes of 1-minute-resampled data and projects 30 minutes forward. It gives the audience something visual to follow.

**Linear regression** (`numpy.polyfit`) is what actually drives the countdown badge. It fits a line on the last 10 data points, extrapolates to the threshold, and returns the minutes remaining. It runs fast, it runs always, and it does not depend on Prophet completing successfully.

The reason for the split is reliability. Prophet uses a Stan backend that can time out on ARM64 VMs with short data windows. The linear regression is a few lines of NumPy and never fails. By keeping the breach prediction outside the try/except that guards Prophet, the countdown badge always shows up even when the chart line does not.

### Slope filter

```python
if slope < 0.5:   # percent per minute
    return None
```

At baseline, metric noise produces slopes well below 0.1 %/min. The degradation scenario produces slopes in the 7 to 11 %/min range. The 0.5 threshold filters out noise without slowing down detection of a real trend.

### Data resampling

Raw readings come in every 5 seconds (720 data points over 60 minutes). Before fitting either model, the data is resampled to 1-minute means and capped at the last 20 minutes. Without the 20-minute cap, the long flat baseline from before degradation starts overwhelms the rising trend and Prophet predicts a flat line even while metrics are actively climbing.

### Passive check thresholds

Each metric has a warning and a critical threshold. The sidecar evaluates the current value on every tick and writes the appropriate status to the Nagios command pipe.

| Metric | Warning | Critical |
|---|---|---|
| CPU | 70% | 80% |
| Disk | 75% | 85% |
| Network | 65% | 75% |

The command written to the pipe looks like this:

```
[timestamp] PROCESS_SERVICE_CHECK_RESULT;monitored-host;CPU Usage;2;CRITICAL - CPU at 83.4% | cpu=83.4%;70;80;0;100
```

---

## Troubleshooting

**Nagios services stay at OK even during degradation**

Check that `check_external_commands=1` is set in `/usr/local/nagios/etc/nagios.cfg` and that the service definitions in `monitored-host.cfg` have `active_checks_enabled 0` and `passive_checks_enabled 1`. Running `update_sidecar.yml` applies both. You can also check whether the command pipe exists:

```bash
ls -la /usr/local/nagios/var/rw/nagios.cmd
```

**Prediction badge never appears**

Check the sidecar logs for errors:

```bash
journalctl -u nagios-ml-sidecar -n 50
```

Also confirm the breach_minutes field in the API response:

```bash
curl -s http://<NAGIOS_SERVER_IP>:8000/api/metrics | python3 -m json.tool | grep breach_minutes
```

**Dashboard does not load**

```bash
systemctl restart nagios-ml-sidecar
systemctl status nagios-ml-sidecar
```

**Full reinstall**

```bash
ansible-playbook -i ansible/inventory.ini ansible/reset.yml
bash ansible/setup.sh
```

---

## Tech stack

| Component | Technology |
|---|---|
| ML Sidecar | Python 3.11, FastAPI, Uvicorn |
| Forecasting | Prophet (Meta) |
| Breach detection | NumPy polyfit (linear regression) |
| Anomaly detection | scikit-learn Isolation Forest |
| Real-time push | WebSocket (native FastAPI) |
| Charts | Chart.js |
| Nagios integration | External command pipe, passive checks |
| Provisioning | Ansible |

---

*Proof of Concept presented at Nagios World Conference 2026 by Luis Contreras.*
