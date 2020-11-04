# Provision Prometheu & Grafana through Ansible

## Documentations
<br>
<img width="337" alt="47" src="https://user-images.githubusercontent.com/13994900/98179028-78425080-1ec3-11eb-8ff4-badc06e11de0.PNG">
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
add the prometheus code
```




