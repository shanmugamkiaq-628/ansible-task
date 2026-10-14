# Ansible DevOps Practice Tasks (Ubuntu)

This project provides step-by-step Ansible playbooks for the eight tasks in the supplied list:
1. Ansible inventory and SSH
2. Docker Engine and Docker Compose plugin
3. Nginx
4. Jenkins
5. Harbor Registry
6. HashiCorp Vault
7. SonarQube
8. Prometheus and Grafana

## Important safety / deployment notes

- Tested design target: Ubuntu 22.04/24.04 managed hosts with SSH and Python 3. Commands and upstream repository instructions can change; verify vendor docs before production use.
- Run these on **practice VMs**, not production hosts. The playbooks install services and may open ports.
- Do not expose admin UIs or credentials directly to the public internet. Use a private network, VPN, firewall allowlists, TLS, and strong secrets.
- This is a learning baseline, not a hardened production deployment. Harbor, Vault, SonarQube, Jenkins, Prometheus, and Grafana require additional security, persistence, backups, TLS, access controls, and resource planning for production.
- Harbor's installer is a separate vendor-distributed bundle. The Harbor playbook downloads the official online installer archive and creates a basic HTTP configuration for an isolated lab. For production, configure HTTPS and a real certificate before use.
- Vault's dev mode is **intentionally insecure and volatile**. It binds to localhost and is suitable only for learning the CLI/API on the Vault host. Do not use dev mode for real secrets.
- SonarQube uses PostgreSQL in Docker Compose and needs sufficient RAM; a 4 GB or larger VM is a practical lab starting point. Docker's `vm.max_map_count` is configured for Elasticsearch.
- Jenkins is installed from the official Debian package repository and Java 21 is installed first.
- For simple single-node lab deployments, run separate playbooks on separate Ubuntu VMs where possible. Ports can conflict if all apps are installed on one host.

## Repository structure

```text
ansible-devops-tasks/
├── ansible.cfg
├── inventory.ini.example
├── requirements.yml
├── group_vars/all.yml.example
├── site/
│   ├── 01-ssh-check.yml
│   ├── 02-docker-compose.yml
│   ├── 03-nginx.yml
│   ├── 04-jenkins.yml
│   ├── 05-harbor.yml
│   ├── 06-vault-dev-lab.yml
│   ├── 07-sonarqube.yml
│   └── 08-prometheus-grafana.yml
├── templates/
│   ├── nginx-index.html.j2
│   ├── prometheus.yml.j2
│   ├── prometheus.service.j2
│   └── grafana-datasource.yml.j2
└── scripts/
    └── test-all.sh
```

## 1. Install Ansible on the control node

On your Ubuntu control node:
```bash
sudo apt update
sudo apt install -y ansible openssh-client
ansible --version
```

On each managed Ubuntu host, install and start SSH:
```bash
sudo apt update
sudo apt install -y openssh-server python3
sudo systemctl enable --now ssh
```

Generate an SSH key on the control node if needed:
```bash
ssh-keygen -t ed25519
```
For ordinary Ubuntu hosts, copy the public key:
```bash
ssh-copy-id ubuntu@192.168.1.100
```
Replace the username and IP with your actual values. For AWS EC2, use the correct login username and existing `.pem` key instead; do not overwrite or upload the private key.

## 2. Configure inventory

Copy the examples and edit them:
```bash
cp inventory.ini.example inventory.ini
cp group_vars/all.yml.example group_vars/all.yml
nano inventory.ini
nano group_vars/all.yml
```

Example `inventory.ini`:
```ini
[control]
localhost ansible_connection=local

[docker_hosts]
docker1 ansible_host=192.168.1.101 ansible_user=ubuntu

[nginx_hosts]
nginx1 ansible_host=192.168.1.102 ansible_user=ubuntu

[jenkins_hosts]
jenkins1 ansible_host=192.168.1.103 ansible_user=ubuntu

[harbor_hosts]
harbor1 ansible_host=192.168.1.104 ansible_user=ubuntu

[vault_hosts]
vault1 ansible_host=192.168.1.105 ansible_user=ubuntu

[sonarqube_hosts]
sonar1 ansible_host=192.168.1.106 ansible_user=ubuntu

[monitoring_hosts]
monitor1 ansible_host=192.168.1.107 ansible_user=ubuntu
```
You can put the same host in multiple groups, or point all groups at one lab VM, but different services may compete for RAM/ports. Use one VM per service if possible.

For a PEM key, add this to the relevant host line:
```ini
ansible_ssh_private_key_file=/home/YOUR_USER/keys/lab.pem
```
Use an absolute path. Keep the key private (`chmod 400 /path/to/lab.pem`).

## 3. Set variables

Edit `group_vars/all.yml`. The sample defaults are suitable for a private practice network only:
- `ansible_become: true`
- `lab_http_allowed_cidr`: restrict firewall access at the network layer; this variable is used for documentation and not automatically applied to UFW.
- `harbor_hostname`: replace with a resolvable hostname or private IP.
- `vault_lab_dev_mode`: leave false unless intentionally using the insecure dev-lab playbook.
- `prometheus_scrape_targets`: configure real reachable targets.

## 4. Validate before running

```bash
ansible-inventory -i inventory.ini --graph
ansible all -i inventory.ini -m ping
```
To check one group:
```bash
ansible docker_hosts -i inventory.ini -m ping
```

## 5. Run each task

Run from the project root. Use `--ask-become-pass` if the remote user needs a sudo password:
```bash
ansible-playbook -i inventory.ini site/01-ssh-check.yml
ansible-playbook -i inventory.ini site/02-docker-compose.yml
ansible-playbook -i inventory.ini site/03-nginx.yml
ansible-playbook -i inventory.ini site/04-jenkins.yml
ansible-playbook -i inventory.ini site/05-harbor.yml
ansible-playbook -i inventory.ini site/06-vault-dev-lab.yml
ansible-playbook -i inventory.ini site/07-sonarqube.yml
ansible-playbook -i inventory.ini site/08-prometheus-grafana.yml
```

If you use one host for all services, change each play's `hosts:` group to the group containing that host or assign the same host to every group in inventory.

## 6. Verify services

```bash
ansible nginx_hosts -i inventory.ini -m command -a 'systemctl is-active nginx'
ansible jenkins_hosts -i inventory.ini -m command -a 'systemctl is-active jenkins'
ansible docker_hosts -i inventory.ini -m command -a 'docker --version'
ansible docker_hosts -i inventory.ini -m command -a 'docker compose version'
```

Web UI ports for lab use:
- Nginx: 80
- Jenkins: 8080
- Harbor HTTP lab: 80 (do not expose publicly; configure HTTPS for real use)
- Vault dev API/UI: 8200, localhost-only in this lab playbook
- SonarQube: 9000
- Prometheus: 9090
- Grafana: 3000

## Troubleshooting

- `UNREACHABLE`: check IP, routing, SSH port 22, security group/firewall, and username.
- `Permission denied (publickey)`: verify the private-key path and remote `authorized_keys`.
- `sudo: a password is required`: add `--ask-become-pass` or configure carefully scoped passwordless sudo for automation.
- Docker permission denied: log out/in after adding a user to the `docker` group; membership in this group is effectively root-equivalent.
- SonarQube exits: check `docker compose logs`; verify RAM and `vm.max_map_count`.
- Harbor: use the vendor documentation for the exact installer version and TLS setup. This lab config is HTTP-only and should stay on an isolated network.
