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
| `app-deploy` | Pull go-uptime-monitor image from GHCR, run via Docker Compose |

---

## Architecture

```
Ansible Control Node (your machine)
            │
            │ SSH (key-based)
            ▼
┌──────────────────────────────────┐
│        Ubuntu 22.04 Server       │
│                                  │
│  UFW Firewall                    │
│  ├─ Port 22    (SSH)             │
│  ├─ Port 80    (HTTP → Nginx)    │
│  ├─ Port 443   (HTTPS)           │
│  └─ Port 9100  (Node Exporter)   │
│                                  │
│  Nginx (reverse proxy)           │
│  └─ :80 → 127.0.0.1:8080        │
│                                  │
│  Docker                          │
│  └─ go-uptime-monitor (127.0.0.1:8080) │
│                                  │
│  Prometheus Node Exporter        │
│  └─ :9100 /metrics               │
└──────────────────────────────────┘
```

---

## Quick Start

**Prerequisites:** Ansible, VirtualBox, Vagrant

```bash
git clone https://github.com/egayurcel990/ansible-server-bootstrap
cd ansible-server-bootstrap

# Spin up test VM
vagrant up

# Run full bootstrap
ansible-playbook -i inventory/hosts.yml site.yml

# Run specific role only
ansible-playbook -i inventory/hosts.yml site.yml --tags ssh
ansible-playbook -i inventory/hosts.yml site.yml --tags nginx
ansible-playbook -i inventory/hosts.yml site.yml --tags app
```

After bootstrap completes, the app is available at **http://192.168.56.10** (or `http://localhost:8080` via the Vagrant port forward).

---

## Project Structure

```
ansible-server-bootstrap/
├── inventory/
│   ├── hosts.yml                   # Target hosts and connection config
│   └── group_vars/
│       └── all.yml                 # Shared variables (SSH port, image, app port, etc.)
├── roles/
│   ├── common/
│   │   ├── tasks/main.yml          # Package updates, timezone
│   │   └── defaults/main.yml       # Default package list
│   ├── ssh-hardening/
│   │   ├── tasks/main.yml
│   │   ├── handlers/main.yml
│   │   └── templates/sshd_config.j2
│   ├── ufw/
│   │   └── tasks/main.yml
│   ├── nginx/
│   │   ├── tasks/main.yml
│   │   ├── handlers/main.yml
│   │   └── templates/app.conf.j2
│   ├── docker/
│   │   └── tasks/main.yml
│   ├── node-exporter/
│   │   ├── tasks/main.yml
│   │   ├── handlers/main.yml
│   │   └── templates/node-exporter.service.j2
│   └── app-deploy/
│       ├── tasks/main.yml
│       └── templates/docker-compose.yml.j2
├── site.yml                        # Master playbook
├── Vagrantfile                     # Local test VM (Ubuntu 22.04)
└── README.md
```

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

## Testing Locally with Vagrant

```bash
vagrant up                                         # Start Ubuntu 22.04 VM
ansible-playbook -i inventory/hosts.yml site.yml   # Bootstrap the VM
vagrant ssh                                        # SSH in to verify manually
vagrant destroy -f && vagrant up                   # Full clean rebuild
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

<p align="center">
  <i>Deploys <a href="https://github.com/egayurcel990/go-uptime-monitor">go-uptime-monitor</a> · Universitas Brawijaya · 2025</i>
</p>
