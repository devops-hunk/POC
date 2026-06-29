# Software Metric Dashboard — Local Setup & Developer Guide

> **AI Chatbot & CI/CD Quality Monitor for Stryker Robotics**
> Repository: https://gitlab.com/strykercorp/robotics/darlin/SGTC_Tools/swapnil_chatbox
> Branch: `Ollmamodel_chatbox`

---

## 1. Introduction

The **Software Metric Dashboard** is an internal web application built for Stryker's Robotics division. It does two things:

1. **CI/CD Quality Monitoring** — automatically pulls build and test metrics from GitLab pipelines and displays them as visual trend charts across multiple surgical robotic software platforms.
2. **AI Chat Assistant** — lets engineers ask plain-English questions about those metrics (e.g. *"What is the coverage trend for THA6.0?"* or *"Which app is regressing on unit tests?"*) and get intelligent answers backed by live database data.

### Applications Tracked

| App ID | Full Name |
|---|---|
| THA6.0 | Total Hip Arthroplasty 6.0 |
| TKA4.0 | Total Knee Arthroplasty 4.0 |
| TKA3.0.1 | Total Knee Arthroplasty 3.0.1 (legacy) |
| TKA3.0OUS | Total Knee Arthroplasty 3.0 Outside US |
| Shoulder1.5 | Shoulder Robotic Platform 1.5 |
| Shoulder2.0 | Shoulder Robotic Platform 2.0 |
| Clubber | Internal Tooling Platform |

### Metrics Tracked Per Application

- Function Coverage % and Decision Coverage % (from GCOV/LCOV reports)
- Unit Test pass/fail counts (from JUnit XML reports)
- Squish UI Test pass rates and individual test results
- ASAN memory error counts
- TODO/FIXME code debt counts
- Peer review stats and open Jira defects

### Tech Stack

| Layer | Technology |
|---|---|
| Backend | Python 3 + Flask + Gunicorn |
| Database | MySQL 8 (running in Docker) |
| AI / LLM | Ollama (local) with `qwen2.5` model |
| Frontend | HTML + Vanilla JavaScript |
| Packaging | Python venv + pip |

---

## 2. Prerequisites

Before setting up the project, make sure the following are installed on your machine.

### 2.1 Required Tools

| Tool | Version | Check Command |
|---|---|---|
| Ubuntu | 22.04 LTS recommended | `lsb_release -a` |
| Python | 3.10 or higher | `python3 --version` |
| pip | Latest | `pip3 --version` |
| Git | Any recent version | `git --version` |
| Docker Engine | v24+ with Compose plugin v2 | `docker compose version` |
| Ollama | Latest | `ollama --version` |
| curl | Pre-installed on Ubuntu | `curl --version` |

Minimum hardware: **8 GB RAM**, **10 GB free disk space**.
Recommended: **16 GB RAM** (needed for larger Ollama models).

---

### 2.2 Install Docker Engine (Ubuntu)

If Docker is not already installed, run the following to install it from Docker's official apt repository:

```bash
# Add Docker's GPG key and repository
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo $VERSION_CODENAME) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine and Compose plugin
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Allow running Docker without sudo
sudo usermod -aG docker $USER

# Log out and back in, then verify
docker info
docker compose version
```

> **Important:** After running `sudo usermod -aG docker $USER`, you must **log out and log back in** (or reboot) for the group change to take effect. Simply opening a new terminal is not enough.

---

### 2.3 Install Python venv Support

```bash
sudo apt install -y python3-venv python3-pip
```

---

### 2.4 Install Ollama

Ollama runs the AI language model locally. Install it and pull a model suitable for your machine:

```bash
# Install Ollama (auto-starts as a systemd service on Ubuntu)
curl -fsSL https://ollama.com/install.sh | sh

# Verify Ollama is running
systemctl status ollama

# Pull the chat model — choose based on your available RAM
ollama pull qwen2.5:7b     # ~4.7 GB — recommended for 8–16 GB RAM
ollama pull qwen2.5:14b    # ~9 GB   — better quality, needs 16+ GB RAM

# Test that Ollama is serving
curl http://localhost:11434/api/tags
```

**Model size guide:**

| Model | Size | RAM Needed | Quality |
|---|---|---|---|
| qwen2.5:3b | 2 GB | 8 GB | Basic |
| qwen2.5:7b | 4.7 GB | 8 GB | Good — recommended |
| qwen2.5:14b | 9 GB | 16 GB | Better |
| qwen2.5:32b | 20 GB | 32 GB | Excellent |

---

## 3. Clone the Repository

The project is hosted on Stryker's internal GitLab. You need a GitLab account with access to the `SGTC_Tools` group.

### Clone via HTTPS

```bash
git clone https://gitlab.com/strykercorp/robotics/darlin/SGTC_Tools/swapnil_chatbox.git

cd swapnil_chatbox

# Switch to the working branch
git checkout Ollmamodel_chatbox

# Confirm you are on the right branch
git branch --show-current
# Expected output: Ollmamodel_chatbox
```

### Clone via SSH (recommended for frequent contributors)

```bash
# First add your SSH public key to GitLab: Settings → SSH Keys
git clone git@gitlab.com:strykercorp/robotics/darlin/SGTC_Tools/swapnil_chatbox.git
cd swapnil_chatbox
git checkout Ollmamodel_chatbox
```

### Repository Structure Overview

```
swapnil_chatbox/
  start.sh                      # One-command project launcher
  .env.example                  # Template for all config values
  requirements.txt              # Full dependencies (incl. AI/RAG/torch)
  requirements.dashboard.txt    # Lightweight dependencies (dashboard only)
  seed_missing_apps.py          # Script to generate dummy data for empty apps
  chatbox/                      # Core chatbot logic (Flask routes, chat service)
  files/dashboard/
    py_dashboard/web/           # Flask app factory, templates, static JS
    db/                         # MySQL schema SQL, db manager, data reader
    config/                     # Per-app JSON configs (GitLab URLs, tokens)
    data_extractor/             # Scripts to pull pipeline artifacts from GitLab
  software_metric_backups/      # MySQL dump files with sample metric data
  archive/                      # Old/deprecated files — do not use
```

---

## 4. Configuration

### 4.1 Create Your .env File

All credentials and settings are read from a `.env` file in the project root:

```bash
cd swapnil_chatbox
cp .env.example .env
```

Open `.env` in your editor and update the following values:

```bash
# ── MySQL ─────────────────────────────────────────────────────
DASHBOARD_DB_HOST=127.0.0.1
DASHBOARD_DB_PORT=3307           # Port exposed by the Docker MySQL container
DASHBOARD_DB_PASSWORD=Dashboard12345
MYSQL_ADMIN_PASSWORD=dashboard_root   # MySQL root password inside Docker

# ── Application ───────────────────────────────────────────────
APP_PORT=3000                    # Port the Flask app listens on

# ── Ollama (AI chat model) ────────────────────────────────────
OLLAMA_HOST=http://localhost:11434    # Point to where Ollama is running
OLLAMA_MODELS=qwen2.5:7b             # Must match the model you pulled above
OLLAMA_READ_TIMEOUT=180
```

> **Do NOT** set `OLLAMA_HOST` to `10.82.7.78` — that is Stryker's internal GPU server, only reachable from inside their network via SSH tunnel. Use `http://localhost:11434` for local Ollama.

---

### 4.2 Install Python Dependencies

The project uses `setup.py` to declare all its dependencies. Install the package in editable mode using:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -e .
```

The `-e` flag installs the package in **editable mode** — meaning any changes you make to the source code are reflected immediately without needing to reinstall. This is the standard install method for local development.

Running `pip install -e .` installs all dependencies declared in `setup.py`, including:

- `flask` — web framework
- `requests` — HTTP client
- `mysql-connector-python` — MySQL database driver
- `chromadb` — vector store for document search
- `sentence-transformers` — embeddings for semantic search
- `langchain`, `langchain-community`, `langchain-ollama` — LLM orchestration
- `python-dotenv` — reads `.env` file
- `pydantic` — data validation

> `start.sh` runs the venv creation and install automatically. You only need to run `pip install -e .` manually if you are setting up outside of `start.sh` or adding the project as a dependency in another Python environment.

---

## 5. Running the Project

### 5.1 Start Everything with One Command

The `start.sh` script automates the entire startup sequence — it creates a Python venv, installs dependencies, starts a MySQL Docker container, applies all database schemas, caches a dashboard snapshot, and launches the Flask app via Gunicorn.

```bash
cd swapnil_chatbox
cp .env.example .env    # skip if you already did this
bash start.sh
```

You will see output progressing through 5 steps:

```
▶▶▶ 1/5 Prerequisites
✅ python3 + curl OK
▶▶▶ 2/5 Python environment
✅ Dependencies OK
▶▶▶ 3/5 Environment file
✅ .env loaded
▶▶▶ 4/5 MySQL (Docker)
✅ MySQL container running
✅ Schemas applied
▶▶▶ 5/5 Starting application
✅ Gunicorn started on http://127.0.0.1:3000
```

---

### 5.2 Access the Application

Once `start.sh` completes, the following URLs are available:

| URL | Description |
|---|---|
| http://127.0.0.1:3000 | Main Dashboard UI |
| http://127.0.0.1:3000/api/app/health | Health check — returns `{ status: ok }` |
| http://127.0.0.1:3000/api/app/preflight | Preflight — verifies DB and Ollama connectivity |
| http://127.0.0.1:3000/api/applications | Lists all configured applications |
| http://127.0.0.1:3000/api/insights?app_id=THA6.0 | Raw insights JSON for a specific app |

---

### 5.3 Verify MySQL Container is Running

```bash
# Confirm the container is up
docker ps
# You should see a container named: dashboard-mysql

# Confirm all 7 app databases were created
docker exec dashboard-mysql mysql -uroot -pdashboard_root \
  -e "SHOW DATABASES LIKE 'dashboard_%';"
```

Expected output:

```
Database (dashboard_%)
dashboard_clubber
dashboard_shoulder1_5
dashboard_shoulder2_0
dashboard_tha6_0
dashboard_tka3_0_1
dashboard_tka3_0ous
dashboard_tka4_0
```

---

## 6. Loading Sample Metric Data

After the server starts, the databases exist but are **empty**. The Insights tab will show no charts until data is loaded. The repository ships with two ways to do this.

---

### 6.1 Option A — Load from SQL Backup Files

The `software_metric_backups/` folder contains mysqldump files exported from a real deployment. The recommended file to load is `all_schemas_schema_data_20260524_191623.sql`, which populates data for **THA6.0, TKA4.0, and Clubber**.

**Step 1 — Load the backup:**

```bash
# Run from the project root (swapnil_chatbox/)
docker exec -i dashboard-mysql mysql -uroot -pdashboard_root \
  < software_metric_backups/all_schemas_schema_data_20260524_191623.sql
```

The command runs silently. It returns to the prompt when complete (may take 30–60 seconds).

**Step 2 — Verify data loaded:**

```bash
docker exec dashboard-mysql mysql -uroot -pdashboard_root -e "
SELECT 'tha6_0'  as app, COUNT(*) as cov_rows FROM dashboard_tha6_0.cov_job_logs
UNION ALL
SELECT 'tka4_0',         COUNT(*) FROM dashboard_tka4_0.cov_job_logs
UNION ALL
SELECT 'clubber',        COUNT(*) FROM dashboard_clubber.cov_job_logs;
"
```

Expected output:

```
app      cov_rows
tha6_0   16
tka4_0   4
clubber  4
```

Non-zero counts confirm the data loaded successfully.

> **Note on the other backup files:** The individual per-app `.sql` files (e.g. `dashboard_shoulder1_5_schema_data_*.sql`) contain only table schemas — no data rows. TKA3.0.1, TKA3.0OUS, Shoulder1.5, and Shoulder2.0 were never synced from GitLab before the backup was taken. Use Option B below for those apps.

---


### 6.3 Refresh the Dashboard

After loading data via either option, open the dashboard:

```
http://127.0.0.1:3000
```

Select any application from the dropdown and click **Insights**. You should see Metrics & Trends cards and trend charts populated with data. The AI chat widget in the bottom-right corner will now also have real numbers to answer questions about.

---

## 7. Using the Application

### 7.1 Dashboard Tabs

| Tab | Description |
|---|---|
| **Insights** | Metrics cards and trend charts for the selected app. Tabs include: Latest Results, Coverage Trend, Unit Test Trend, Squish Trend, TODO Trend, Pipeline History, ASAN Trend, Static Analysis, Peer Review, Defect Trend. |
| **Config** | Shows the per-application GitLab/Jira configuration from `files/dashboard/config/*.json`. |
| **Data Sync** | Triggers a live data pull from Stryker's GitLab CI/CD pipelines. Requires a valid `GITLAB_TOKEN` and network access to Stryker's internal GitLab. Will fail on external networks — use sample data instead. |
| **Doc Chat** | Upload documents (PDF, DOCX, XLSX) and ask questions about them. Requires `ENABLE_DOCUMENT_CHAT=1` and the full `requirements.txt`. |
| **Feedback** | Submit feedback about the dashboard. |

---

### 7.2 AI Chat Assistant

The chat widget (bottom-right corner of the dashboard) connects to Ollama and answers questions about the metrics in the database. Example questions you can ask:

- *What is the current function coverage for THA6.0?*
- *Compare all applications*
- *Which app has the most TODO/FIXME items?*
- *Is TKA4.0 improving or regressing on unit tests?*
- *What should I watch out for in Shoulder2.0?*

> The chat requires Ollama to be running and `OLLAMA_HOST` in `.env` to point to a reachable Ollama server with the model pulled. Verify with: `curl http://localhost:11434/api/tags`

---

## 8. Troubleshooting

| Symptom | Fix |
|---|---|
| `start.sh` says "Docker not found" | Run `newgrp docker` or log out and back in to apply group membership, then retry. |
| MySQL container fails to start (port conflict) | Run `sudo ss -ltnp \| grep 3307` — if something is on port 3307, stop it. Also check port 3306 with `sudo systemctl stop mysql`. |
| `Access denied` on MySQL commands | Use `-pdashboard_root` (the Docker root password), not the value of `DASHBOARD_DB_PASSWORD`. |
| Insights tab empty / no charts | Data has not been loaded. Follow Section 6 to load sample data. |
| Coverage % shows N/A | `cov_src_coverage` table is empty. Run `python3 seed_missing_apps.py` to populate it. |
| Squish shows N/A | `squish_test_results` table is empty. Run `seed_missing_apps.py` which inserts individual test result rows. |
| Chat says Ollama unreachable | Check `OLLAMA_HOST` in `.env`. Run `systemctl status ollama` and `curl http://localhost:11434/api/tags`. |
| Data Sync fails with GitLab error | Expected outside Stryker's network. Use sample data instead (Section 6). |
| `pip install` fails on `torch`/`chromadb` | Only needed for `ENABLE_DOCUMENT_CHAT=1`. Use `requirements.dashboard.txt` for standard setup. |
| VS Code notification about `.env` file injection | Harmless VS Code prompt — `start.sh` loads `.env` itself. Click "Don't Show Again". |

---

## 9. Quick Reference

```bash
# To install the depedencies required for this project..
Install as: pip install -e .

# Start the project
bash start.sh

# Check running containers
docker ps

# Open MySQL shell
docker exec -it dashboard-mysql mysql -uroot -pdashboard_root

# Load sample data backup
docker exec -i dashboard-mysql mysql -uroot -pdashboard_root \
  < software_metric_backups/all_schemas_schema_data_20260524_191623.sql

# Check application health
curl http://127.0.0.1:3000/api/app/health

# Check Ollama is serving
curl http://localhost:11434/api/tags

# Stop the MySQL container
docker stop dashboard-mysql

# Remove the MySQL container (WARNING: all data will be lost)
docker rm dashboard-mysql
```

---

*Software Metric Dashboard — Local Setup Guide*
*Last updated: June 2026*
