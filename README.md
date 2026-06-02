# Local AI Inference Stack with Podman

> A fully containerized, locally-hosted AI assistant — built on Rocky Linux and automated with Jenkins.  
> Developed as part of the **UVT × IBM MLOps Collaboration**.

---

## Table of Contents

1. [What This Project Does](#-what-this-project-does)
2. [Tech Stack](#-tech-stack)
3. [Architecture](#-architecture)
4. [Quick Start (VirtualBox)](#-quick-start-virtualbox)
5. [Manual Setup (Without Jenkins)](#-manual-setup-without-jenkins)
6. [Automated Setup with Jenkins](#-automated-setup-with-jenkins)
7. [Accessing the Services](#-accessing-the-services)
8. [Kubernetes Deployment](#-kubernetes-deployment)
9. [Project Structure](#-project-structure)
10. [Troubleshooting](#-troubleshooting)

---

## What This Project Does

This project deploys a **private, local AI assistant** (similar to ChatGPT) that runs entirely on your own machine — no internet required, no API costs, no data leaving your computer.

It combines two containers into one isolated stack:
- A **local LLM engine** that processes your questions
- A **chat interface** you can open in any browser

The entire deployment is automated via a **Jenkins CI/CD pipeline** — one click and the whole stack comes up.

**Why local inference?**  
Cloud AI APIs are expensive at scale and raise privacy concerns. Running a model locally gives you full control, zero cost per query, and complete data privacy — key principles in modern MLOps.

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
| Orchestration | Kubernetes (via `podman generate kube`) | Optional production deployment |

---

## Architecture

Both containers communicate through an **isolated Podman bridge network** called `llm-net`. No container is exposed to the outside world unnecessarily.

```

                    llm-net (bridge network)              
                                                          
              
      llama-server         open-webui          
      (port 8080)                 (port 3000)         
              

```

**Data flow:**
1. You type a message in **Open WebUI** (browser)
2. It sends your prompt to **llama-server** through the internal network
3. **llama-server** runs inference on the local model and returns the response

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
sudo podman start llama-server open-webui
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

---

## Manual Setup (Without Jenkins)

Use this if you want to set up the stack from scratch on a fresh Rocky Linux machine.

### Prerequisites

- Rocky Linux 9
- Podman installed
- Jenkins running on port `8081`
- At least **6 GB RAM** and **20 GB disk space**

### Step 1 — Create the models directory

```bash
sudo mkdir -p /opt/models
```

### Step 2 — Create the container network

```bash
sudo podman network create llm-net
```

### Step 3 — Download the AI model

```bash
sudo curl -L -o /opt/models/model.gguf \
  "https://huggingface.co/Qwen/Qwen2.5-0.5B-Instruct-GGUF/resolve/main/qwen2.5-0.5b-instruct-q4_k_m.gguf"
```

> WARNING: This file is approximately 400 MB. Make sure you have a stable connection.

### Step 4 — Start all containers

```bash
# AI inference engine
sudo podman run -d --name llama-server --network llm-net \
  -v /opt/models:/models:Z -p 8080:8080 \
  ghcr.io/ggml-org/llama.cpp:server \
  -m /models/model.gguf --host 0.0.0.0 --port 8080

# Chat interface
sudo podman run -d --name open-webui --network llm-net \
  -p 3000:8080 \
  -e OPENAI_API_BASE_URL=http://llama-server:8080/v1 \
  -e OPENAI_API_KEY=sk-no-key-required \
  ghcr.io/open-webui/open-webui:main
```

### Verify everything is running

```bash
sudo podman ps
```

Both containers should show status `Up`.

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
| 7 | **Verify containers** | Confirms all containers are in `Running` state |

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

---

## Kubernetes Deployment

The stack can be exported to Kubernetes manifests directly from Podman:

```bash
# Generate YAML from currently running containers
sudo podman generate kube llama-server open-webui > stack.yaml

# Deploy to a Kubernetes cluster
kubectl apply -f stack.yaml

# Or replay locally with Podman
podman play kube stack.yaml
```

---

## Project Structure

```
ibm-mlops/
 Jenkinsfile         # Full CI/CD pipeline definition
 stack.yaml          # Kubernetes-compatible deployment manifest
 README.md           # This file
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
2. **Containers aren't running** → run `sudo podman start llama-server open-webui`
3. **Wrong network adapter selected** → if you switched from Wi-Fi to Ethernet (or vice versa), your IP changed. Run `hostname -I` again and use the new address.

---

### Containers show as stopped after a VM reboot

Containers don't auto-start by default. Run:

```bash
sudo podman start llama-server open-webui
```

---

### Port conflict — llama-server won't start

If Jenkins is already using port `8080`, the llama-server container will fail to bind.

**Fix:** In the `Jenkinsfile`, change the port mapping from `-p 8080:8080` to `-p 8081:8080`. Then access the AI API at port `8081` instead.

---

### Container crashed — how to read the logs

```bash
sudo podman logs llama-server
sudo podman logs open-webui
```

---

### Check all container statuses at once

```bash
sudo podman ps -a
```

Containers with status `Exited` need to be started or debugged.

---

*Built with Rocky Linux · Podman · Jenkins · llama.cpp · Open WebUI*
