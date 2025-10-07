# K3s Production-Ready Ansible Playbook

This repository provides a production-ready Ansible playbook for deploying a K3s Kubernetes cluster with reusable roles and secure defaults.

## ✅ Features
- Declarative configuration via `config.yaml`
- Secure token handling from `secrets/`
- Role-based, modular structure
- SSH-based provisioning (inventory-driven)
- No `traefik`, `servicelb`, or `metrics-server` — you decide what to deploy

## 📁 Structure
```
playbooks/k3s-cluster/site.yml      # Main playbook
inventory/production.ini            # Your inventory
roles/
  ├── common/                       # Base system config
  ├── k3s_server/                   # K3s server setup
  ├── k3s_agent/                    # K3s agent setup
  └── firewall/                     # Optional UFW setup
secrets/
  └── k3s_token                     # Shared join token (not committed)
```

## 🖥️ Inventory Example (`inventory/production.ini`)
```ini
[k3s_servers]
server1 ansible_host=192.168.1.10

[k3s_agents]
agent1 ansible_host=192.168.1.11
agent2 ansible_host=192.168.1.12

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/id_rsa
k3s_server_url=https://192.168.1.10:6443
```


## 🚀 Deploy the Cluster
```bash
ansible-playbook -i inventory/production.ini playbooks/k3s-cluster/site.yml
```

## 📌 Requirements
- Python 3 + Ansible >= 2.10
- SSH access to all nodes
- Ubuntu 20.04/22.04 (can be extended)

## 🛠 Recommended Next Steps
- Install `nginx-ingress` via Helm
- Deploy Laravel app via Helm chart or Kustomize
- Add monitoring (Prometheus + Grafana)
- Configure backups (e.g., Velero, etcd snapshots)


