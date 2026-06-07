<h1 align="center">ansible-server-bootstrap</h1>

<p align="center">
  Ansible automation for turning a fresh Ubuntu server into a hardened Docker host with Nginx, UFW, Node Exporter, and an uptime monitoring app.
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Ansible-Automation-EE0000?style=flat-square&logo=ansible&logoColor=white" />
  <img src="https://img.shields.io/badge/Ubuntu-22.04-E95420?style=flat-square&logo=ubuntu&logoColor=white" />
  <img src="https://img.shields.io/badge/Nginx-Reverse%20Proxy-009639?style=flat-square&logo=nginx&logoColor=white" />
  <img src="https://img.shields.io/badge/Docker-Host-2496ED?style=flat-square&logo=docker&logoColor=white" />
  <img src="https://img.shields.io/badge/Prometheus-Node%20Exporter-E6522C?style=flat-square&logo=prometheus&logoColor=white" />
</p>

---

## Portfolio Context

This repository is the infrastructure side of a DevOps and SysAdmin portfolio lab.

It deploys [`go-uptime-monitor`](https://github.com/egayurcel990/go-uptime-monitor), a self-hosted uptime monitoring service, onto a Linux server using repeatable Ansible roles.

The goal is to show a practical server lifecycle:

1. Start with a clean Ubuntu server.
2. Apply baseline packages and timezone configuration.
3. Harden SSH and install fail2ban.
4. Configure firewall rules with UFW.
5. Install Docker and Docker Compose.
6. Configure Nginx as a reverse proxy.
7. Install Prometheus Node Exporter for server metrics.
8. Deploy and update a containerized application from GHCR.

## What This Automates

Run one command:

```bash
ansible-playbook -i inventory/hosts.yml site.yml
```

The playbook prepares the server and deploys the application with idempotent roles.

## Architecture

```text
Ansible control node
    |
    | SSH key-based connection
    v
Ubuntu 22.04 server
    |
    |-- SSH hardening
    |   |-- root login disabled
    |   |-- password auth disabled
    |   `-- fail2ban enabled
    |
    |-- UFW firewall
    |   |-- 22/tcp SSH
    |   |-- 80/tcp HTTP
    |   |-- 443/tcp HTTPS
    |   `-- 9100/tcp Node Exporter
    |
    |-- Nginx reverse proxy
    |   `-- port 80 -> 127.0.0.1:8080
    |
    |-- Docker host
    |   `-- go-uptime-monitor container
    |
    `-- Prometheus Node Exporter
        `-- /metrics on port 9100
```

## Automation Coverage

| Role | Responsibility |
|------|----------------|
| `common` | Updates packages, installs base tools, sets timezone |
| `ssh-hardening` | Hardens SSH and enables fail2ban |
| `ufw` | Applies default-deny firewall policy and allows required ports |
| `docker` | Installs Docker CE and Docker Compose plugin |
| `nginx` | Configures Nginx reverse proxy for the app |
| `node-exporter` | Installs Node Exporter as a systemd service |
| `app-deploy` | Pulls the app image from GHCR and runs it with Docker Compose |

## Screenshots

Health check:

![Health check](docs/healthz.png)

API status:

![API status](docs/api-status.png)

Node Exporter metrics:

![Node Exporter metrics](docs/metrics.png)

## Prerequisites

For local testing:

- VirtualBox
- Vagrant
- Ansible
- `community.docker` Ansible collection

Install the Docker collection:

```bash
ansible-galaxy collection install community.docker
```

On Windows, run Ansible from WSL because Ansible does not run natively on Windows.

## Quick Start: Local VM

### 1. Clone the repository

```bash
git clone https://github.com/egayurcel990/ansible-server-bootstrap
cd ansible-server-bootstrap
```

### 2. Generate an SSH key for Ansible

```bash
mkdir -p ~/.ssh/ansible-server-bootstrap
ssh-keygen -t ed25519 -f ~/.ssh/ansible-server-bootstrap/vagrant_private_key -N ""
```

### 3. Start the VM

```bash
vagrant up
```

The VM uses:

| Setting | Value |
|---------|-------|
| Guest OS | Ubuntu 22.04 |
| Private IP | `192.168.56.10` |
| App URL from host | `http://localhost:8080` |
| Node Exporter from host | `http://localhost:9100/metrics` |

### 4. Copy the SSH key into the VM

```bash
ssh-copy-id -i ~/.ssh/ansible-server-bootstrap/vagrant_private_key.pub \
  -o StrictHostKeyChecking=no \
  vagrant@192.168.56.10
```

Default Vagrant password: `vagrant`.

### 5. Run the playbook

```bash
ansible-playbook -i inventory/hosts.yml site.yml
```

The full run usually takes a few minutes on the first setup.

### 6. Verify the deployment

Open the dashboard:

```text
http://localhost:8080
```

Check the health endpoint:

```bash
curl http://localhost:8080/healthz
```

Add a monitored target:

```bash
curl -X POST http://192.168.56.10/api/v1/targets \
  -H "Content-Type: application/json" \
  -d '{"name": "Example", "url": "https://example.com", "interval": 60}'
```

Check server metrics:

```bash
curl http://localhost:9100/metrics
```

## Running Specific Roles

Use tags to re-run only part of the system:

```bash
ansible-playbook -i inventory/hosts.yml site.yml --tags ssh
ansible-playbook -i inventory/hosts.yml site.yml --tags firewall
ansible-playbook -i inventory/hosts.yml site.yml --tags docker
ansible-playbook -i inventory/hosts.yml site.yml --tags nginx
ansible-playbook -i inventory/hosts.yml site.yml --tags monitoring
ansible-playbook -i inventory/hosts.yml site.yml --tags app
```

## Smart App Deploy

The `app-deploy` role pulls the latest image and recreates the container only when the image changes.

```text
Image unchanged -> keep container running
Image updated   -> recreate container from the new image
```

This makes `--tags app` safe to run repeatedly during deployments.

## Configuration

Edit `inventory/group_vars/all.yml`.

```yaml
# SSH
ssh_port: 22
ssh_user: vagrant

# App
app_image: ghcr.io/egayurcel990/go-uptime-monitor:latest
app_port: 8080

# Monitoring
node_exporter_port: 9100
node_exporter_version: "1.7.0"
```

For a real VPS or cloud instance, update `inventory/hosts.yml`:

```yaml
all:
  hosts:
    myserver:
      ansible_host: YOUR_SERVER_IP
      ansible_user: ubuntu
      ansible_ssh_private_key_file: ~/.ssh/id_rsa
```

This works with Ubuntu 22.04 servers on AWS EC2, Oracle Cloud, DigitalOcean, or similar VPS providers.

## Project Structure

```text
ansible-server-bootstrap/
|-- inventory/
|   |-- hosts.yml
|   `-- group_vars/all.yml
|-- roles/
|   |-- app-deploy/
|   |-- common/
|   |-- docker/
|   |-- nginx/
|   |-- node-exporter/
|   |-- ssh-hardening/
|   `-- ufw/
|-- ansible.cfg
|-- site.yml
|-- Vagrantfile
`-- README.md
```

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `vagrant up` fails with virtualization error | Enable virtualization in BIOS/UEFI |
| Ansible returns `unreachable=1` | Wait for the VM to finish booting and retry |
| `Permission denied (publickey)` | Re-run the SSH key copy step |
| Dashboard does not load on `localhost:8080` | Run `vagrant reload` and re-run the playbook |
| Container is not running | SSH into the VM and run `docker logs uptime-monitor` |
| Nginx does not proxy correctly | Check `sudo nginx -t` and `sudo systemctl status nginx` |

## What This Demonstrates

- Linux server provisioning with Ansible.
- SSH hardening and firewall baseline practices.
- Docker host setup and container deployment.
- Nginx reverse proxy configuration.
- Server metrics exposure with Node Exporter.
- Repeatable local testing with Vagrant.
- A deploy workflow connected to a real application repository.

## Roadmap

- Add TLS automation with Certbot.
- Add Prometheus and Grafana stack for full observability.
- Add Molecule tests for Ansible roles.
- Add support for AWS EC2 provisioning with Terraform.
- Add environment-specific inventories for local, staging, and cloud.

---

<p align="center">
  <i>Infrastructure layer for a DevOps and SysAdmin portfolio lab by Ega Yurcel Satriaji.</i>
</p>
