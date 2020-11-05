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

## Steps from Scratch based on my Documentation


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
<br>

Prometheus tools is up and running 

<img width="952" alt="48" src="https://user-images.githubusercontent.com/13994900/98199481-d5a0c680-1ef0-11eb-9739-31d47c0e17b0.PNG">


### Install and Set up Grafana tool
<br>

1. Create a file called ```grafana.repo ``` with configuration for Grafana tool.

```
vi Grafana/grafana.repo 
```
Add the bellow configuration. 
```
[grafana] 
name=grafana 
baseurl=https://packages.grafana.com/oss/rpm 
repo_gpgcheck=1 
enabled=1 
gpgcheck=1 
gpgkey=https://packages.grafana.com/gpg.key 
sslverify=1 
sslcacert=/etc/pki/tls/certs/ca-bundle.crt 
```
2. Playbook to Install Grafana. 
```
---
- name: Node Exporter Installation
  hosts: localhost
  become: true
  become_method: sudo
  user: centos
  tasks:
    - name: Create repo
      template:
      src: grafana.repo
      dest: /etc/yum.repos.d/grafana.repo
      
    - name:
      shell: "sudo yum install grafana fontconfig freetype* urw-fonts -y"
      
    - name: Starts Grafana
      command: "{{item}}"
      with_items: 
      - sudo systemctl daemon-reload 
      - sudo systemctl start grafana-server 
      - sudo systemctl enable grafana-server.service 
      - sudo systemctl status grafana-server 
```
Grafana tool is up and running 

<img width="960" alt="50" src="https://user-images.githubusercontent.com/13994900/98199555-03860b00-1ef1-11eb-8523-7ec35e26b9db.PNG">

<br>

### Install and Set up Node Exporter, to collect metrics from Prometheus than to visualize on Grafana Dashboard. 

1. Create a folder called ``` mkdir Node_Exporter ``` 
2. It is time to configure Node Exporter as a service inside systemd. Create a file ``` vi node_exporter.service ``` and put the lines mentioned below in the file and save it.

```
[Unit] 
Description=Node Exporter 
Wants=network-online.target 
After=network-online.target 
[Service] 
User=prometheus 
ExecStart=/etc/prometheus/node_exporter/node_exporter 
[Install] 
WantedBy=default.target 
```
3. Create a playbook to install Node Exporter on a different machine.
```
---
- name: Node Exporter Installation
  hosts: localhost
  become: true
  become_method: sudo
  user: centos
  tasks:



  - name: Create folders and Set permissions
    shell: "{{item}}"
    with_items:
    -  sudo useradd --no-create-home --shell /bin/false prometheus
    -  mkdir -p /etc/prometheus
    -  mkdir -p /var/lib/prometheus
    -  chown prometheus:prometheus /etc/prometheus

   
  - name: Install Node Exporter
    shell: "{{item}}"
    with_items:
    - sudo wget https://github.com/prometheus/node_exporter/releases/download/v0.18.1/node_exporter-0.18.1.linux-amd64.tar.gz
    - sudo tar xvfz node_exporter-0.18.1.linux-amd64.tar.gz 
    - sudo mv node_exporter-0.18.1.linux-amd64 /etc/prometheus/node_exporter



  - name: Copy the Node Exporter systemd service file
    template:
      src: node_exporter.service
      dest: /etc/systemd/system/node_exporter.service
#     notify: restart node_exporter



  - name: Restart
    command: "{{item}}"
    with_items:
    - sudo systemctl daemon-reload 
    - sudo  systemctl start node_exporter
    - sudo systemctl enable node_exporter
    - sudo systemctl enable node_exporter
```

I do have access to the metrics of this Machine.

<img width="850" alt="51" src="https://user-images.githubusercontent.com/13994900/98201647-c1ab9380-1ef5-11eb-8da4-5566d58881a4.PNG">

### How to see how much CPU is using my machine? 

1. Take Node_Exporter's Ip and add as the bellow example. Now, you will make a connection between Prometheus machine and Node_Exporter machine.
```
vi Prometheus/ prometheus.yml 
```
Add a new job 
```
- job_name: 'prometheus' 
    scrape_interval: 5s 
    static_configs: 
    - targets: ['add-Node_exporter_Ips:9090'] 
```
