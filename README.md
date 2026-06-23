# Local AI Inference Stack with Podman

> A fully containerized, locally-hosted AI assistant — built on Rocky Linux and automated with Jenkins.  
> Now with built-in monitoring via **Prometheus** and **Grafana**.  
> Developed as part of the **UVT × IBM MLOps Collaboration**.
> 
---

## What This Project Does

This project deploys a **private, local AI assistant** (similar to ChatGPT) that runs entirely on your own machine — no internet required, no API costs, no data leaving your computer.

It combines four containers into one isolated stack:
- A **local LLM engine** that processes your questions
- A **chat interface** you can open in any browser
- A **metrics collector** that scrapes container and host metrics
- A **dashboard** to visualize the health and performance of the whole stack

The entire deployment is automated via a **Jenkins CI/CD pipeline** — one click and the whole stack comes up.

**Why local inference?**  
Cloud AI APIs are expensive at scale and raise privacy concerns. Running a model locally gives you full control, zero cost per query, and complete data privacy — key principles in modern MLOps.

**Why monitoring?**  
A local inference stack is still a production-style service. Prometheus and Grafana give visibility into container health, resource usage, and request patterns — without sending any of that data outside your machine either.

---

## Tech Stack

| Layer | Technology | Role |
|-------|-----------|------|
| OS | Rocky Linux 9 | Stable, enterprise-grade Linux base |
| Containers | Podman | Daemonless, rootless container engine |
| CI/CD | Jenkins | Automates the full deployment pipeline |
| AI Engine | llama.cpp | Runs the LLM efficiently on CPU |
| LLM Model | Qwen2.5 0.5B Instruct (Q4_K_M) | Lightweight quantized language model |
| Chat UI | Open WebUI | ChatGPT-like browser interface |
| Metrics Collection | Prometheus | Scrapes and stores time-series metrics from all containers |
| Dashboards | Grafana | Visualizes Prometheus metrics in real-time dashboards |
| Orchestration | Kubernetes (via `podman generate kube`) | Optional production deployment |

---

## Architecture

All containers communicate through an **isolated Podman bridge network** called `llm-net`. No container is exposed to the outside world unnecessarily.

```

                          llm-net (bridge network)                       
                                                                          
                                                            
      llama-server -------------- open-webui                          
      (port 8080)                 (port 3000)                         
            │                                                            
            │ metrics                                                    
            ▼                                                            
       prometheus ------------------- grafana                         
       (port 9090)                   (port 3001)                      
                                                            

```

**Data flow:**
1. You type a message in **Open WebUI** (browser)
2. It sends your prompt to **llama-server** through the internal network
3. **llama-server** runs inference on the local model and returns the response
4. **Prometheus** continuously scrapes metrics (CPU, memory, request counts/latency) from the containers
5. **Grafana** queries Prometheus and renders the metrics on dashboards you view in your browser

---

## Quick Start (VirtualBox)

> Use this section if you received a `.ova` appliance file from a teammate.

### Step 1 — Import the VM

1. Open **Oracle VM VirtualBox**
2. Go to **File → Import Appliance...**
3. Select the `rocky.ova` file → click **Next** → click **Finish**

### Step 2 — Fix the Network (Important!)

The VM must be on your local network so you can access it from your browser.

1. Right-click the VM → **Settings → Network**
2. Set **"Attached to"** → `Bridged Adapter`
3. Set **"Name"** → your active network card:
   - On Wi-Fi → select your wireless adapter (e.g., `Intel Wireless`, `Realtek Wi-Fi`)
   - On Ethernet → select your Ethernet adapter (e.g., `Gigabit Ethernet`, `PCIe LAN`)
4. Click **OK**

### Step 3 — Start the VM and get the IP

Start the VM. Once it boots, log in and run:

```bash
hostname -I
```

You'll see something like `192.168.1.42`. **Save this IP** — you'll use it in your browser instead of `localhost`.

### Step 4 — Start the containers

```bash
sudo podman start llama-server open-webui prometheus grafana
```

### Step 5 — Disable the firewall (for local access)

```bash
sudo systemctl stop firewalld
```

### Step 6 — Open the chat interface

On your **host machine** (Windows/Mac), open a browser and go to:

```
http://<YOUR_VM_IP>:3000
```

You should see the Open WebUI chat interface. Type a message and the local AI will respond.

### Step 7 — Open the monitoring dashboard

```
http://<YOUR_VM_IP>:3001
```

Log in to Grafana and open the **AI Stack Overview** dashboard to see live metrics.

---

## Automated Setup with Jenkins

The full deployment is managed by the `Jenkinsfile` in this repository. Jenkins runs each step in order, handles errors, and verifies the result.

### Pipeline Stages

| # | Stage | What it does |
|---|-------|-------------|
| 1 | **Cleanup** | Removes stale Podman runtime files from `/tmp/` |
| 2 | **Create `/opt/models`** | Creates the model storage directory with correct permissions |
| 3 | **Create network** | Initializes the `llm-net` bridge network |
| 4 | **Download model** | Fetches the Qwen2.5 `.gguf` file from HuggingFace (skipped if cached) |
| 5 | **Start llama-server** | Deploys the inference container |
| 6 | **Start open-webui** | Deploys the chat interface container |
| 7 | **Start Prometheus** | Deploys the metrics collector with the stack's scrape config |
| 8 | **Start Grafana** | Deploys the dashboard container and provisions the default dashboard |
| 9 | **Verify containers** | Confirms all containers are in `Running` state |

### How to trigger a deployment

1. Open Jenkins: `http://<VM_IP>:8081`
2. Open the **AI-deploy** pipeline
3. Click **Build Now**
4. Watch the stages complete in real time

---

## Accessing the Services

> Replace `<VM_IP>` with the IP from `hostname -I`.  
> Always use `http://` — **not** `https://`.

| Service | URL | What you'll see |
|---------|-----|-----------------|
| Chat Interface | `http://<VM_IP>:3000` | Open WebUI — chat with the local AI |
| Jenkins | `http://<VM_IP>:8081` | CI/CD pipeline and build history |
| AI API | `http://<VM_IP>:8080` | llama-server REST API (OpenAI-compatible) |
| Prometheus | `http://<VM_IP>:9090` | Raw metrics, targets, and query browser |
| Grafana | `http://<VM_IP>:3001` | Dashboards visualizing the whole stack |

### API example (from terminal)

```bash
curl -X POST http://<VM_IP>:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "What is Podman?"}
    ]
  }'
```

### Prometheus query example

Check whether Prometheus sees the llama-server target as healthy by opening:

```
http://<VM_IP>:9090/targets
```

---

## Monitoring Stack Details

### Prometheus

- Scrapes metrics from `llama-server` and `open-webui` (and itself) on a configurable interval (default: 15s)
- Configuration lives in `prometheus.yml`, mounted into the container at `/etc/prometheus/prometheus.yml`
- Stores time-series data in a local volume so history survives container restarts

Example scrape config snippet:

```yaml
scrape_configs:
  - job_name: 'llama-server'
    static_configs:
      - targets: ['llama-server:8080']
  - job_name: 'open-webui'
    static_configs:
      - targets: ['open-webui:3000']
```

### Grafana

- Connects to Prometheus as its default data source (`http://prometheus:9090`)
- Ships with a provisioned **AI Stack Overview** dashboard showing:
  - Request latency and throughput for llama-server
  - Container CPU and memory usage
  - Container uptime / restart counts
- Default login: `admin` / `admin` (you'll be prompted to change this on first login)

---

## Kubernetes Deployment

The stack can be exported to Kubernetes manifests directly from Podman:

```bash
# Generate YAML from currently running containers
sudo podman generate kube llama-server open-webui prometheus grafana > stack.yaml

# Deploy to a Kubernetes cluster
kubectl apply -f stack.yaml

# Or replay locally with Podman
podman play kube stack.yaml
```

---

## Project Structure

```
ibm-mlops/
 Jenkinsfile           # Full CI/CD pipeline definition
 stack.yaml            # Kubernetes-compatible deployment manifest
 prometheus.yml        # Prometheus scrape configuration
 grafana/
    dashboards/        # Provisioned Grafana dashboard JSON
    datasources/       # Prometheus datasource provisioning config
 README.md             # This file
```

---

## Troubleshooting

### `hostname -I` returns nothing or a `10.0.x.x` address

The VM is using NAT instead of Bridged networking.

**Fix:** Shut down the VM → VirtualBox Settings → Network → set to **Bridged Adapter** → select your correct network card → restart the VM → run `sudo systemctl restart NetworkManager`.

---

### Can't connect to `<VM_IP>:3000` in the browser

Three possible causes:

1. **Firewall is active** → run `sudo systemctl stop firewalld`
2. **Containers aren't running** → run `sudo podman start llama-server open-webui prometheus grafana`
3. **Wrong network adapter selected** → if you switched from Wi-Fi to Ethernet (or vice versa), your IP changed. Run `hostname -I` again and use the new address.

---

### Containers show as stopped after a VM reboot

Containers don't auto-start by default. Run:

```bash
sudo podman start llama-server open-webui prometheus grafana
```

---

### Port conflict — llama-server won't start

If Jenkins is already using port `8080`, the llama-server container will fail to bind.

**Fix:** In the `Jenkinsfile`, change the port mapping from `-p 8080:8080` to `-p 8081:8080`. Then access the AI API at port `8081` instead.

---

### Grafana dashboard shows "No Data"

This usually means Prometheus isn't reachable from Grafana, or no targets are being scraped yet.

1. Check Prometheus targets are `UP`: `http://<VM_IP>:9090/targets`
2. Confirm Grafana's data source URL points to `http://prometheus:9090` (the in-network hostname, not `localhost`)
3. Restart Grafana after fixing the data source: `sudo podman restart grafana`

---

### Port conflict — Grafana or Prometheus won't start

Grafana defaults to port `3000`, which collides with Open WebUI in this stack.

**Fix:** Grafana is mapped to host port `3001` instead (`-p 3001:3000` in the `Jenkinsfile`). If `9090` or `3001` are already taken on your host, change the host-side port mapping in the `Jenkinsfile` and update the URLs in this README accordingly.

---

### Container crashed — how to read the logs

```bash
sudo podman logs llama-server
sudo podman logs open-webui
sudo podman logs prometheus
sudo podman logs grafana
```

---

### Check all container statuses at once

```bash
sudo podman ps -a
```

Containers with status `Exited` need to be started or debugged.

---

*Built with Rocky Linux · Podman · Jenkins · llama.cpp · Open WebUI · Prometheus · Grafana*
