# Provision Prometheu & Grafana through Ansible

## Documentations
<br>
<img width="600" alt="47" src="https://user-images.githubusercontent.com/13994900/98179028-78425080-1ec3-11eb-8ff4-badc06e11de0.PNG">
<br>
<br>

## Prerequistes

```
yum install ansible wget git -y
python --version   /* 2.7 */
```
<br>

## Steps from Scratche based on my Documentation


* Create a repo called  ``` grafana-prometheus-ansible ```

### Install and Set up Prometheus tool
1. Create a saperate folder for Prometheus
```
mkdir grafana-prometheus-ansible/Prometheus
```
2. In this step, we will configure prometheus as a systemd service. We will create a new service file prometheus.service on the '/etc/systemd/system' directory.
Go to the '/etc/systemd/system/' directory and create new service file 'prometheus.service' using vim editor.
<br>

```
cd /etc/systemd/system/
vim prometheus.service

```
<br>
Paste the prometheus service configuration below.
<br>

```
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
--config.file /etc/prometheus/prometheus.yml \
--storage.tsdb.path /var/lib/prometheus/ \
--web.console.templates=/etc/prometheus/consoles \
--web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
```
<br>

3. In this step, we will add the node_exporter to the prometheus server. 

<br>

```
vi prometheus.yml
```

<br>

```
global: 
  scrape_interval: 10s 
  evaluation_interval: 1s  

scrape_configs: 

  - job_name: 'prometheus' 
    scrape_interval: 5s 
    static_configs: 
    - targets: ['localhost:9090'] 

  - job_name: 'node' 
    ec2_sd_configs: 
      - region: us-east-1 
        port: 9100 

  - job_name: 'node_exporter' 
    static_configs: 
      - targets: ['localhost:9100']
```

4. Create a playbook called ``` Prometheus_install.yml ``` to install prometheus. 

```
---
- name: Prometheus Installation
  hosts: localhost
  become: true
  become_method: sudo
  user: centos
  ignore_errors: yes
  tasks:
  - name: Create new user, set specific permission 
    shell: "{{item}}"
    with_items:
    - sudo yum update -y
    - sudo useradd --no-create-home --shell /bin/false prometheus
    - sudo mkdir /etc/prometheus
    - sudo mkdir /var/lib/prometheus
    - sudo chown prometheus:prometheus /etc/prometheus
    - sudo chown prometheus:prometheus /var/lib/prometheus

  - name: Install Prometheus
    shell: "{{item}}"
    with_items:
    - sudo wget https://github.com/prometheus/prometheus/releases/download/v2.3.2/prometheus-2.3.2.linux-amd64.tar.gz
    - tar -xvf prometheus-2.3.2.linux-amd64.tar.gz
    - mv prometheus-2.3.2.linux-amd64 prometheus-files
  


  - name: Copy binaries 
    command: "{{item}}"
    with_items:
    - sudo cp prometheus-files/prometheus /usr/local/bin/
    - sudo cp prometheus-files/promtool /usr/local/bin/
    - sudo chown prometheus:prometheus /usr/local/bin/prometheus
    - sudo chown prometheus:prometheus /usr/local/bin/promtool
    

  
  - name: Move console files
    shell: "{{item}}"
    with_items:
    - sudo cp -r prometheus-files/consoles /etc/prometheus 
    - sudo cp -r prometheus-files/console_libraries /etc/prometheus 
    - sudo chown -R prometheus:prometheus /etc/prometheus/consoles 
    - sudo chown -R prometheus:prometheus /etc/prometheus/console_libraries 


  - name: Creates prometheus config file
    template:
      src: prometheus.yml
      dest: /etc/prometheus/prometheus.yml



  - name: Change permission
    shell: "sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml"


  - name: Creates prometheus config file
    template:
      src: prometheus.service
      dest: /etc/systemd/system/prometheus.service



  - name:  Restarts
    command: "{{item}}"
    with_items:
    - sudo iptables -F
    - sudo systemctl daemon-reload 
    - sudo systemctl start prometheus 
    - sudo systemctl enable prometheus 
    - sudo systemctl status prometheus 
```




