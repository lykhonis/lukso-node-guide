# Lukso Validator Node

This is a guide to setup a Lukso validator node in home environment. The guide suggests a use of a dedicated machine to run a node with validators, separate from a personal working machine.

> **_NOTE:_** Most of the steps require working in a terminal

- [Prerequisites](#prerequisites)
  - [My Setup](#my-setup)
    - [Auto Start](#auto-start)
- [System Setup](#system-setup)
  - [Update](#update)
  - [Remote Access](#remote-access)
    - [Install SSH](#install-ssh)
    - [Configure SSH](#configure-ssh)
    - [Configure Firewall](#configure-firewall)
    - [Enable SSH](#enable-ssh)
    - [Resolve Hostname](#resolve-hostname)
    - [Disable Password Authentication](#disable-password-authentication)
    - [Disable Non-Key Remote Access](#disable-non-key-remote-access)
    - [Verify Remote Access](#verify-remote-access)
  - [Keep System Up to Date](#keep-system-up-to-date)
  - [Disable Root Access](#disable-root-access)
  - [Block Unathorised Access](#block-unathorised-access)
  - [Configure Firewall](#configure-firewall)
  - [Improve SSH Connection](#improve-ssh-connection)
- [Node Setup](#node-setup)
- [Monitoring](#monitoring)
  - [Prometheus](#prometheus)
    - [Configure](#configure)
    - [Configure Service](#configure-service)
  - [Grafana](#grafana)
    - [Configure Service](#configure-service-1)
    - [Configure Dashboard](#configure-dashboard)
      - [Data Source](#data-source)
      - [Install Dashboard](#install-dashboard)
      - [Enable Alerts](#enable-alerts)
  - [Node Exporter](#node-exporter)
    - [Configure Service](#configure-service-2)
  - [Json Exporter](#json-exporter)
    - [Prerequisites](#prerequisites-1)
    - [Build and Install](#build-and-install)
    - [Configure](#configure-1)
    - [Configure Service](#configure-service-3)
  - [Ping](#ping)
    - [Configure](#configure-2)
    - [Configure Service](#configure-service-4)
- [Credits](#credits)

## Prerequisites

- [Ubuntu](https://ubuntu.com/)
- Dedicated PC

### My Setup

- Ubuntu 20.04.4 LTS
- [Intel NUC 10 Performance Kit – Intel Core i5](https://www.amazon.com/dp/B083GH1SSN/ref=cm_sw_r_cp_api_glt_i_PAVWC2JD4QTRSVHFFX10?_encoding=UTF8&psc=1)
- [Memory Kit 16GB (8GBx2)](https://www.amazon.com/dp/B083VWCZLQ/ref=cm_sw_r_cp_api_glt_i_1YV0G8K4GAH0NYP6BN00?_encoding=UTF8&psc=1)
- [Samsung 970 EVO Plus SSD 1TB](https://www.amazon.com/dp/B07MFZY2F2/ref=cm_sw_r_cp_api_glt_i_XF7ZS3XYJCQM5PH05P7V?_encoding=UTF8&psc=1)

I've spent around $850. Current price seem to be less around $700. I connected my NUC with ethernet cable directly into my router.

#### Auto Start

I had to manually change BIOS settings of NUC to ensure if power gets reset, NUC auto starts. A simple test is unplug and plug power cord back in.

For NUC I followed following steps:
1. Press F2 during boot to enter BIOS setup
2. Go to `Power` -> `Secondary Power Settings` menu
3. Set the option for `After Power Failure` to `Power One`
4. Press F10 to save changes and exit BIOS

## System Setup

> **_NOTE:_** Following steps are performed directly on a node machine.

In order to remotelly access a machine running a node, it needs to be configured.

### Update

```shell=
sudo apt update
sudo apt upgrade -y
sudo apt install -y nano wget make git
```

### Remote Access

SSH is used to enable remote access from other machine using localy network through WiFi or broadband connections. This is a common practice and can be quite useful if a node machine does not have input (keyboard/mouse) nor a display. Once setup, a node machine can be placed elsewhere and only personal computer could be used to control and maintain it.

#### Install SSH

```shell=
sudo apt install --assume-yes openssh-server
```

#### Confiugre SSH

Choose a port number larger than `50000`. This will be used later.

```shell=
sudo nano /etc/ssh/sshd_config
```

Change and enable a port by uncommenting (removing `#`) and changing `22` to new chosen port number:
```shell=
Port 50000
```

Close editor by pressing `ctrl` + `X`, then save.

#### Configure Firewall

Enable ssh in firewall by replacing *replace-port* with new port:

```shell=
sudo ufw allow replace-port
```

#### Enable SSH

```shell=
sudo systemctl start ssh
sudo systemctl enable ssh
```

#### Resolve Hostname

In order to locate a node machine in local network, it requires IP. Execute following command to resolve a node machine's IP:

```shell=
ifconfig
```

Locate IP  address (`inet`) in `eno1` section, e.g. `192.168.86.29`.

```
eno1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
      inet 192.168.86.29  netmask 255.255.255.0  broadcast 192.168.86.255
```

Close ssh session by executing `exit`.

> **_NOTE:_** Following steps are performed a personal computer.

Verify basic access to a node machine by using ssh. SSH requires user name of a node machine, its hostname and previously chosen ssh port.

```shell=
nano ~/.ssh/config 
```

Type in the following and replace *replace-user*, *replace-ip*, and *replace-port*:

```shell=
Host lukso
  User replace-user
  HostName replace-ip
  Port replace-port
```

Attempt to connect to verify the configuration:

```shell=
ssh lukso
```

Once connected, enter a password of user on a node machine. If a connection was okay, a shell should be presented in a terminal. At this point, it could closed.

#### Disable Password Authentication

On a personal computer, create new key pair for ssh authentication if needed.

```shell=
ssh-keygen -t rsa -b 4096
```

Copy a generated public key **keyname.pub** to a node machine. Replace **keyname.pub** with a key in home directory.

```shell=
ssh-copy-id -i ~/.ssh/keyname.pub lukso
```

#### Disable Non-Key Remote Access

On a personal computer, try to ssh again. This time it should not prompt for a password.

```shell=
ssh lukso
```

Configure SSH by opening a configuration file and modifying several options:

```shell=
sudo nano /etc/ssh/sshd_config
```

Options:

```shell=
ChallengeResponseAuthentication no
PasswordAuthentication no
PermitRootLogin prohibit-password
PermitEmptyPasswords no
```

Close editor by pressing `ctrl` + `X`, then save. Validate SSH configuration and restart ssh service.

```shell=
sudo sshd -t
sudo systemctl restart sshd
```

Close ssh session by executing `exit`.

#### Verify Remote Access

```shell=
ssh lukso
```
    
Stay connected to a remote node machine to perform next steps.

### Keep System Up to Date
    
Update a system manually:
    
```shell=
sudo apt-get update -y
sudo apt dist-upgrade -y
sudo apt-get autoremove
sudo apt-get autoclean
```

Keep a system up to date automatically:

```shell=
sudo apt-get install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

### Disable Root Access
    
A root access should not be used. Instead, a user should be using `sudo` to perform privilged operations on a system.

```shell=
sudo passwd -l root
```

### Block Unathorised Access

Install `fail2ban` to block IP addresses that exceed failed ssh login attempts.

```shell=
sudo apt-get install fail2ban -y
```

Edit a config to monitor ssh logins

```shell=
sudo nano /etc/fail2ban/jail.local
```

Replace *replace-port* to match the ssh port number.

```shell=
[sshd]
enabled=true
port=replace-port
filter=sshd
logpath=/var/log/auth.log
maxretry=3
ignoreip=
```

Close editor by pressing `ctrl` + `X`, then save. Restart `fail2ban` service:

```shell=
sudo systemctl restart fail2ban
```

### Configure Firewall

By default deny all traffic:

```shell=
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

Allow P2P ports for Lukso clients:

```shell=
sudo ufw allow 30303/tcp
sudo ufw allow 13000/tcp
sudo ufw allow 12000/udp
sudo ufw allow 30303/udp
```

> **_NOTE:_** make sure to open same ports on your home router

Enable Firewall:

```shell=
sudo ufw enable
```

Verify firewall configuration:

```shell=
sudo ufw status
```

It should look something like this (may be missing some ports):

```shell=
Status: active

To                         Action      From
--                         ------      ----
13000/tcp                  ALLOW       Anywhere
12000/udp                  ALLOW       Anywhere
30303/tcp                  ALLOW       Anywhere
ssh-port/tcp               ALLOW       Anywhere
30303/udp                  ALLOW       Anywhere
13000/tcp (v6)             ALLOW       Anywhere (v6)
12000/udp (v6)             ALLOW       Anywhere (v6)
30303/tcp (v6)             ALLOW       Anywhere (v6)
ssh-port/tcp (v6)          ALLOW       Anywhere (v6)
30303/udp (v6)             ALLOW       Anywhere (v6)
```

### Improve SSH Connection

While setting up a system, ssh terminal may seem to be slow due wifi power management settings on a node machine. To disable it, modify a config.

```shell=
sudo nano /etc/NetworkManager/conf.d/default-wifi-powersave-on.conf
```

Config:
```shell=
[connection]
wifi.powersave = 2
```

Close editor by pressing `ctrl` + `X`, then save. Restart `NetworkManager` service:

```shell=
sudo systemctl restart NetworkManager
```

## Node Setup

> **_NOTE:_** Following steps are performed on personal machine.

Access a remote node machine

```shell=
ssh lukso
```

**TBD AS LUKSO IS PREPEARING FOR L16 TESTNET**.
In the meantime follow developments and instructions of [L16 beta](https://docs.lukso.tech/networks/l16-testnet).

## Monitoring

Sets up a dashboard to monitor state of a node machine, node, and validators.

> **_NOTE:_** Following steps are performed on personal machine.

Access a remote node machine

```shell=
ssh lukso
```

### Prometheus

```shell=
sudo adduser --system prometheus --group --no-create-home
```

Identify latest version for `linux-amd64` [here](https://prometheus.io/download/), e.g. `2.34.0`. Install prometheus by replacing `{VERSION}` in the following:

```shell=
cd
wget https://github.com/prometheus/prometheus/releases/download/v{VERSION}/prometheus-{VERSION}.linux-amd64.tar.gz
tar xzvf prometheus-{VERSION}.linux-amd64.tar.gz
cd prometheus-{VERSION}.linux-amd64
sudo cp promtool /usr/local/bin/
sudo cp prometheus /usr/local/bin/
sudo chown root:root /usr/local/bin/promtool /usr/local/bin/prometheus
sudo chmod 755 /usr/local/bin/promtool /usr/local/bin/prometheus
cd
rm prometheus-{VERSION}.linux-amd64.tar.gz
rm -rf prometheus-{VERSION}.linux-amd64
```

#### Configure

```shell=
sudo mkdir -p /etc/prometheus/console_libraries /etc/prometheus/consoles /etc/prometheus/files_sd /etc/prometheus/rules /etc/prometheus/rules.d
```

Edit configuration file:

```shell=
sudo nano /etc/prometheus/prometheus.yml
```

The content of configuration file:

```
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: ['127.0.0.1:9090']
  - job_name: 'beacon node'
    scrape_interval: 5s
    static_configs:
      - targets: ['127.0.0.1:8080']
  - job_name: 'node_exporter'
    scrape_interval: 5s
    static_configs:
      - targets: ['127.0.0.1:9100']
  - job_name: 'validator'
    scrape_interval: 5s
    static_configs:
      - targets: ['127.0.0.1:8081']
  - job_name: 'ping_google'
    metrics_path: /probe
    params:
      module: [icmp]
    static_configs:
      - targets:
        - 8.8.8.8
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9115  # The blackbox exporter's real hostname:port.
  - job_name: 'ping_cloudflare'
    metrics_path: /probe
    params:
      module: [icmp]
    static_configs:
      - targets:
        - 1.1.1.1
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9115  # The blackbox exporter's real hostname:port.
  - job_name: json_exporter
    static_configs:
    - targets:
      - 127.0.0.1:7979
  - job_name: json
    metrics_path: /probe
    static_configs:
    - targets:
      - https://api.coingecko.com/api/v3/simple/price?ids=lukso-token&vs_currencies=usd
    relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: 127.0.0.1:7979
```

Prepare data directory for prometheus:

```shell=
sudo chown -R prometheus:prometheus /etc/prometheus
sudo mkdir /var/lib/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus
sudo chmod 755 /var/lib/prometheus
```

Open port to access to metrics. This is optional, only for external use:

```shell=
sudo ufw allow 9090/tcp
```

#### Configure Service

```shell=
sudo nano /etc/systemd/system/prometheus.service
```

The content of service configuration file:

```
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=always
RestartSec=5
ExecStart=/usr/local/bin/prometheus \
	--config.file /etc/prometheus/prometheus.yml \
	--storage.tsdb.path /var/lib/prometheus/ \
	--storage.tsdb.retention.time=31d \
	--web.console.templates=/etc/prometheus/consoles \
	--web.console.libraries=/etc/prometheus/console_libraries
ExecReload=/bin/kill -HUP $MAINPID

[Install]
WantedBy=multi-user.target
```

Enable service:

```shell=
sudo systemctl daemon-reload
sudo systemctl start prometheus
sudo systemctl enable prometheus
```

### Grafana

Install:

```shell=
cd
sudo apt-get install -y apt-transport-https
sudo apt-get install -y software-properties-common wget
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
sudo apt-get update
sudo apt-get install grafana-enterprise
```

#### Configure Service

```shell=
sudo nano /lib/systemd/system/grafana-server.service
```

The content of service configuration file:

```
[Unit]
Description=Grafana instance
Documentation=http://docs.grafana.org
Wants=network-online.target
After=network-online.target
After=postgresql.service mariadb.service mysql.service

[Service]
EnvironmentFile=/etc/default/grafana-server
User=grafana
Group=grafana
Type=simple
Restart=on-failure
WorkingDirectory=/usr/share/grafana
RuntimeDirectory=grafana
RuntimeDirectoryMode=0750
ExecStart=/usr/sbin/grafana-server \
                            --config=${CONF_FILE} \
                            --pidfile=${PID_FILE_DIR}/grafana-server.pid \
                            --packaging=deb \
                            cfg:default.paths.logs=${LOG_DIR} \
                            cfg:default.paths.data=${DATA_DIR} \
                            cfg:default.paths.plugins=${PLUGINS_DIR} \
                            cfg:default.paths.provisioning=${PROVISIONING_CFG_DIR}


LimitNOFILE=10000
TimeoutStopSec=20
CapabilityBoundingSet=
DeviceAllow=
LockPersonality=true
MemoryDenyWriteExecute=false
NoNewPrivileges=true
PrivateDevices=true
PrivateTmp=true
PrivateUsers=true
ProtectClock=true
ProtectControlGroups=true
ProtectHome=true
ProtectHostname=true
ProtectKernelLogs=true
ProtectKernelModules=true
ProtectKernelTunables=true
ProtectProc=invisible
ProtectSystem=full
RemoveIPC=true
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX
RestrictNamespaces=true
RestrictRealtime=true
RestrictSUIDSGID=true
SystemCallArchitectures=native
UMask=0027

[Install]
Alias=grafana.service
WantedBy=multi-user.target
```

Enable service:

```shell=
sudo systemctl daemon-reload
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
```

Open port to access to metrics. This is optional, only for external use:

```shell=
sudo ufw allow 3000/tcp
```

#### Configure Dashboard

Login to grafana by navigating to webrowser `http://192.168.86.29:3000`. Replace `192.168.86.29` with IP of your node machine. This is same IP used to ssh.

Default credentials are username and password `admin`. Set a new secure (long) password when prompted by grafana.

##### Data Source

1. On the left-hand menu, hover over the gear menu and click on `Data Sources`
2. Then click on the Add Data Source button
3. Hover over the Prometheus card on screen, then click on the Select button
4. Enter http://127.0.0.1:9090/ into the URL field, then click Save & Test

##### Install Dashboard

1. Hover over the plus symbol icon in the left-hand menu, then click on Import
2. Copy and paste [the dashboard](/grafana/dashboard.json) into the `Import via panel json` text box on the screen
3. Then click the Load button
4. Then click the Import button

##### Enable Alerts

1. On the left-hand menu, hover over the alarm menue and click on `Notification channels`
2. Click on `New channel`
3. Select `Type` and [configure](https://grafana.com/docs/grafana/latest/alerting/old-alerting/notifications/)

On lukso dashboard:

1. Scroll down on a dashboard to `Alerts` section
2. Select each alert and click `Edit`
3. In `Alert` tab, select notifications `send to`
4. Save and repeat for each alert

### Node Exporter

Monitors node stats:

```shell=
sudo adduser --system node_exporter --group --no-create-home
```

Install:

```shell=
cd
wget https://github.com/prometheus/node_exporter/releases/download/v1.0.1/node_exporter-1.0.1.linux-amd64.tar.gz
tar xzvf node_exporter-1.0.1.linux-amd64.tar.gz
sudo cp node_exporter-1.0.1.linux-amd64/node_exporter /usr/local/bin/
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
rm node_exporter-1.0.1.linux-amd64.tar.gz
rm -rf node_exporter-1.0.1.linux-amd64
```

#### Configure Service

```shell=
sudo nano /etc/systemd/system/node_exporter.service
```

The content of service configuration file:

```
[Unit]
Description=Node Exporter

[Service]
Type=simple
Restart=always
RestartSec=5
User=node_exporter
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

Enable service:

```shell=
sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter
```

### Json Exporter

#### Prerequisites 

Check `go` version if installed:

```shell=
go version
```

If it is less than `1.17.7` please install following:

```shell=
wget https://dl.google.com/go/go1.17.7.linux-amd64.tar.gz
sudo tar -xvf go1.17.7.linux-amd64.tar.gz
rm go1.17.7.linux-amd64.tar.gz
sudo mv go /usr/local/go-1.17.7
sudo ln -sf /usr/local/go-1.17.7/bin/go /usr/bin/go
go version
```

#### Build and Install

User:

```shell=
sudo adduser --system json_exporter --group --no-create-home
```

Install:

```shell=
cd
git clone https://github.com/prometheus-community/json_exporter.git
cd json_exporter
make build
sudo cp json_exporter /usr/local/bin/
sudo chown json_exporter:json_exporter /usr/local/bin/json_exporter
cd
rm -rf json_exporter
```

#### Configure

```shell=
sudo mkdir /etc/json_exporter
sudo chown json_exporter:json_exporter /etc/json_exporter
```

Setup `LYX` token price:

```shell=
sudo nano /etc/json_exporter/json_exporter.yml
```

The content of configuration file:

```
metrics:
- name: lyxusd
  path: "{.lukso-token.usd}"
  help: Lukso (LYX) price in USD
```

Change ownership of configuration file:

```shell=
sudo chown json_exporter:json_exporter /etc/json_exporter/json_exporter.yml
```

#### Configure Service

```shell=
sudo nano /etc/systemd/system/json_exporter.service
```

The content of service configuration file:

```
[Unit]
Description=JSON Exporter

[Service]
Type=simple
Restart=always
RestartSec=5
User=json_exporter
ExecStart=/usr/local/bin/json_exporter --config.file /etc/json_exporter/json_exporter.yml

[Install]
WantedBy=multi-user.target
```

Enable service:

```shell=
sudo systemctl daemon-reload
sudo systemctl start json_exporter
sudo systemctl enable json_exporter
```

### Ping

Pings google and cloudflare to track latency. This is optional.

```shell=
sudo adduser --system blackbox_exporter --group --no-create-home
```

Install:

```shell=
cd
wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.18.0/blackbox_exporter-0.18.0.linux-amd64.tar.gz
tar xvzf blackbox_exporter-0.18.0.linux-amd64.tar.gz
sudo cp blackbox_exporter-0.18.0.linux-amd64/blackbox_exporter /usr/local/bin/
sudo chown blackbox_exporter:blackbox_exporter /usr/local/bin/blackbox_exporter
sudo chmod 755 /usr/local/bin/blackbox_exporter
rm blackbox_exporter-0.18.0.linux-amd64.tar.gz
rm -rf blackbox_exporter-0.18.0.linux-amd64
```

Enable ping permissions:

```shell=
sudo setcap cap_net_raw+ep /usr/local/bin/blackbox_exporter
```

#### Configure

```shell=
sudo mkdir /etc/blackbox_exporter
sudo chown blackbox_exporter:blackbox_exporter /etc/blackbox_exporter
```

```shell=
sudo nano /etc/blackbox_exporter/blackbox.yml
```

The content of configuration file:

```
modules:
        icmp:
                prober: icmp
                timeout: 10s
                icmp:
                        preferred_ip_protocol: ipv4
```

Change ownership of configuration file:

```shell=
sudo chown blackbox_exporter:blackbox_exporter /etc/blackbox_exporter/blackbox.yml
```

#### Configure Service

```shell=
sudo nano /etc/systemd/system/blackbox_exporter.service
```

The content of service configuration file:

```
[Unit]
Description=Blackbox Exporter

[Service]
Type=simple
Restart=always
RestartSec=5
User=blackbox_exporter
ExecStart=/usr/local/bin/blackbox_exporter --config.file /etc/blackbox_exporter/blackbox.yml

[Install]
WantedBy=multi-user.target
```

Enable service:

```shell=
sudo systemctl daemon-reload
sudo systemctl start blackbox_exporter
sudo systemctl enable blackbox_exporter
```

## Credits

- https://github.com/metanull-operator/eth2-ubuntu
