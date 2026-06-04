<h1 align="center">ansible-server-bootstrap</h1>

<p align="center">
  Automate Ubuntu server setup from zero to production-ready — one command, fully idempotent.
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Ansible-EE0000?style=flat-square&logo=ansible&logoColor=white" />
  <img src="https://img.shields.io/badge/Ubuntu-22.04-E95420?style=flat-square&logo=ubuntu&logoColor=white" />
  <img src="https://img.shields.io/badge/Nginx-Reverse%20Proxy-009639?style=flat-square&logo=nginx&logoColor=white" />
  <img src="https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white" />
  <img src="https://img.shields.io/badge/Prometheus-Node%20Exporter-E6522C?style=flat-square&logo=prometheus&logoColor=white" />
  <img src="https://img.shields.io/badge/Vagrant-VirtualBox-1868F2?style=flat-square&logo=vagrant&logoColor=white" />
</p>

---

## What This Does

Run one command and get a hardened, monitored, production-ready Ubuntu server with [go-uptime-monitor](https://github.com/egayurcel990/go-uptime-monitor) deployed on top.

```bash
ansible-playbook -i inventory/hosts.yml site.yml
```

---

## Automation Coverage

| Role | What it does |
|------|--------------|
| `common` | Update & upgrade packages, install essentials, set timezone (Asia/Jakarta) |
| `ssh-hardening` | Disable root login, disable password auth, enforce key-based auth, install and enable fail2ban |
| `ufw` | Enable firewall with default-deny, whitelist SSH, HTTP, HTTPS, and Node Exporter |
| `docker` | Install Docker CE + Compose plugin, add deploy user to docker group |
| `nginx` | Install Nginx, configure as reverse proxy for the app (port 80 → app container) |
| `node-exporter` | Install Prometheus Node Exporter as a dedicated systemd service |
| `app-deploy` | Pull go-uptime-monitor image from GHCR, auto-recreate container if image updated |

---

## Architecture

```
Ansible Control Node (your machine)
            │
            │ SSH (key-based)
            ▼
┌──────────────────────────────────────┐
│          Ubuntu 22.04 Server         │
│                                      │
│  UFW Firewall                        │
│  ├─ Port 22    (SSH)                 │
│  ├─ Port 80    (HTTP → Nginx)        │
│  ├─ Port 443   (HTTPS)               │
│  └─ Port 9100  (Node Exporter)       │
│                                      │
│  Nginx (reverse proxy)               │
│  └─ :80 → 127.0.0.1:8080            │
│                                      │
│  Docker                              │
│  └─ go-uptime-monitor (127.0.0.1:8080) │
│                                      │
│  Prometheus Node Exporter            │
│  └─ :9100 /metrics                   │
└──────────────────────────────────────┘
```

---

## Prerequisites

Before you begin, make sure the following tools are installed on your machine.

### 1. VirtualBox

VirtualBox is the virtualization provider used to run the local Ubuntu VM.

- Download: https://www.virtualbox.org/wiki/Downloads
- Pick the installer for your OS (Windows, macOS, or Linux) and run it

### 2. Vagrant

Vagrant manages the VM lifecycle (create, start, stop, destroy).

- Download: https://developer.hashicorp.com/vagrant/downloads
- Pick the installer for your OS and run it

Verify both are installed:
```bash
vagrant --version    # should print e.g. Vagrant 2.4.x
vboxmanage --version # should print e.g. 7.0.x
```

### 3. Ansible

Ansible runs the playbooks from your machine to configure the VM over SSH.

**Ubuntu / Debian / WSL:**
```bash
sudo apt update
sudo apt install -y ansible
```

**macOS:**
```bash
brew install ansible
```

**Windows (native):** Ansible does not run natively on Windows. Use WSL (Windows Subsystem for Linux) and install Ansible inside it following the Ubuntu steps above.

Verify:
```bash
ansible --version # should print e.g. ansible [core 2.x.x]
```

### 4. Ansible Docker Collection

The `app-deploy` role uses the `community.docker` collection to manage containers.

```bash
ansible-galaxy collection install community.docker
```

---

## Quick Start (Local VM with Vagrant)

### Step 1 — Clone the repo

```bash
git clone https://github.com/egayurcel990/ansible-server-bootstrap
cd ansible-server-bootstrap
```

### Step 2 — Generate an SSH key for Ansible

This key will be used by Ansible to connect to the VM. Skip this step if you already have one you want to use.

```bash
mkdir -p ~/.ssh/ansible-server-bootstrap
ssh-keygen -t ed25519 -f ~/.ssh/ansible-server-bootstrap/vagrant_private_key -N ""
```

### Step 3 — Start the VM

```bash
vagrant up
```

This downloads the Ubuntu 22.04 box (only on first run, ~500MB) and starts the VM. The VM will be accessible at `192.168.56.10`.

### Step 4 — Copy your SSH key into the VM

```bash
ssh-copy-id -i ~/.ssh/ansible-server-bootstrap/vagrant_private_key.pub \
  -o StrictHostKeyChecking=no \
  -p 22 vagrant@192.168.56.10
# Default Vagrant password: vagrant
```

### Step 5 — Run the playbook

```bash
ansible-playbook -i inventory/hosts.yml site.yml
```

You should see all tasks complete with `failed=0`. The full run takes about 3–5 minutes.

### Step 6 — Open the dashboard

Open your browser and go to **http://localhost:8080**

You should see the Go Uptime Monitor dashboard. Add a target to start monitoring:

```bash
curl -X POST http://192.168.56.10/api/v1/targets \
  -H "Content-Type: application/json" \
  -d '{"name": "Google", "url": "https://google.com", "interval": 60}'
```

---

## Stopping and Starting the VM

```bash
# Stop the VM (data is preserved)
vagrant halt

# Start it again later
vagrant up

# Destroy the VM completely (data is lost)
vagrant destroy -f
```

---

## Running Specific Roles

You can re-run individual roles without running the full playbook using tags:

```bash
ansible-playbook -i inventory/hosts.yml site.yml --tags ssh
ansible-playbook -i inventory/hosts.yml site.yml --tags docker
ansible-playbook -i inventory/hosts.yml site.yml --tags nginx
ansible-playbook -i inventory/hosts.yml site.yml --tags app
```

---

## Smart Deploy — Auto-Recreate on Image Update

The `app-deploy` role detects whether a new image is available and only recreates the container when needed:

```
Pull latest image  →  image unchanged  →  ensure container is running (no restart)
Pull latest image  →  image updated    →  recreate container with new image
```

This means running `--tags app` is always safe — it will never unnecessarily restart the container, but will always pick up a new image when one is pushed.

---

## Configuration

Edit `inventory/group_vars/all.yml` to customize:

```yaml
# SSH
ssh_port: 22        # Change to a non-default port for real servers
ssh_user: vagrant   # The deploy user (use 'ubuntu' for AWS EC2, etc.)

# App
app_image: ghcr.io/egayurcel990/go-uptime-monitor:latest
app_port: 8080

# Monitoring
node_exporter_port: 9100
node_exporter_version: "1.7.0"
```

---

## Project Structure

```
ansible-server-bootstrap/
├── inventory/
│   ├── hosts.yml                   # Target hosts and connection config
│   └── group_vars/
│       └── all.yml                 # Shared variables
├── roles/
│   ├── common/                     # Package updates, timezone
│   ├── ssh-hardening/              # sshd config, fail2ban
│   ├── ufw/                        # Firewall rules
│   ├── nginx/                      # Reverse proxy config
│   ├── docker/                     # Docker CE install
│   ├── node-exporter/              # Prometheus Node Exporter
│   └── app-deploy/                 # Pull and run the app container
├── site.yml                        # Master playbook
├── Vagrantfile                     # Local test VM (Ubuntu 22.04)
└── README.md
```

---

## Extending to Real Servers

Swap out the inventory to point at a real VPS:

```yaml
# inventory/hosts.yml
all:
  hosts:
    myserver:
      ansible_host: YOUR_SERVER_IP
      ansible_user: ubuntu
      ansible_ssh_private_key_file: ~/.ssh/id_rsa
```

Works with AWS EC2, Oracle Cloud, DigitalOcean, or any Ubuntu 22.04 VPS — same playbook, no changes needed.

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `vagrant up` fails with "VT-x not available" | Enable virtualization in your BIOS/UEFI settings |
| `unreachable=1` in Ansible | VM not fully booted yet — wait 30s and retry |
| `Permission denied (publickey)` | Re-run Step 4 to copy the SSH key into the VM |
| Dashboard not loading at localhost:8080 | Run `vagrant reload` to re-apply port forwarding |
| Container not running | SSH into VM with `vagrant ssh`, then check `docker logs uptime-monitor` |

---

<p align="center">
  <i>Deploys <a href="https://github.com/egayurcel990/go-uptime-monitor">go-uptime-monitor</a> · Ega Yurcel Satriaji · 2025</i>
</p>
